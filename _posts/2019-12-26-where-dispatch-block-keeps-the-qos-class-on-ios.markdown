---
date: '2019-12-26 13:48:00'
layout: post
slug: where-dispatch-block-keeps-the-qos-information-on-ios
status: publish
title: Where Dispatch Block Keeps the QoS Information on iOS?
categories:
- eyes
---
[Grand Central Dispatch](https://developer.apple.com/documentation/dispatch) is the de-facto task-based parallelism / scheduling system on macOS / iOS. It has been open-sourced as [libdispatch](https://github.com/apple/swift-corelibs-libdispatch) and ported to many platforms including Linux and FreeBSD.

libdispatch has been designed to work closely with the Clang extension: [Blocks](https://clang.llvm.org/docs/BlockLanguageSpec.html). Blocks is a simple, yet powerful function closure implementation that can implicitly capture variables to facilitate the design of task-based parallelism systems.

That choice imposed some constraints when designing the QoS classification system for libdispatch. Blocks' metadata is of the Clang's internal. It would leave a bad taste if we were required to modify Clang in order to add Blocks based QoS information. It would be interesting to discover how libdispatch engineers overcame these design dilemmas.

There are also some API limitations for the Blocks' QoS API. We cannot inspect the QoS assignments for a given block. That makes certain wrappers around libdispatch APIs challenging. For example, we cannot simply put a wrapper to account for how many blocks we executed like this:
```c
static atomic_int executed_count;

void my_dispatch_async(dispatch_queue_t queue, dispatch_block_t block) {
    dispatch_async(queue, ^{
        ++executed_count;
        block();
    });
}
```

The above could have unexpected behavior because the new block doesn't carry over the QoS assignment for the block passed in. For all we know, that block could be wrapped with [`dispatch_block_create_with_qos_class`](https://developer.apple.com/documentation/dispatch/1431068-dispatch_block_create_with_qos_c). Specifically:
```c
dispatch_block_t block = dispatch_block_create_with_qos_class(DISPATCH_BLOCK_ENFORCE_QOS_CLASS, QOS_USER_INITIATED, 0, old_block);
```

If dispatched, would lift the underlying queue's QoS to `QOS_USER_INITIATED`. However, with our wrapper `my_dispatch_async`, the QoS assignment will be stripped.

We would like to have a way at least to copy the QoS assignment over to the new block. This requires to inspect libdispatch internals.

## What is a Block?

Blocks is the function closure implementation from Clang that works across Objective-C, C and C++. Under the hood, it is really just a function pointer to a piece of code with some variables from the calling context copied over. Apple conveniently provided a header that specified exactly the layout of the Block metadata in memory:

[https://github.com/apple/swift-corelibs-libdispatch/blob/master/src/BlocksRuntime/Block_private.h#L59](https://github.com/apple/swift-corelibs-libdispatch/blob/master/src/BlocksRuntime/Block_private.h#L59)
```c
// ...
struct Block_descriptor_1 {
    unsigned long int reserved;
    unsigned long int size;
};
// ...
struct Block_layout {
    void *isa;
    volatile int32_t flags; // contains ref count
    int32_t reserved; 
    void (*invoke)(void *, ...);
    struct Block_descriptor_1 *descriptor;
    // imported variables
};
// ...
```

The first two fields just so happen to match the Objective-C object's memory layout. This will facilitate the requirement for Objective-C compatibility especially with [ARC](http://clang.llvm.org/docs/AutomaticReferenceCounting.html). The whole Block moved to the heap along with the imported variables in one allocation. Thus, if you have the pointer to the block metadata, you can already inspect captured variables if you know the exact order of their capturing.

At runtime, once a block is called, the compiler will restore the captured variables, and then cast and invoke `block->invoke` as if it is a normal function.

## The Additional Block Metadata

As we can see, the `Block_layout` is relatively tight with no much space for additional block metadata. How libdispatch engineers find the extra space for the QoS information?

The answer lies in another indirection:

[https://github.com/apple/swift-corelibs-libdispatch/blob/master/src/block.cpp#L113](https://github.com/apple/swift-corelibs-libdispatch/blob/master/src/block.cpp#L113)
```c
dispatch_block_t
_dispatch_block_create(dispatch_block_flags_t flags, voucher_t voucher,
		pthread_priority_t pri, dispatch_block_t block)
{
	struct dispatch_block_private_data_s dbpds(flags, voucher, pri, block);
	return reinterpret_cast<dispatch_block_t>(_dispatch_Block_copy(^{
		// Capture stack object: invokes copy constructor (17094902)
		(void)dbpds;
		_dispatch_block_invoke_direct(&dbpds);
	}));
}
```

`dispatch_block_create` or `dispatch_block_create_with_qos_class` ultimately calls into this `_dispatch_block_create` private function.

It captures a particular variable `dbpds` that contains [numerous fields](https://github.com/apple/swift-corelibs-libdispatch/blob/4659503fee11a3c0cad79a771de53dbde0ca92cc/src/queue_internal.h#L1189) onto the block, and then invoke the actual block directly.

As we can see in the previous section, it is relatively easy to inspect the captured variables if you know the actual layout. It just happens we know the layout of `struct dispatch_block_private_data_s` exactly.

## Copying QoS Metadata

Back to the previously mentioned `my_dispatch_async` implementation. If we want to maintain the QoS metadata, we need to copy it over to the new block. Now we have cleared the skeleton, there are only a few implementation details.

First, we cannot directly inspect the captured variables.

It is straightforward to cast `(struct dispatch_block_private_data_s *)((uint8_t *)block + sizeof(Block_layout))`, and then check the fields. However, there is no guarantee that a passed-in block is wrapped with `dispatch_block_create` method always. If a passed-in block happens to contain no captured variables, you may access out-of-bound memory address.

The way libdispatch implemented is to first check the `invoke` function pointer. If it is wrapped with `dispatch_block_create`, it will always point to the same function inside the `block.cpp` implementation. We can find this function pointer at link time like [what libdispatch did](https://github.com/apple/swift-corelibs-libdispatch/blob/master/src/block.cpp#L121) or we can find it at runtime.
```c
typedef void (*dispatch_f)(void*, ...);
dispatch_f dispatch_block_special_invoke()
{
    static dispatch_once_t onceToken;
    static dispatch_f f;
    dispatch_once(&onceToken, ^{
        f = (__bridge struct Block_layout *)dispatch_block_create(DISPATCH_BLOCK_INHERIT_QOS_CLASS, ^{})->invoke;
    });
    return f;
}
```

Second, we need to deal with runtime changes. We don't expect libdispatch has dramatic updates to its internals, however, it is better safe than sorry. Luckily, `struct dispatch_block_private_data_s` has a magic number to compare notes. We can simply check `dbpds->dbpd_magic` against library updates and corruptions.

Finally, we can assemble our `my_dispatch_async` method properly.
```c
static atomic_int executed_count;

void my_dispatch_async(dispatch_queue_t queue, dispatch_block_t block) {
    dispatch_block_t wrapped_block = ^{
        ++executed_count;
        block();
    };
    struct Block_layout *old_block_layout = (__bridge struct Block_layout *)block;
    if (old_block_layout->invoke == dispatch_block_special_invoke()) {
        wrapped_block = dispatch_block_create(DISPATCH_BLOCK_INHERIT_QOS_CLASS, wrapped_block);
        struct Block_layout *wrapped_block_layout = (__bridge struct Block_layout *)wrapped_block;
        struct dispatch_block_private_data_s *old_dbpds = (struct dispatch_block_private_data_s *)(old_block_layout + 1);
        struct dispatch_block_private_data_s *wrapped_dbpds = (struct dispatch_block_private_data_s *)(wrapped_block_layout + 1);
        if (old_dbpds->dbpd_magic == 0xD159B10C) {
            wrapped_dbpds->dbpd_flags = old_dbpds->dbpd_flags;
            wrapped_dbpds->dbpd_priority = old_dbpds->dbpd_priority;
        }
    }
    dispatch_async(queue, wrapped_block);
}
```

This new `my_dispatch_async` wrapper now will respect the block QoS assignments passed in, you can check this by dispatch a block with `dispatch_block_create` and observe the executed QoS with `qos_class_self()`.

## Closing Thoughts

The implementation of QoS in dispatch block is quite indigenous. However, it does present challenges outside of libdispatch scope. This implementation is specialized against `dispatch_block_t` type of blocks, you cannot simply extend that to other types of blocks. I am particularly not happy that `dispatch_block_create` is not a generic function such that any given block, parameterized or not can have QoS wrapped and somehow respected (for example, taking its QoS out and assign it to a plain `dispatch_block_t` when you do `dispatch_async` dance).

Implementing your own QoS-carrying block this way would be quite painful. Each parameterized block would require a specialized function that carries the QoS information. You probably can do that with C macro hackery, but that would be ugly too quickly. You'd better off to have an object that takes both the block and QoS information plainly, than trying to be clever and embedding the QoS information into the block.