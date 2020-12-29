---
date: '2020-12-06 18:27:00'
layout: post
slug: on-designing-deep-learning-library-api-in-swift
status: publish
title: On Designing Deep Learning Library API in Swift
categories:
- eyes
---

From the onset of implementing [libnnc](https://libnnc.org), it meant to be a common ground for higher-level language bindings beyond Python. The underlying architecture has been stable for a year or so, and I have been using it for some personal projects for a while. But the raw C interface is not the easiest to use, it is the time to implement some high-level language bindings for that library.

The default high-level language for deep learning likely would be Python. However, I am not happy with how it performs on a typical many-core system even with things that are supposed to help. Swift, on the other hand, has no issues with saturating my many-core system and it has a reasonable [Python binding](https://github.com/pvieito/PythonKit) to tap into the rich Python tool-kits. Not to mention the *calling C functions from Swift* path is as easy as you possibly can get.

This conviction resulted [s4nnc](https://github.com/liuliu/s4nnc), a Swift language binding for [libnnc](https://libnnc.org). Because [s4nnc](https://github.com/liuliu/s4nnc) is a pure interface to interact with the underlying deep learning library. I paid close attention to its API design. Below are some design notes around why it is, and how Swift as a language fares on such a task. If you want to read the introduction to s4nnc, feel free to visit the [GitHub homepage](https://github.com/liuliu/s4nnc).

### What is a Deep Learning Library API

A good deep learning API to me, can be largely modeled after [Keras](https://keras.io) and [PyTorch](https://pytorch.org). It concerns, above all, with 2 questions:

 1. How to specify a deep learning model?
 2. How to construct a training loop?

Everything else is nice-to-have and largely orthogonal to these two questions (but these whistles-and-bells are a lot of hard work!).

A training loop consists of a repeated sequence of operations to: evaluate a model, compute gradients against the loss, apply gradients to update model parameters.

The details can be flexible: you could evaluate one part of the model in one round, and another part in another round; you could have different losses for different model outputs each round; you could modify the gradients, scale them, truncate them to whatever you liked; and you could apply gradients to update different model parameters with different optimizers. But at the core, I didn't see much changes for this 3-step throughout many model training code.

What constitutes a model can be more interesting, but it seems we converged to a concept where a model consists of some inputs, some outputs, and some parameters. Particularly, parameters are stateful and internals to the model itself. Go beyond that, a model could have different inputs / outputs and different input / output shapes during the training loop. However, the shapes and number of parameters during the training are likely to be constant.

### Basic Data Types

A deep learning library operates on multi-dimensional arrays (or tensors). In Swift, a concrete tensor can be represented as a value type like `Array` itself in Swift. That means the `Tensor` type would need things such as copy-on-write to implement said value-type semantics. Extra attention needs to be paid to make sure throughout the implementation of the API, no unnecessary copy was made. This value-type choice is a bigger deal than it sounds (it sounds like a no-brainer given [S4TF](https://www.tensorflow.org/swift) made exactly the same choice) because in Python, everything is a reference type.

This becomes more interesting when deciding whether [tensor variables](https://libnnc.org/tech/nnc-dy/) could be value types or not. Tensor variable is an abstract tensor type which you can compute gradients (has a `grad` property). It is bound to a computation graph, and can be the parameter to update during the training loop. While it is possible to make many functions associated with tensor variables taking `inout` parameters and marking some of tensor variables' functions as `mutating`, the more hairy part is about updates during the training loop.

In [PyTorch](https://pytorch.org/docs/stable/optim.html), an optimizer takes a list of parameters and then applies new gradients to update these parameters when `step()` method is called. This is possible because parameters are reference types in Python. Any updates to the parameters will be reflected to the model who holds these parameters. This is not possible if tensor variables are value types. In Swift, you cannot hold a reference to a value type.

Thus, practically, tensor variables have to be implemented as reference types in Swift.

Despite my best intention, it turns out most of the objects, including `Graph`, `Model`, `StreamContext` are still implemented as reference types. It is possible for some of them (for example: the `Model`) to be value types. The lack of `deinit` in `struct` requires us to wrap a reference type inside a value type to create such API. At the end of day, I don't see much of the value from API aesthetics or performance-wise to make these value types.

### Automatic Differentiation

While Swift has [a proposal for automatic differentiation](https://forums.swift.org/t/differentiable-programming-for-gradient-based-machine-learning/42147), the automatic differentiation right now is implemented at library level and only applicable to models and tensor variables.

On the API side, it is popular to have a `backward()` method on the final loss variable. The said method will compute gradients against all variables associated with the final computation of the loss.

This also means we need to keep track of all variables in the computation graph, unless some point is reached and we can free them. In PyTorch, such point is when the `step()` method is called.

[libnnc early on made the decision to avoid holding vast amount of memory by default](https://libnnc.org/tech/nnc-dy/#optimizations). That resulted in the interface `backward(to tensors: Sequence<Tensor>)` where you have to specify to which tensors you compute the gradients against. Because we are doing backward-mode AD, we still compute gradients on all variables up until these tensors. But we don't compute gradients against variables passed that point. In effect, we can rely on reference-counting to free memory associated with tensors beyond that point.

In return for this a bit more complex interface, you don't have to worry about scoping to `no_grad` to avoid unbounded memory allocations.

### Optimizers in the Training Loop

An optimizer in a deep learning library represents a particular gradient descent method associated with some parameters to update each round.

One particular challenge is about how to use multiple optimizers with different parameters in one round. While for simpler cases, you could call `step()` many times in one round. It may be more efficient to call `step()` once.

Swift makes this particular choice easier by supporting extensions on built-in types.

```swift
public extension Collection where Element: Optimizer {
  func step() {
    ...
  }
}
```

This is a good starting point to support more concise updates such as: `[adam1, adam2].step()`.

The same pattern can be applied if you want to support gradients with multiple losses: `[loss1, loss2, loss3].backward(to: x)`.

These extension methods are conditional, and type-safe in Swift.

### Operators

While Swift allows operator overloading and the ability to introduce custom operators, somehow the ambiguity of `*` is not addressed. We cannot have consistency with Python because `@` cannot be overloaded in Swift language. I have to resort to `.*` and `.+` for element-wise multiplications and additions.

### Type-safe Functions

While Swift can enforce some type consistency, without more language level changes, we cannot deduce shapes, and would still encounter runtime errors if shape doesn't match. Even with language-level support, we may still need an escape-hatch because some tensors could be loaded from IO. Not to mention it would be nice to support dynamic input shapes to a model while it can still statically compute its parameter shapes.

### Transparent Multi-GPU Data Parallel Training

One thing annoying in the raw C interface, is the transition from one GPU to multi-GPU, even with simple data-parallel models. Unable to abstract tensor to higher-level in C makes the code unnecessarily complex.

Fortunately, with basic generics, this is not a problem in Swift. However, to use such generics turns out to be more complicated than I expected on the library author side. If I want to avoid runtime type-check (`is` / `as` keyword), there are quite a bit of protocol / extension type dance I would need to do. This doesn't help with Swift's borderline hostility against protocol-associated types. Luckily, there are [new proposals](https://forums.swift.org/t/improving-the-ui-of-generics/22814/) to lift some of the restrictions (and no, `some` is not all it needs). You can see [this monstrosity here](https://github.com/liuliu/s4nnc/blob/main/nnc/Functional.swift#L3). At the end, I have to introduce [some](https://github.com/liuliu/s4nnc/blob/main/nnc/Optimizer.swift#L125) [runtime](https://github.com/liuliu/s4nnc/blob/main/nnc/ModelBuilder.swift#L65) [type-checks](https://github.com/liuliu/s4nnc/blob/main/nnc/Store.swift#L31) to keep my sanity.

### Closing Words

I am pretty happy with the end result of [s4nnc](https://github.com/liuliu/s4nnc). It is small, versatile and does exactly what I set out to: an ergonomics win from the raw C interface. Generics, type-safety and reference-counting in Swift the language really made a difference in the API ergonomics. The language itself is not the fastest, but has a good balance in ergonomics, expressivity and performance. In the next blog post, I am going to detail the Swift data science workflow I had, and why we moved to this from the initial Python one. Stay tuned!