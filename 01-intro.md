# Introduction

Zeek scripting is one of the primary mechanisms that users interact with Zeek. Even without writing detections, users often `redef` constants or setup their environment in scripts. Even then, they use other scripts. Thus, it's a core part of Zeek.

From the Zeek community survey from June 23 to July 17, 2025, 74.1% of respondents (20/27) who were asked "What's hard about getting started with Zeek?" responded with "Understanding the scripting language." This is the only choice that had a majority (it was a select-all-that-apply question). Many of the user-provided pain points also mentioned scripting. This makes it clear that scripting is a pain point for many users.

The problem here is likely a mix of factors. The most obvious is that the documentation and learning resources are lacking. We are addressing this with a new tutorial for Zeek, alongside video tutorials on scripting.

Another reason people likely find scripting hard to grasp is because it is unlike many other languages. A majority of users rate themselves "Beginner" (21.1%) or "Intermediate" (38.9%). Most respondents describe their role as "Security Engineer," (58.9%) "Incident Responder," (32.2%) and "SOC Analyst" (25.6%). These are typically not roles that "live" in the programming language world and can easily transfer skills among a set of languages. Instead, they likely know a few languages well, and use them in order to perform the tasks that they care about.

Zeek scripting is particularly bad for these types of users. The language takes programming principles and flips them on their head. Field accesses use a dollar sign (`$`), which still feels wrong to many. Looping through a vector will retrieve indices; you must use `for (_, val in my_vec)` in order to get values. Adding elements to a set uses the `add` statement, but adding to a vector uses `+=`. Each of these may well have good reasons, but the result is that the end user has to learn new syntax, new semantics for common concepts, and hold each of these new ideas in their head as they navigate *within* the language.

But, in its current form, we simply don't have the resources to fix these issues. This comes down to two major problems:

1) Changing this behavior will break existing code with no good deprecation.
2) The internal script execution is too frustrating to work with, so we do not want to spend significant time changing or adding features.

The first is a fundamental problem: we simply cannot leave existing code, that has worked for a decade, in a state that it cannot function under any circumstance. There are too many existing codebases written using this engine to completely rip some parts up in one release, even with deprecations. The solution here would be Zeek script 2.0, or another language.

So, we must consider that Zeek scripting, as a language, has shortcomings that we should address in order to make the language better. This is a syntax and language design problem, which itself is interesting. But, we cannot address this in the current core. As-is, Zeek scripts are still executed with the tree walking interpreter. If we want to have a large batch of syntax and language updates, this should almost certainly be alongside existing syntax. We should have a system that enables both.

That is a discussion for another day, but should be kept in the back of your mind as you read on: we should try to make a solution enable this.

The second is completely solvable in its current state. As-is, the Zeek scripting execution uses a **tree-walking interpreter**. That means that it creates C++ values for Expressions (which produce values), Statements (which are executed code that do not produce values), functions, and more. The engine sees that it parsed some `EqExpr`, then goes into the exact structure it got from parsing and evaluates it. While evaluating, it will ask two other expressions (its children) to evaluate themselves and return the results.

This is an impractical design for an interpreter for a few reasons, which I will not get into. The primary one is that it is slow (it uses virtual methods which require double-pointer indirection). The second is that it cannot be easily optimized with modern optimization techniques. The third is that it is hard to debug, as you end up being deeply nested in virtual function calls.

The language itself is very tightly tied to its syntax.

## ZAM

Because of this, the Zeek Abstract Machine (ZAM) was created. This is effectively a bytecode: instead of traversing a tree, we are walking through a linear list of instructions. Created values are simply "frame objects" stored on a per-function basis. *This works!* Not only that, this is the proper way for a modern scripting language to work internally.

However, ZAM is much more of an optimization on existing infrastructure. There are a few points here worth mentioning:

1) You cannot simply print ZAM as a binary, then load that. This would reduce parsing time (and initial startup), as well as potentially providing some intellectual property protection since you wouldn't need to ship Zeek script content.
2) ZAM still falls back to Zeek scripting values for some complex values. This is a primary reason that it cannot be printed.
3) ZAM relies on C++ memory management, which leads to segfaults or incorrect reference values if computed incorrectly.

Thus, I believe ZAM is extremely useful as a bytecode layout which is capable of optimizations that we should leverage. We should, however, strive to make it better. Its instruction set can very well be left relatively the same, as well as many of the optimizations and a chunk of the internals.

One extra issue that ZAM has is that the core Zeek team was relatively uninvolved with its creation and development. It seems problematic for one person who is not officially a maintainer to maintain what should be a core component of Zeek. Of course, we as maintainers should have a detailed understanding of all code in Zeek's core, but that has fallen behind.

## Zeek's future considerations

The third point above extends beyond ZAM, or Zeek script in general (the language has had its fair share of segfaults during execution). C++ is not a safe language. Zeek's core maintainers do not particularly like it. Contributors to the project are thrown off by it being written in C++. When contributor code is reviewed, we cannot simply review for correctness, much care is given to ensure that the code will not produce memory issues.

There is a solution here. Fish shell used to be written in C++. They [rewrote it in Rust](https://fishshell.com/blog/rustport/) with the partial goal of making it *fun* to work on Rust. Before the change, they had 17 people create at least 10 commits to the C++ code. They didn't provide a direct comparison, but they had over 200 authors contribute to the rewrite. To quote a [Mastodon post](https://aus.social/@zanchey/111760405118706568):

> However, some of the social goals have definitely been achieved. Large parts of the rewrite came from contributors who had never worked on fish before. There's been a lot of buzz in various online fora

Zeek simply does not have much buzz with developers, nor do we get one of the true benefits of open source software, namely external contributors. People do not want to work on Zeek, but we can make it easier, and more fun, to contribute.

Rewriting Zeek in Rust is a non-starter. But, we can write a new, core component of Zeek. This is a pretty clear area where Rust would be beneficial. It's very efficient, so we don't have performance losses. It's safe(-er), which is greatly important since the interpreter works with user code. We don't want a user to maliciously cause memory issues in a Zeek install. It also introduces the capability of introducing Rust to Zeek's core, which will help for future projects which may benefit. This expands to external plugins, where people may be more likely to contribute if they can write it in Rust.

Though, the biggest benefit is that Zeek's core is tightly coupled with its scripting language. This was, almost certainly, an advantage at one point. If there is no boundary between Zeek scripts and the core, you can directly manipulate the core to your liking! But, this has simply gotten too far. When making record values, the core Zeek code uses hard-coded integer offsets. We dynamically allocate values that get passed into the scripting layer. Creating the exact values used in the scripting layer means that any changes to the *core* change the *script* behavior.

The interpreter can be decoupled. In a hypothetical new world, the core can ask the interpreter to create a value, then if it needs to retrieve the values, it asks the interpreter. This means that the *interpreter* is uniquely in charge of its domain. The boundary here is important to get right, so that we do not have unnecessary marshalling (ie converting between Zeek scripting and C++ values). I believe this problem is surmountable.


Now, the interpreter will have a boundary which Zeek's core interacts with. We can change the core without the interpreter and vice-versa. The *hope* is that we can feel more comfortable with more changes, since we can just adapt that boundary if something in the core conflicts with the "old way."

If we write the interpreter in Rust, then this boundary *has* to be carefully considered. We are forced to design this properly. We can see exactly where and when the boundary is an issue and correct it. We can think about each side as its own component. The interpreter must, then, stand alone, which makes it easier to test and verify.
