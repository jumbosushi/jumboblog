+++
date = '2026-05-06'
draft = true
title = 'Bazel Skyframe explained'
+++

The term "SkyFrame" is referenced often as you dig deeper into the Bazel internals. However, I find the available explanations confusing and found myself re-reading the [Skyframe documentation](https://bazel.build/versions/7.2.0/contribute/codebase#skyframe) several times. I've looked around the source code just enough to get the essence of it, so this post summarizes what I've learned.

We'll explore what SkyFrame is and how it's implemented (in a VERY high level overview). Basic Bazel knowledge is assumed. For the sake of simplicity, we'll just walk through the local machine worker scenario (no RBE).

After reading this post, you should understand why the total action count gradually increases from 1 as the build progresses!

![Bazel build action count](/images/2026-05-06-bazel-skyframe/action-count.gif)

## Repo setup

I prepared a demo Bazel repo here: https://github.com/jumbosushi/bazel-hello-world-py

The only BUILD.bazel builds a `py_binary` target that depends on two files:
```txt
load("@rules_python//python:py_binary.bzl", "py_binary")

py_binary(
    name = "hello",
    srcs = [
        "hello.py",
        "lib.py",
    ],
)
```

Here's what it looks like when you run it:
```txt
$ bazel run //:hello
INFO: Analyzed target //:hello (90 packages loaded, 3009 targets configured).
INFO: Found 1 target...
Target //:hello up-to-date:
  bazel-bin/hello
INFO: Elapsed time: 6.105s, Critical Path: 0.41s
INFO: 9 processes: 9 internal.
INFO: Build completed successfully, 9 total actions
INFO: Running command line: bazel-bin/hello
hello world!
```

Skyframe DAG is available from the following command:
```txt
bazel dump --skyframe deps
```

Even in this hello world example, the dump file is >160K lines. Here's a small subset of the graph that's related to the local files in the repo. We'll be using this subset starting with `ARTIFACT_NESTED_SET` to walk through how Skyframe might execute it:

![Skyframe DAG subset](/images/2026-05-06-bazel-skyframe/dag-subset.png)

## Simplified mental model

Skyframe is a parallel and incremental evaluation framework. The way I understand it, Skyframe execution is roughly consists of three components that uses a standard Java classes at its core:
- Evaluator: Orchestrates the whole Skyframe build lifecycles
- Graph: DAG of build targets & dependencies (implemented with [`ConcurrentHashMap`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html))
- Executor: Executes the actual computation of nodes in parallel (implemented with [`ThreadPoolExecutor`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html))

![Skyframe components](/images/2026-05-06-bazel-skyframe/components.png)

The nodes in the graph is a wrapper for `SkyValue` and they are refenced by it's `SkyKey`. Evaluator uses the SkyKey's associated singleton pure function called SkyFunction to build SkyValue. Every type of computation done by Skyframe is implemented as SkyFunction (there's [87 listed](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/SkyFunctions.java;l=120;drc=8584d864876062a15dd11423717b0412f3d1fd8d?q=ARTIFACT_NESTED_SET&sq=&ss=bazel%2Fbazel) as of v9.0.0).

The simplest SkyFunction I found was [`FileStateFunction`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/FileStateFunction.java;l=35?q=FileStateFunction&ss=bazel%2Fbazel) which essentially returns a file content. To map to the definitions
- SkyFunction: `FileStateFunction`
- SkyValue:  The file content of the given path at `/workspace/hello.py`
- SkyKey: `"FILE_STATE{/workspace/hello.py}"` (a pair of type & name)

 SkyFunction can also call other SkyFunction (e.g. `ARTIFACT` calls `FILE` in our subgraph). In fact when we call `bazel build //...`, Bazel passes the `//...` pattern param to [TargetPatternPhaseFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/TargetPatternPhaseFunction.java) during the loading phase which is the starting point for recursively constructing the DAG.

## Evaluation Model

It's common in build systems DAG to have thousands nodes which would be time consuming to traverse the entire graph before execution. SkyFrame's evaluation model is optimized to withstand dependency discovery latency, and implements incremental evaluation mechanism.

Each build target & its dependencies are represented as a DAG node. Node can be in one of three states: ready, waiting, and done. Node also have a counter of how many dependencies it needs to wait for which is used to transition between states. Evaluator enqueues the nodes that are ready to the Executor, then the Executor runs the node's SkyFunction `.compute()` method.

The interesting behavior in Skyframe is the "restart". On a first pass, a node might be evaluated but it may need to wait for its dependency to be evaluated first. In this case, Skyframe sets the parent node state to `waiting`. When the dependency's state transitions to `done`, evaluator uses the reverse dependency edge and "signal" the parent to decrement the dependency counter. If the counter reaches 0, the parent node is enqueued again. Eventually, the parent node will "restart" and be evaluated again - this time with complete dependencies available to run its own `.compute` step.

This part is complicated! Let's try to examine this behavior in a simplified code.

## Simplified Python code

I created a [skyframe.py gist](https://gist.github.com/jumbosushi/cdde35941cb4de3e73f8d157c9145253 ) that emulates Skyframe execution model in a simplified way. From the example above, let's examine how a sub-graph from `"FILE: [workspace]/lib.py"` node might be evaluated (in a single threaded environment) using this code:
```txt
...
[5] FILE:/workspace/lib.py  [dequeued]
      enqueued FILE_STATE:/workspace/lib.py
      -> FILE:/workspace/lib.py is WAITING
      waiting on 2 deps: [FILE:/workspace/, FILE_STATE:/workspace/lib.py]
[6] FILE:/workspace/  [dequeued]
      enqueued FILE_STATE:/workspace/
      -> FILE:/workspace/ is WAITING
      waiting on 1 deps: [FILE_STATE:/workspace/]
...
[8] FILE_STATE:/workspace/lib.py  [dequeued]
      done: FileStateValue('/workspace/lib.py', content='bytes(/workspace/lib.py)')
      signal FILE:/workspace/lib.py (1 remaining)
[9] FILE_STATE:/workspace/  [dequeued]
      done: FileStateValue('/workspace/', content='bytes(/workspace/)')
      signal FILE:/workspace/ (0 remaining)
      -> FILE:/workspace/ is READY (enqueued)
[10] FILE:/workspace/  [dequeued]
      done: FileValue('/workspace/')
      signal FILE:/workspace/hello.py (0 remaining)
      -> FILE:/workspace/hello.py is READY (enqueued)
      signal FILE:/workspace/lib.py (0 remaining)
      -> FILE:/workspace/lib.py is READY (enqueued)
```

![Skyframe execution trace](/images/2026-05-06-bazel-skyframe/execution-trace.png)

Notice that on Step 9, `FILE_STATE:/workspace/` restarted after its dependencies were available. The restart behavior is also available in the Python Code's Executor class:
```python
# Code snippet from the python gist without the noise
class Evaluator:
    def evaluate(self, key: SkyKey) -> SkyValue | None:
	    while self.queue:
		    ...
            env = Environment(self.graph, self.queue, key)
            result = self.executor.run(key, env)

            node = self.graph.nodes[key]

			# .compute was successful
            if result is not None:
                node.value = result
                node.state = Node.DONE

                for parent in node.reverse_deps:
                    parent_node = self.graph.nodes[parent]
                    parent_node.waiting_on -= 1

                    if parent_node.waiting_on == 0:
                        parent_node.state = Node.READY
                        self.queue.append(parent)
			else:
                node.waiting_on = len(env._new_deps)
                node.state = Node.WAITING
```

This evaluation model also explains why the action count in the CLI grows from 1 to a much bigger number during the execution phase. SkyFrame doesn't have any idea of what the entire DAG looks like, but it generates the DAG via SkyFunctions and follows the edges to recursively evaluate nodes. It then updates the total action count as it continues the evaluation.

## Caching

Because SkyKey is immutable and available globally in the graph, same computation only executes once. In our sub-graph, `FILE:/workspace/` node has two incoming edges. Once its computation is done, parent nodes reuse the same SkyValue result.

![Skyframe caching](/images/2026-05-06-bazel-skyframe/caching.png)

As mentioned earlier, each of Bazel's execution phases have a corresponding SkyFunction:
- Loading: [TargetPatternPhaseFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/TargetPatternPhaseFunction.java)
- Analysis: [ConfiguredTargetFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ConfiguredTargetFunction.java;l=122?q=ConfiguredTargetFunction&ss=bazel%2Fbazel)
- Execution:  [ActionExecutionFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ActionExecutionFunction.java;l=137?q=ActionExecutionFunction&ss=bazel%2Fbazel) / [TargetCompletionFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/TargetCompletor.java;l=44?q=TargetCompletionFunction&ss=bazel%2Fbazel)

Because each phase share a same global graph and SkyFunctions are composed of each other, cache is reused across phases as well (e.g. reading the `hello.py` from file system only once). Caching also goes beyond a single build! The Bazel daemon state persists across the builds, so if the file doesn't change (based on `inotify`) then Bazel reuses the local cache ([this blog](https://jmmv.dev/2020/12/google-no-clean-builds.html) post covers this topic well).

## Conclusion

Hopefully this gives you a overview of Skyframe, but I can't emphasize enough how this just scratches the surface. Some example sources complexities are:
- Build performance improvements with [Skymeld](https://github.com/bazelbuild/bazel/issues/14057)
- Optimizing memory usage with [Skyfocus](https://bazel.build/advanced/performance/memory#trade-flexibility)
- Reducing node restarts with [StateMachines](https://bazel.build/contribute/statemachine-guide)

With that being said, doing this research helped me build a enough basic intuition on how Skyframe works to keep up with latest changes in Bazel internals.

## References

- [`BuildTool.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/buildtool/BuildTool.java;l=192;drc=215ec5cd3a0aa09cd9ede99d132465f3c9c59000) is "crux of the build system" and is a great starting point for LLM to dig into the Bazel build process internals.
- [`AbstractParallelEvaluator.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/skyframe/AbstractParallelEvaluator.java?q=AbstractParallelEvaluator&ss=bazel%2Fbazel) manages the DAG node states
- [`AbstractInMemoryNodeEntry.java`](https://cs.opensource.google/bazel/bazel/+/master/src/main/java/com/google/devtools/build/skyframe/AbstractInMemoryNodeEntry.java;l=65;drc=215ec5cd3a0aa09cd9ede99d132465f3c9c59000;bpv=0;bpt=1) implements the DAG node class
- [`SkyframeExecutor.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/SkyframeExecutor.java;l=1?q=SkyframeExecutor.java&sq=&ss=bazel%2Fbazel) is the main wrapper class that drives the SkyFrame executions
- All SkyFunctions are included in `src/main/java/com/google/devtools/build/lib/skyframe` directory
