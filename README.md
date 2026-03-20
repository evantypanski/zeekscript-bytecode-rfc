# Zeekscript Bytecode RFC

> [!IMPORTANT]
> AI usage note: AI did not write a single character in this document. However, I did use AI as a search tool and to "rubber ducky" my ideas. This is based on my understanding of Zeek and its scripting language, and therefore may be incorrect, but any shortcomings are entirely my own. I would appreciate if any AI usage regarding this document is disclosed and used sparingly, as my experience is that it gets some core principles in Zeek's internals incorrect, which leads it to incorrect conclusions. Thanks.

This is an RFC for a new Zeekscript bytecode.

## Motivations

While this document will lay out some problems and solutions to various issues within Zeek, its scripting language, and its integrations with other languages, there is another realm of motivations that guide this document. I will list them here:

1) Tree-walking interpreters are not made for production environments. They are okay for a domain-specific language or a toy language. But, they simply are not cut out for a system like Zeek. They are difficult to debug and can be hard to reason about. They are very inefficient. They are difficult to optimize. Though, it may be most important to say that they aren't enjoyable to work on. I have wanted to work on Zeek script and simply haven't since dealing with this tree-walking interpreter feels bad to me.
2) The core and interpreter influence each other too much. I find that changes to the core may be more difficult since it also affects the interpreter. You see this most with analyzers that create values (I just found one example of the binpac SSL analyzer making record values [here](https://github.com/zeek/zeek/blob/06b43274bf51759551d624bae119839a979b6b48/src/analyzer/protocol/ssl/tls-handshake-analyzer.pac#L260-L263), but take any analyzer and it does something like this). Of course these are cases where they are providing values to the script layer, so this is necessary regardless. But is it not odd to hardcode offsets for the field numbers directly into the analyzer? Then the analyzer depends on the script layout. These analyzers are now tightly coupled with the script execution and script layout. This is not going to immediately get fixed, but this tight coupling can be rough, annoying, and bug prone.
3) ZAM should have always been the one true way forward. We *have* a bytecode. We *have* a faster execution engine. But, for various reasons, it is *also* tightly coupled to the interpter. ZAM bodies are drop-in replacements within the scripting language. `ZVal` falls back to old values, which then get "hardcoded" into the bytecode, so it can't exist without having a whole compilation phase. It's an optimization engine, but it could be the way to decouple script execution from Zeek's core.
4) Zeek script is here to stay. I don't see a world where Zeek script dies, and we fully transition to X other language. It's worth it to invest in this space, since it will be relevant for a *long* time.
5) Integrations are difficult. This goes in with tight coupling, but we should be able to integrate a bit more easily.
6) Zeek could do with a refresh. There are various aspects that just feel outdated. Notably, BiFs are just a foreign function interface. If we let users write them in `.cc` files, they get language server integration. We could do so much more, but it's currently a pain to do a number of things in Zeek. We can revisit these as they get hit and see if they can be done better. It's an opportunity to make Zeek better in ways we don't currently envision.

Notably, none of these are "Zeek scripts should run faster." I mainly care about my experience working on Zeek, and other people's experience using it. I think that Zeek simply isn't as nice for maintainers to work on, or others to develop with, as it could be. We should take an opportunity when we see it and try to fix that.

## How to read this document

Each section is named in order (eg `00-` means the `0th` document, `01-` means the `1st` document, anything in `02-` means the second documents, etc. It makes sense to read the intro, since that goes more into motivations, then the proposed architecture (so `01-intro.md` and `03-proposed-architecture.md`). If you have time, feel free to read more :)

Also a note, this is not meant to be a complete plan, since we will (of course) find issues when implementing and change a lot. But, it's an attempt to get some details figured out to not go in blind.
