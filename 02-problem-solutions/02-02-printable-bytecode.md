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
  // Function pointers and types would be constants too!!
}
```

When it is possible to store something in a simple form, the constant pool will inline. For example, constant entry 0 may just be an integer with value 2. Simple.

Complex types (like vector above) will contain a vector of indices. Since it's constant, we can define a layout, like the number of elements followed by the index of each other value included in the vector:

```
6123456
```

This would be a vector of size `6` and whose elements are the constants at index `1`, `2`, ..., `6`. This won't be ASCII, it'll just be bytes in the constant pool.

> [!NOTE]
> We can define the constant pool layout however we want. This is just an example.

Then, we just have instructions to access a particular constant. We can convert this constant into `ZVal` easily.

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

## Exports

A script may export many things. We need a separate section that describes which identifiers get exported and what their values are.

First, the value is defined in the constant pool. Then, we add an entry to the exports list, which maps a global name to the index into the constant pool. Then, the VM will register all of the exported names into persistent memory.

# Proposal

The bytecode itself would be extremely simple. Imagine the Rust struct as:

```
struct Bytecode {
  consts: Vec<Constant>,
  instrs: Vec<Instruction>,
  exports: Vec<(String, ConstIdx)>,
  // Importantly, we will probably need a mapping of instruction index to
  // "span" for debug info, which is printed if given a debug option.
  spans: Vec<(u32, u64)>,
}
```

Then, the constants are all defined entirely within the file and read into memory as `Constant`s. Then, these get converted into `ZVal` with some resolution step. Instructions will all point to frame slots (which get populated elsewhere) or constants, so everything is self-contained.
