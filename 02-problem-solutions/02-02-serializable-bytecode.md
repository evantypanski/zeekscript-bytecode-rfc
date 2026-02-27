# Problem 2: The bytecode should be serializable

One issue with ZAM is that starting it up, you get a long phase of compiling the scripts down into ZAM. We need a better solution for this. To do this, the bytecode must be serializable. This means that all values used within a given bytecode are in a constant pool. Furthermore, any function calls must get the address somehow.

The bytecode itself is trivial to serialize by design: an opcode is a byte followed by a known number of bytes. These bytes mean special things to that opcode. Thus, we will not discuss how the actual bytecode itself is serialized.

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

### Attributes

The above example ignores attributes, but attributes should be fairly straightforward. The general structure above assumes a bytecode like:

```
<type tag><value>
```

This will more accurately, with attributes, be:

```
<type tag><num attributes><attribute1><attribute2><value>
```

Where each `attribute1` etc. will have an `attribute tag` followed by an optional value. This optional value will be an index into the constant map.

We may also choose to serialize attributes themselves in the constant map as well! They would just have their own type tag, an attribute tag, followed by an optional constant index. In its current form this would not buy us anything but generality, which may be worth it in and of itself.

### Arbitrary expressions

The attributes have one caveat: values can just be expressions. Because of this, we also need some way to point into the bytecode and execute expressions, from the constant map. This means that the bytecode cannot simply be a list of functions, it also must contain other groups of instructions like attribute expressions. Then attributes would have an optional pointer into the bytecode's instructions.

Thankfully, the instructions from the attribute will always live in the same place as the attribute itself. This means that we do not have to worry about file boundaries.

Note that this same problem does not apply to functions. A function will have a lookup table, so we can always find it that way (more on that later). Theoretically a function could be stored in the constant map, but that would require a linker pass in order to find the correct offsets for each function. That is likely not worth it.

### `redef`

The constant map *can* be changed by other scripts! Thus, we have to eagerly load `&redef` constants and `option`s (meaning we put them into memory with their maleable value). This will require a scan of the constant map on load for any values marked `&redef`.

The downside here is that we cannot guarantee constants in memory are constant, since we have to be able to change them with `redef` or the configuration framework. It would be a bug to let the user reassign a constant value, so it will be consistently important to only allow modifying these values in very particular scenarios.

In order to get the correct value, the VM must be able to find a loaded constant if there is one, otherwise load it. For `&redef` and `option`, since the value is eagerly loaded, it will always use the pre-loaded value.

## Functions

Functions are just special globals. Say we want to load the function `test()` from Zeekscript, like:

```
local x = test();
```

The constant pool would have an entry for the function, say at index 0. It simply says it is a function with the function's name `"test"`:

```
CONSTANTS:
  #0: String("test")
```

The VM would use an instruction to load this from memory. This would be through a map lookup of global name to memory address. On load, we find the memory addresses from each function and store them in this map.

This means that functions are just any other global, just with a function type. This same logic applies to anything from another script.

> [!NOTE]
> Since we look it up in a hash map, it could be slow. Once we run it once, we can cache where it lives in memory. This is called [inline caching](https://en.wikipedia.org/wiki/Inline_caching). Modify the instruction itself to use the known location of the function.
>
> Or, since these are constants, we can resolve before ever running the program. That would be like a linker pass.

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
  name: Option<String>, // Function name, if any
  frame_slot_count: u32, // Potentially needed
  instrs: Vec<Instruction>, // This function's body
  debug_info: Option<DebugInfo>, // More on this later!
}

struct Bytecode {
  // Functions are code blocks, but so are top-level statements or attribute
  // expressions!
  blocks: Vec<CodeBlock>,
  consts: Vec<Constant>, // We can de-dup constants used in multiple functions!
  loads: Vec<ConstIdx>, // Each load is just a constant string
  exports: Vec<Export>,
}
```

Then, the constants are all defined entirely within the file and read into memory as `Constant`s. These get converted into `ZVal` with some resolution step (or, we convert them lazily and cache the result, minus `redef`able ones!). Instructions will all point to frame slots (which get populated elsewhere) or constants, so everything is self-contained.
