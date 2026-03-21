# Problem 7: Spicy

Spicy has its own value model. It does not align with Zeek's. Thus, there is marshalling between the Spicy value model and Zeek's. It can be slow when raising a lot of events with a lot of records, for example.

If we make a new value model for Zeek, and decouple it from Zeek, perhaps Zeek and Spicy can share a value model. Each could be used in isolation without ever including the other, then together, it would be zero-copy. Spicy can simply hand off the values to Zeek via a move. No more marshalling tax.

This means that Spicy has to change its output. But, this is another case for a decoupled value model. Spicy should function without Zeek (it is easier to maintain, debug, and can integrate with other systems). The value model needs to reflect that. We should use the same ideology for Zeek scripting values, then Spicy no longer needs to use its own value model in order to stay independent from Zeek.

A few extra problems worth considering:

1) Reference types: Spicy uses references, Zeek does not. We can simply transfer ownership, or if Spicy must see the values after it passes them along, `.clone()` them.
2) Spicy values "build up": This would make the record fields optional, but they *should not* always be optional at Zeek. For this, Spicy would use builders, then "finalize" them when passing along to Zeek. We could also *not* finalize for some custom host applications, if they want the optional values.
3) Rust: Spicy needs to emit Rust. We want to be efficient, so they should share the exact same value model without a middle ground.
4) Type info: *Zeek* would own type infos. Spicy uses the same type infos. Then, they can agree on structures ahead-of-time.

Remember, we hope that they share a value model now. This value model is decoupled from Zeek, so Spicy remains decoupled, but perhaps with an optional dependency (the runtime types and values would become a generic library that both use).

## Fibers

One big concern here is with fibers. If we swap stacks, then we cannot do zero-copy data sharing between Zeek and Spicy. We MUST copy stack data, there is no other way (I think :) ).

But, many cases are allocated on the heap, so won't become dangling later. Spicy internally won't even deep-copy value references when swapping the fiber stacks. We can use a similar approach: value references don't need copied!

Though, we have an extra case here. `Bytes` types (and `String`) have an underlying `std::string`. If we pass *these* to Zeek, they will *probably* be on the heap. But small-string optimization could mean it's not, which makes it a dangling reference. This may happen (I think) if that is not a part of a unit, like if we pass the return value of a function call to a Zeek event. We should make a string type that cannot be small-string optimized (there are a few ways to do this, it's just an interesting side case).

For the most part, data stored on the heap will be zero-copy, while data on the stack must be copied. The "handles" (like the value reference to a unit) will be copied, since that's on the stack. That is cheap since it should not need to deep copy.
