# Problem 8: AST Reduction

One of the phases in ZAM is "AST Reduction." This is a separate step in the process which transforms the Zeekscript AST into a form that is easier to lower.

A more general purpose compiler would likely use a separate intermediate representation. This serves a few purposes: first, you don't use the same machinery, so it has a nicer separation of concerns. Second, you can view which AST nodes reduced and how much easier. Third, you can choose separate optimizations for each step.

But, this is a case that we would probably stick with the AST reduction. If Zeek was a more general-purpose language, this would likely be too restrictive, and add complexity in debugging and "phases" of the AST. But, there is a lot of extra complexity with adding a new IR and transforming between them. It probably isn't worth the effort for Zeek.

# Proposal

Lower to the bytecode after AST reduction, and keep it as-is in Zeekscript.
