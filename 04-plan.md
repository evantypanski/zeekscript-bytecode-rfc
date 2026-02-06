# The Transition Plan

The full plan here is pretty tough to go through fully: this model will like change (quite heavily) as we learn from doing. Goals will change and new priorities will arise.

Nonetheless, the goal will not be to provide an accurate *timeframe*, as that feels impossible at this stage. The guess there is on the order of years. Instead, this will have a few key phases for the project, and what that means for the rest of the project at any particular part.

> [!NOTE]
> There is a large incentive to keep development in-tree within Zeek. This means that development would be in the same code that other Zeek code is, but is not built by default. This means that we can have a full history, as well as an incremental understanding of the code. It's very difficult to be handed large chunks of code and told "maintain this." Because of this, we will assume that this is incrementally developed in-tree just like any bug fixes or other features are.

## Phase 1: Separation

The first phase would be pretty isolated from Zeek itself. A new interpreter must be able to act independently: it aids with debugging and with easy switching between interpreters. Thus, the first phase would happen without integration with Zeek *at all*. The first step is to create a relatively minimal implementation of ZAM's bytecode in order to build out basic functionality.

This is not meant to be complete! It's just meant to be enough to do the basic things we may want to test further developments.

Here, we would establish basic tests (including for errors, bytecode, and simple utilities like serializing/deserializing). We would ensure the VM is easy and fun (!) to develop, and potentially modify ZAM as necessary in order to properly serialize. Optimizations should be baked into the flow here, even if they are rudimentary, to ensure it's always easy to rewrite the bytecode and values to improve performance.

## Phase 2: Integration

Now, we would bring the new VM into Zeek's build process and integrate it with existing C++. Here, we can build up the "legacy value" and `ZVal` integration. We can create the C shims to call BiFs.

One missing point: we will not touch Zeekscript. The bytecode will still be completely separate, it is just the integration of bytecode with existing Zeek infrastructure here.

## Phase 3: The Compiler

Now, we have a working mechanism to work with the two interpreters. We now need a compiler to make the Zeekscript into the bytecode.

This is just a compiler to generate a bytecode. Nothing more, nothing less. We can now add end-to-end tests of executing Zeekscripts with the new interpreter.

Inevitably, we will also find issues with the bytecode structure and adjust it accordingly. But, this phase is almost entirely about getting many of the Zeek constructs down into a bytecode.

## Phase 4: Refinement

At this point, everything will be pretty rough. Error messages probably won't be pretty, we'll have type inconsistencies. Here is where we spend significant effort on:

1) Error messages
2) Type checking
3) Debugging

Each of these will have forms in earlier versions, since that enhances developer experience. But, now we will get this to user-facing levels. Error messages should have information about the source code if available, potentially even printing out offinding lines. We should add various semantic analyses in order to ensure the bytecode is actually valid. Finally, we need a bunch of debug streams and utilities to make finding bugs and fixing them as easy as possible.

## Phase 5: Move Away

At this point, we should aim to move fully away from the legacy interpreter. Its influence will only be negative when moving forward, as jumping the FFI boundary and working with an entirely different system will prove very difficult.

This means that the compiler must now compile *any* valid Zeekscript into the bytecode, and Zeek should work well with only bytecode provided. Complete the compiler.

Of course, we will need to adjust the bytecode and VM accordingly, likely in unpredictable ways.

## Phase 6: Optimize

The first step would be to have true benchmarks. Since everything is within the new VM at this point, a true benchmark would get a true result.

We should already have some basic optimizations, but now we can go overboard. Once we load everything, we know it is static. We can start examining some more complex optimizations. These might include:

1) Interprocedural optimizations
2) JIT compilation
3) More complex local analyses

Essentially, we can make a real bytecode, like Ruby, Python, or anything else. This is the fun part.

> [!NOTE]
> It's unlikely that we need very crazy optimizations. It might be fun though.

## Phase 7: Zeekscript 2?

Finally, this can just be the start. If Zeek bytecode is just a compilation target, we can look at a bunch of new options for user-facing experiences with Zeek. Do we want to make a new language, just with some quirks ironed out? Do we want to move fully to an established language? Do we want to just provide binaries that write Zeek bytecode?

The possibilities are huge at this point, many more than we could get without the bytecode. We should take full advantage.

# The Project

As a whole, this is a new bytecode for an existing language. We will have a LOT of quirks to iron out. We also will change behavior, inevitably. These are bugs, since if we break existing code, people will just not upgrade.

But, that means it will take time. This document has many mistakes that will not work, many of which we will not notice until the project starts. Instead of a complete plan, think of this as a path to prove that this *could* work, even if it's not necessarily in this way.
