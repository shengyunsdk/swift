# Frequently Asked Questions

Please check here before posting a new issue.

### Why do I get ["error: array input is not a constant array of tensors"](https://github.com/tensorflow/swift/issues/10)?

If you ran into this error, you likely wrote some code using `Tensor` without running Swift with optimizations (`-O`). 

The `-O` flag enables optimizations and is currently required for the [graph program extraction
algorithm](https://github.com/tensorflow/swift/blob/master/docs/GraphProgramExtraction.md) to work correctly.
We're working on making `-O` not required, but in the meantime you need to specify it.

Here's how to enable optimizations in different environments:

* REPL: No need to add extra flags. Optimizations are on by default. 
* Interpter: `swift -O main.swift`
* Compiler: `swiftc -O main.swift`
* `swift build`: `swift build -Xswiftc -O`
* Xcode: Go to `Build Settings > Swift Compiler > Code Generation > Optimization Level` and select `Optimize for Speed [-O]`.
  * You may also need to add `libtensorflow.so` and `libtensorflow_framework.so` to `Linked Frameworks and Libraries` and change `Runtime Search Paths`.
    See [this comment](https://github.com/tensorflow/swift/issues/10#issuecomment-385167803) for specific instructions with screenshots.

### Why do I get ["error: internal error generating TensorFlow graph: GraphGen cannot lower a 'send/receive' to the host yet"](https://github.com/tensorflow/swift/issues/8)?

This error is related to the [graph program extraction algorithm](https://github.com/tensorflow/swift/blob/master/docs/GraphProgramExtraction.md), specifically
[host-graph communication](https://github.com/tensorflow/swift/blob/master/docs/GraphProgramExtraction.md#adding-hostgraph-communication).

If you ran into this error, you likely wrote some code using `Tensor`, like the following:

```swift
import TensorFlow
let x = Tensor([[1, 2], [3, 4]])
print(x)
let y = x + 1
```

Swift’s [graph program extraction algorithm](https://github.com/tensorflow/swift/blob/master/docs/GraphProgramExtraction.md)
assumes there are two "devices" running a program: the host and the accelerator.
When you run this program, the Swift compiler finds all of the `Tensor`-related code,
then extracts it and builds a TensorFlow graph to be run on on the accelerator.
In this case, the `Tensor` code to-be-extracted are lines 2 and 4 (the calculations of `x` and `y`).

However, notice line 3: it's a call to the `print` function (which must be run on the host),
and it requires the value of `x` (which is computed on the accelerator).
It's also not the last computation run in the graph (which is the calculation of `y`).

Under these circumstances, the compiler adds a "send" node in the TensorFlow graph
to send a copy of `x` to the host for printing.

Send/receive aren't fully implemented, which explains why you got your error.
The core team is working on it as a high-priority feature.
Once send/receive are done, your error should go away!

To work around the generated "send" for this example, you can reorder the code:

```swift
import TensorFlow
let x = Tensor([[1, 2], [3, 4]])
let y = x + 1
print(x)
```

The `Tensor` code is no longer "interrupted" by host code so there's no need for "send"!

### How can I use Python 3 with the `Python` module?

Currently, Swift is hard-coded to use Python 2.7.
Adding proper Python 3 support is non-trivial but in discussion.
See issue #13 for more information.