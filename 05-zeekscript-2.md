# Zeekscript 2.0

One of the primary reasons for building a bytecode is that we should consider how viable a new Zeekscript is. The current language has technically worked, but the world has simply moved on from many of its idioms.

We must consider that the reason people don't write Zeek scripts is because no one wants to. The language is not fun to write. It's not even consistent, many features are done one way for some types, and another for another type. It's the product of years without a coherent goal.

The first question to ask is: does Zeekscript have to be a great language? The answer is no, of course not. Many people will admit it has its quirks, but the functionality is the primary concern here. The Zeek team uses Zeekscript in order to add new analyzers and update existing ones. Companies and some users write scripts to do what they need.

## What about other languages?

We already have [ZeekJS](https://github.com/corelight/zeekjs). What if we hook into that more?

It's pretty reasonable to say that if we could start over, we should now use a library, or simple existing language runtime, for Zeek. But, as-is, we have to hook it in to the existing infrastructure. This means that users have to know certain Zeek scripting quirks in order to work with it. Still, it is almost certainly a better option to get started with Zeek than Zeekscript is.

One primary concern is: What do we do about all of the Zeek scripts within the Zeek tree? Will we rewrite it in Javascript? That seems pretty unlikely.

The problem here is mostly that there isn't a clear direction. And with that, we embed a Javascript runtime, which should be seen as a very radical decision.

A proposal worth considering is that a *runtime* might be the wrong way to think about this. If you want to write Zeek-compatible code, you now have a bytecode (with this proposal). All you need is a library which would build up this bytecode and print it out. The programmer can think higher level, and use languages like Python, in order to build up a Zeekscript. Then, they finish it and print it out!

Or, we could create a runtime and bake it more deeply into the VM. This could work well with Javascript: if we choose to keep it, try to bake its ideas into the VM in such a way that we can use it.

But, what if we want a way forward for Zeekscripts? The existing scripts still exist, and we will need to consider them. Making a whole new bytecode reinforces that this is an important avenue. What should be their evolution?

## Why keep a DSL?

Zeekscript can (loosely, now) be described as a domain specific language (DSL). This is an interesting take since it means we can treat certain obvious patterns as first-class citizens (addresses, subnets, etc.). We can even optimize these for common patterns, which is awesome!

The network traffic domain is also quite interesting, since it has unique characteristics. Values' lifetimes are quite tied to the lifetime of a *packet* and *connection*. These are fundamental language problems that can shape how you think about a problem, in other languages they may get muddied.

WIP

## Switching to Zeekscript 2.0

But, Zeekscript is here, and it's not going to swap unless people have to. If we just force people to use new semantics, the obvious response will be "don't update." That wouldn't be good for anyone.

There are strategies for this, though!

### Automagic Migration

The first, and necessary, step would be: make it easy.
