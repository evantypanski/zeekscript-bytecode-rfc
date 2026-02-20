# Problem 7: Spicy

Spicy has its own value model. It does not align with Zeek's. Thus, there is marshalling between the Spicy value model and Zeek's. It can be slow when raising a lot of events with a lot of records, for example.

If we make a new value model for Zeek, and decouple it from Zeek, perhaps Zeek and Spicy can share a value model. Each could be used in isolation without ever including the other, then together, it would be zero-copy. Spicy can simply hand off the values to Zeek via a move. No more marshalling tax.

This means that Spicy has to change its output. But, this is another case for a decoupled value model. Spicy should function without Zeek (it is easier to maintain, debug, and can integrate with other systems). The value model needs to reflect that. We should use the same ideology for Zeek scripting values, then Spicy no longer needs to use its own value model in order to stay independent from Zeek.
