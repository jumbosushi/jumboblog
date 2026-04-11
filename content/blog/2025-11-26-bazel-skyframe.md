+++
date = '2025-11-26T22:35:35-08:00'
draft = true
title = 'Bazel Skyframe explained'
+++

<!-- https://excalidraw.com/#json=jBPiR7mOStUrB0pUPZEis,qvxLhEEFpNJd7Gbo1gR0BA -->

Someone reading into Bazel sooner or later runs into a concept called "Skyframe". But it's so hard to understand (<insert links>). This post goes over the architecture in closer details with an example

Good overview:
- https://bazel.build/versions/6.4.0/rules/challenges#coding-patterns
- https://bazel.build/reference/skyframe

For learning how it works:
- https://cs.opensource.google/bazel/
- https://deepwiki.com/bazelbuild/bazel

## Initialization

`SkyframeExecutor` intitializes `MemoizingEvaluator` interface implementation `InMemoryMemoizingEvaluator`

InMemoryGraphImpl is an in-memory graph implementation that uses ConcurrentMap

### Execution order
From deepwiki: https://deepwiki.com/bazelbuild/bazel/2.1-skyframe-execution-system

![execution](/images/2025-11-26-skyframe/sequence.png)

## TODO:
- [ ] Understand the big picture
- [ ] IMplement my own SkyFunction for debugging
