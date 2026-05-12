+++
date = '2026-05-06'
draft = false
title = 'Bazel Skyframe explained'
+++

<!-- https://excalidraw.com/#json=_Sqq8GAFYKdgnXSGdhCvn,0kM9YnXrRGWFhX0leKCNlQ -->

The term "Skyframe" comes up often as you dig deeper into Bazel internals. However, the available explanations are pretty confusing and I found myself re-reading the [Skyframe documentation](https://bazel.build/reference/skyframe) multiple times. I've looked around the source code just enough to get the essence of it, so this post summarizes what I've learned.

We'll explore what Skyframe is and how it's implemented (at a very high level). Basic Bazel knowledge is assumed. For the sake of simplicity, we'll just walk through the local machine worker scenario (no RBE).

After reading this post, you should understand why the total action count gradually increases from 1 as the build progresses!

![Bazel build action count](/images/2026-05-06-bazel-skyframe/action-count.gif)

## Introduction

Skyframe is a **parallel** and **incremental** evaluation engine. Let's break this down phrase by phrase.

**Evaluation engine**

BUILD files establish explicit DAG relationships between build targets. The framework walks the graph and builds in an efficient order.

**Incremental**

Each node in the DAG is evaluated step by step, and the result is cached. Bazel has mechanisms to determine what changed since the last build and only rebuild those. In other words, the 2nd build is "incremental" relative to the cache of the 1st build.

**Parallel**

It's common for even a simple target to depend on dozens of implicit dependencies. Building them sequentially is not an option for a build tool, so Skyframe evaluates independent nodes in parallel.

Let's look at a hello world example to explore this further.

## Repo setup

I prepared a demo Bazel repo here: https://github.com/jumbosushi/bazel-hello-world-c

The only BUILD.bazel builds a `cc_binary` target that depends on few files:
```txt
load("@rules_cc//cc:cc_binary.bzl", "cc_binary")
load("@rules_cc//cc:cc_library.bzl", "cc_library")

cc_library(
    name = "lib",
    srcs = ["lib.c"],
    hdrs = ["lib.h"],
)

cc_binary(
    name = "hello",
    srcs = ["hello.c"],
    deps = [":lib"],
)
```

Here's what it looks like when you run it:
```txt
$ bazel run //:hello
INFO: Analyzed target //:hello (89 packages loaded, 534 targets configured).
INFO: Found 1 target...
Target //:hello up-to-date:
  bazel-bin/hello
INFO: Elapsed time: 1.460s, Critical Path: 0.22s
INFO: 14 processes: 10 internal, 4 darwin-sandbox.
INFO: Build completed successfully, 14 total actions
INFO: Running command line: bazel-bin/hello
Hello World!
```

Skyframe DAG is available from the following command:
```txt
bazel dump --skyframe deps
```

Even in this hello world example, the dump file is >60K lines. To make it more manageable, we'll be using this subset of the graph that's related to reading local files starting with `ARTIFACT` to walk through how Skyframe might execute it:

![Skyframe DAG subset](/images/2026-05-06-bazel-skyframe/dag-subset.png)

## Architecture overview

The way I understand it, Skyframe execution roughly consists of three components that use standard Java classes at its core:
- Evaluator: Orchestrates the whole Skyframe build lifecycle
- Graph: DAG of build targets & dependencies (implemented with [`ConcurrentHashMap`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html))
- Executor: Executes the actual computation of nodes in parallel (implemented with [`ThreadPoolExecutor`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html))

![Skyframe components](/images/2026-05-06-bazel-skyframe/components.png)

Each node in the graph wraps a SkyValue and is referenced by a SkyKey. The Evaluator uses the SkyKey's associated pure function (called a SkyFunction) to build the SkyValue. Every type of computation done by Skyframe is implemented as SkyFunction (there's [87 listed](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/SkyFunctions.java;l=120;drc=8584d864876062a15dd11423717b0412f3d1fd8d?q=ARTIFACT_NESTED_SET&sq=&ss=bazel%2Fbazel) as of v9.0.0).

The simplest SkyFunction I found was [`FileStateFunction`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/FileStateFunction.java;l=35?q=FileStateFunction&ss=bazel%2Fbazel) which essentially returns file content. To map to the definitions
- SkyKey: `"FILE_STATE{demo/hello.c}"` (a pair of type & name)
- SkyValue:  The file content of the given path at `demo/hello.c`
- SkyFunction: `FileStateFunction`

A SkyFunction can also call other SkyFunctions (e.g., `ARTIFACT` calls `FILE` in our subgraph). In fact, when we call `bazel build //...`, Bazel passes the `//...` pattern to [TargetPatternPhaseFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/TargetPatternPhaseFunction.java) during the loading phase, which is the starting point for recursively constructing the DAG.

## Evaluation model

Each build target and its dependencies are represented as DAG nodes. A node can be in one of three states: `READY`, `WAITING`, and `DONE`. Each node also has a counter tracking how many dependencies it needs to wait for, which determines state transitions. When all dependencies are resolved and the counter is 0, the node is considered as `READY`. Evaluator enqueues the `READY` nodes to the Executor, then the Executor runs the node's SkyFunction `.compute()` method. The state then transitions to `DONE` after the `.compute()` call returns.

The interesting behavior in Skyframe is the "restart". On a first pass, a node might be evaluated but it may need to wait for its dependency to be evaluated first (e.g. symlink target needs to be resolved before the symlink itself). In this case, Skyframe sets the parent node state to `WAITING`. When the dependency's state transitions to `DONE`, evaluator uses the reverse dependency edge to "signal" the parent which decrements the dependency counter. When the counter reaches 0, the parent node is enqueued again. Eventually, the parent node will "restart" and be evaluated again with complete dependencies available to run its own `.compute` step.

This part is complicated! Let's try to examine this behavior in a simplified code.

## Simplified Python code

I created a [skyframe.py gist](https://gist.github.com/jumbosushi/1e53dacfb765f2772f54e24db3909c3c) that emulates the Skyframe execution model in a simplified way. From the example above, let's examine how a sub-graph from the `"ARTIFACT"` node might be evaluated (in a single-threaded environment) using this code:
```txt
============================================================
Evaluating ArtifactKey
============================================================
      enqueued ARTIFACT:hello.c
[1] ARTIFACT:hello.c  [dequeued]
      enqueued FILE:demo/hello.c
      -> ARTIFACT:hello.c is WAITING
      waiting on 1 deps: [FILE:demo/hello.c]
[2] FILE:demo/hello.c  [dequeued]
      enqueued FILE:demo/
      enqueued FILE_STATE:demo/hello.c
      -> FILE:demo/hello.c is WAITING
      waiting on 2 deps: [FILE:demo/, FILE_STATE:demo/hello.c]
[3] FILE:demo/  [dequeued]
      enqueued FILE_STATE:demo/
      -> FILE:demo/ is WAITING
      waiting on 1 deps: [FILE_STATE:demo/]
[4] FILE_STATE:demo/hello.c  [dequeued]
      done: FileStateValue('demo/hello.c', content='bytes(demo/hello.c)')
      signal FILE:demo/hello.c (1 remaining)
[5] FILE_STATE:demo/  [dequeued]
      done: FileStateValue('demo/', content='bytes(demo/)')
      signal FILE:demo/ (0 remaining)
      -> FILE:demo/ is READY (enqueued)
[6] FILE:demo/  [dequeued]
      done: FileValue('demo/')
      signal FILE:demo/hello.c (0 remaining)
      -> FILE:demo/hello.c is READY (enqueued)
[7] FILE:demo/hello.c  [dequeued]
      done: FileValue('demo/hello.c')
      signal ARTIFACT:hello.c (0 remaining)
      -> ARTIFACT:hello.c is READY (enqueued)
[8] ARTIFACT:hello.c  [dequeued]
      done: ArtifactValue('hello.c')
```

Notice that on Step 8, `ARTIFACT:hello.c` restarted after its dependencies were available. The restart behavior is also visible in the Python Evaluator class:
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

Because SkyKey is immutable and available globally in the graph, the same computation only executes once. In our sub-graph, `FILE:demo/` node has two incoming edges. Once its computation is done, parent nodes reuse the same SkyValue result.

![Skyframe caching](/images/2026-05-06-bazel-skyframe/caching.png)

As mentioned earlier, each of Bazel's execution phases has a corresponding SkyFunction:
- Loading: [TargetPatternPhaseFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/TargetPatternPhaseFunction.java)
- Analysis: [ConfiguredTargetFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ConfiguredTargetFunction.java;l=122?q=ConfiguredTargetFunction&ss=bazel%2Fbazel)
- Execution:  [ActionExecutionFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/ActionExecutionFunction.java;l=137?q=ActionExecutionFunction&ss=bazel%2Fbazel) / [TargetCompletionFunction](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/TargetCompletor.java;l=44?q=TargetCompletionFunction&ss=bazel%2Fbazel)

Each phase builds on top of each other's Skyframe nodes. Because it shares the same global graph, the cache is reused across phases as well (e.g., reading `hello.c` from the filesystem only once). Here's a simplified version of how each phase creates different Skyframe nodes:

![Skyframe phases](/images/2026-05-06-bazel-skyframe/build-phases.png)

## Incrementality

The Skyframe evaluation cache is also reused beyond a single build! Bazel persists the Skyframe state in its [server](https://bazel.build/run/client-server) which is used across different client build invocations. It then sets up a file system watcher for local filesystems (using `inotify` on linux and `fsevents` on MacOS). 

When a file is updated, Bazel marks the changed leaf `FileState` node as `CHANGED`. The state change is propagated up the reverse dependencies, but they are instead marked as `DIRTY`. Skyframe uses two states in order to implement "change pruning". Even if the file itself was changed, if the computed output is identical then it can be reused as it is. During the bottom-up re-evaluation, the `CHANGED` node is evaluated first. If the output is identical (e.g. same SHA) then re-evaluation exits early at reverse deps nodes marked as `DIRTY` and skip `.compute()`.

![Change pruning](/images/2026-05-06-bazel-skyframe/change_prune.png)

For example, this is what the initial build with [`--subcommands`](https://bazel.build/reference/command-line-reference#build-flag--subcommands) flag looks like for our C example:
```txt
$ bazel build --subcommands //...
SUBCOMMAND: # cc_library rule target //:lib [action 'Compiling lib.c', configuration: 680dcaf1caca59c28bd4314128d3f53c6fa1ff268a38933011ddb444ea1ec37b, execution platform: @@platforms//host:host, mnemonic: CppCompile] ...
INFO: Analyzed 2 targets (89 packages loaded, 534 targets configured).
SUBCOMMAND: # cc_binary rule target //:hello [action 'Compiling hello.c', configuration: 680dcaf1caca59c28bd4314128d3f53c6fa1ff268a38933011ddb444ea1ec37b, execution platform: @@platforms//host:host, mnemonic: CppCompile] ...
SUBCOMMAND: # cc_library rule target //:lib [action 'Linking liblib.a', configuration: 680dcaf1caca59c28bd4314128d3f53c6fa1ff268a38933011ddb444ea1ec37b, execution platform: @@platforms//host:host, mnemonic: CppArchive] ...
SUBCOMMAND: # cc_binary rule target //:hello [action 'Linking hello', configuration: 680dcaf1caca59c28bd4314128d3f53c6fa1ff268a38933011ddb444ea1ec37b, execution platform: @@platforms//host:host, mnemonic: CppLink] ...
INFO: Found 2 targets...
INFO: Elapsed time: 1.498s, Critical Path: 0.22s
INFO: 14 processes: 10 internal, 4 darwin-sandbox.
INFO: Build completed successfully, 14 total actions
```

Notice that it ran four compile / link subcommands. I then update the C file to include a comment that will change the file node's state to `CHANGED`:
```diff
   #include <stdio.h>
   #include "lib.h"

++ // Test
   int main(void) {
       printf("%s!\n", greeting());
       return 0;
   }
```

When I build again, the change pruning kicks in. Only the action to compile `hello.c` runs, and once Skyframe verifies that the build output is the same it uses the result from the cache:

```txt
$ bazel build --subcommands //...
INFO: Analyzed 2 targets (0 packages loaded, 0 targets configured).
SUBCOMMAND: # cc_binary rule target //:hello [action 'Compiling hello.c', configuration: 680dcaf1caca59c28bd4314128d3f53c6fa1ff268a38933011ddb444ea1ec37b, execution platform: @@platforms//host:host, mnemonic: CppCompile] ...
INFO: Found 2 targets...
INFO: Elapsed time: 0.159s, Critical Path: 0.07s
INFO: 2 processes: 1 action cache hit, 1 internal, 1 darwin-sandbox.
INFO: Build completed successfully, 2 total actions
```


Note that this dirtiness state is tracked seprately from the evaluation state we saw earlier (e.g. `WAITING`). If you're curious, [this blog post](https://jmmv.dev/2020/12/google-no-clean-builds.html) covers the file change detection more in depth.

## Conclusion

Hopefully this gives you an overview of Skyframe, but I can't emphasize enough that this only scratches the surface. Some sources of complexity are:
- Build performance improvements with [Skymeld](https://github.com/bazelbuild/bazel/issues/14057)
- Optimizing memory usage with [Skyfocus](https://bazel.build/advanced/performance/memory#trade-flexibility)
- Reducing node restarts with [StateMachines](https://bazel.build/contribute/statemachine-guide)

With that being said, doing this research helped me build a basic intuition on how Skyframe works. I feel more confident keeping up with the latest changes in Bazel internals.

## References

- [`BuildTool.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/buildtool/BuildTool.java;l=192;drc=215ec5cd3a0aa09cd9ede99d132465f3c9c59000) is "crux of the build system" and is a great starting point to dig into the Bazel build process internals.
- [`AbstractParallelEvaluator.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/skyframe/AbstractParallelEvaluator.java;l=63?q=AbstractParallelEvaluator&ss=bazel%2Fbazel) manages the DAG node states
- [`AbstractInMemoryNodeEntry.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/skyframe/AbstractInMemoryNodeEntry.java;l=26?q=AbstractInMemoryNodeEntry.java&ss=bazel%2Fbazel) implements the DAG node class
- [`SkyframeExecutor.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/SkyframeExecutor.java;l=345?q=SkyframeExecutor.java&ss=bazel%2Fbazel) is the main wrapper class that drives the SkyFrame executions
- [`LocalDiffAwareness.java`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/LocalDiffAwareness.java;l=39?q=inotify&ss=bazel%2Fbazel) sets up the file system watcher
- All SkyFunctions are in [`src/main/java/com/google/devtools/build/lib/skyframe`](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/skyframe/) directory within the Bazel repository
