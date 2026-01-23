# Problem 2: The bytecode should be printable

One issue with ZAM is that starting it up, you get a long phase of compiling the scripts down into ZAM. We need a better solution for this. To do this, the bytecode must be printable. This means that all values used within a given bytecode are in a constant pool. Furthermore, any function calls must get the address somehow.

## Constant Pool

The constant pool will be a vector of a predefined constant. These constants will just be like:

```
enum Constant {
  Int(i64),
  Count(u64),
  String(String),
  Vector(Vec<u32>),
  PrimitiveType(TypeTag), // not sure how these will look
  AggregateType(u32),
  // Function pointers and types would be constants too!!
}
```

On-disk, each constant would start with its type, then proceed to some number of bytes of data depending on the type.

For example, here is how a vector with integer elements `1`, `2`, and `3` might look (for demonstration, integer widths are 4 bytes):

```
+------------------------------------------------------+
|                  Constant Pool                       |
+--------+-------+-------------------------------------+
| offset | bytes | description                         |
+--------+-------+-------------------------------------+
| 0      | 3     | Vector type tag                     |
| 1..4   | 21    | element type index (const index 21) |
| 5..8   | 3     | length                              |
| 9..12  | 24    | element 0 (const index 24)          |
| 13..16 | 29    | element 1 (const index 29)          |
| 17..20 | 34    | element 2 (const index 34)          |
+--------+-------+-------------------------------------+
| 21     | 4     | `type` type tag                     |
| 22     | 0     | 0 describes it as primitive         |
| 23     | 1     | `int` type tag                      |
+--------+-------+-------------------------------------+
| 24     | 0     | `int` type tag                      |
| 25..28 | 1     | `int` value of `1`                  |
+--------+-------+-------------------------------------+
| 29     | 0     | `int` type tag                      |
| 30..33 | 2     | `int` value of `2`                  |
+--------+-------+-------------------------------------+
| 34     | 0     | `int` type tag                      |
| 35..38 | 3     | `int` value of `3`                  |
+--------+-------+-------------------------------------+
```

> [!NOTE]
> We can define the constant pool layout however we want. This is just an example.

Then, we just have instructions to access a particular constant. We can convert this constant into `ZVal` easily.

For this, cycles will not be allowed. This simplifies the constant pool substantially.

## Functions

Functions are just special globals. Say we want to load the function `test()` from Zeekscript, like:

```
local x = test();
```

Then, we would store a constant at slot 0 as the string `"test"`:

```
CONSTANTS:
  #0: String("test")
```

And the VM would use an instruction to load this from persistent memory. This would be through a map lookup of global name to persistent memory address.

This means that functions are just any other global, just with a function type. This same logic applies to anything from another script.

> [!NOTE]
> Since we look it up in a hash map, it could be slow. Once we run it once, we can cache where it lives in memory. This is called [inline caching](https://en.wikipedia.org/wiki/Inline_caching).
>
> Or, since these are constants, we can resolve before ever running the program.

## Exports

A script may export many things. We need a separate section that describes which identifiers get exported and what their values are.

First, the value is defined in the constant pool. Then, we add an entry to the exports list, which maps a global name to the index into the constant pool. Then, the VM will register all of the exported names into persistent memory (likely when loaded).

Since we will need to distinguish between what is getting exported, this could look like:

```
enum ExportKind {
  Value(ConstIdx), // Just a value in the constant table
  Function(ConstIdx),
  GlobalValue(ConstIdx),
  Type(ConstIdx),
};

// Each export is defined by this very compact type:
struct Export {
  name: String,
  kind: ExportKind,
};
```

# Proposal

The bytecode itself would be extremely simple. Imagine the Rust struct as:

```
struct CodeBlock {
  name: String, // Function name
  frame_slot_count: u32, // Potentially needed
  instrs: Vec<Instruction>, // This function's body
  consts: Vec<Constant>, // Each function has its own constant pool for now
  debug_info: Option<DebugInfo>, // More on this later!
}

struct Bytecode {
  // We potentially need to point to a code block for top-level statements, too
  blocks: Vec<CodeBlock>,
  exports: Vec<Export>,
}
```

Then, the constants are all defined entirely within the file and read into memory as `Constant`s. Then, these get converted into `ZVal` with some resolution step (or, we convert them lazily and cache the result!). Instructions will all point to frame slots (which get populated elsewhere) or constants, so everything is self-contained.
