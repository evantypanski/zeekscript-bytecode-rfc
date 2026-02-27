# Background

This is not part of the RFC. It is just some background on compiler and Zeek topics before getting started.

# Interpreters

We will discuss two kinds of interpreters: *tree-walking* and *bytecode*.

## Tree-walking interpreters

A *tree-walking* interpreter constructs an abstract syntax tree (AST) from the program, then it walks this tree in order to execute. For example, given some program like:

```
function f(a: count): count {
  local x = a + 5;
  return x;
}
```

A corresponding AST might look like:

```
- FunctionDeclaration f
  - Body
    - LocalVariable x
      - BinaryOp Add
        - Variable a
        - CountLiteral 5
    - Return
      - Variable x
```

When a tree-walking interpreter executes this function, it essentially just calls `func.Exec()`. Then, that function will call `body.Exec()`. To execute the body, it will simply walk through all of its statements and call them in the same way, like `stmt.Exec()`. An expression (like `BinaryOp` would be) in Zeek gets called with `Eval()` in order to produce a `ValPtr`. So `BinaryOp` might have an `Eval()` function that just adds each operand, like `return lhs.Eval() + rhs.Eval()`.

This is a relatively simple implementation, but there are a couple fundamental flaws with tree-walking interpreters.

First, a tree-walking interpreter is always slow. Each time you call the `Exec` or `Eval` function, those are `virtual` functions. Each call has double-indirection, once to dereference the object, once to dereference the virtual method table, *then* you can dereference the function pointer. This is slow and horrible on the cache in regular instances, but for interpreters, it ends up being the bulk of the work.

Second, it couples a program's structure with its execution. Since each node determines how it is executed, it is harder to create optimizations or analyses that have greater context about it. That's more of an architectural problem, but it contributes to huge unmaintainable code.

Third, it's very inefficient on memory use. Even a tiny function (like above) ends up with *eight* different objects. Each object is pretty heavy. Each pointer is (likely) 64 bits of information. Note that there's no guarantee that these pointers are close to each other either, so caches do *terribly*. We would want something with more spatial locality.

For those reasons, tree-walking interpreters are almost never used as production-grade interpreters (at least anymore).

## Bytecode interpreters

Bytecodes work by first *compiling* the source code into some format, then later *interpreting* this other format. This format is a linear list of instructions. Generally, a bytecode will be a variable sized instruction set. That means how many bytes each instruction is depends on what instruction it is. A `OP_RETURN` may be just one byte (the opcode for return), while `OP_BINARY` may need that opcode *plus* what operator it is (plus, minus, etc.) *plus* its operands.

The thing that executes the bytecode is a *virtual machine* or VM.

Many virtual machines you see will be *stack-based*. That means that various opcodes *produce* a value (ie shove that value onto the stack---the `OP_CONSTANT` opcode would simply put a constant on the stack). Then, some opcodes *retrieve* values from the stack (like `OP_BINARY` would just take two values off the stack). What a given opcode does with the stack is its *stack effect*. Now forget this because it doesn't matter for Zeek.

Zeek already has a *register-based* system within its frames. That means that when you're accessing a variable, you know at compile time the variable `x` is at frame slot `2`. Thus, instead of `OP_BINARY` popping two values off the stack, it will be told "add frame slot 1 and 2."

There are times when an opcode will want different kinds of values. For example, it might want a *constant*, not a frame slot. For that, systems make naming conventions, like `OP_BINARY_VVV` would mean the result of adding two frame slots (last 2 `VV`) is stored in a frame slot (first `V`), or `OP_BINARY_VVC` would mean the result of adding a frame slot and a constant (last `VC`) is stored in a frame slot (first `V`). This is roughly how ZAM currently encodes information about what an opcode expects.

Note that picking registers is an interesting problem, especially for optimizations. You want to create as few "virtual registers" as you can (frame slots), so you may want to reuse some.

## Values

So, what does a frame slot *actually* hold? That's a value. A significant portion of this RFC is dedicated to this very concept, since it's where issues start to arise for Zeek. A bytecode should have a space-efficient way to store values.

Current Zeek values can show how this goes wrong. Say an expression returns a simple count. If the interpreter wants to access this value, it gets a *pointer* (`ValPtr`). Then we have to dereference it whenever we want to access its value. But it's a pointer! It could just hold the count itself.

ZAM changes this a bit, it's a union!

```
union ZVal {
    // ...

    // Used for bool, int, enum.
    zeek_int_t int_val;

    // Used for count and port.
    zeek_uint_t uint_val;

    // Used for double, time, interval.
    double double_val;

    // ...
};
```

Looks good! Except after this, we have:

```
    StringVal* string_val;
    AddrVal* addr_val;
    SubNetVal* subnet_val;
    File* file_val;
    Func* func_val;
    ListVal* list_val;
    OpaqueVal* opaque_val;
    PatternVal* re_val;
    TableVal* table_val;
    RecordVal* record_val;
    VectorVal* vector_val;
    TypeVal* type_val;

    // Used for "any" values.
    Val* any_val;
```

So ZAM just falls back to `Val*`. This is necessary for any language: anything heap-allocated will be represented by a pointer. This also makes it into the *bytecode* though. This is in the `ZInst`:

```
    ZVal c; // constant associated with instruction, if any
```

For reference, you can find a simple bytecode value [in Lox](https://github.com/britannio/lox/blob/2e66d24bdaa274e0b27f86eef01c11674bda3e00/clox/value.h#L66-L75), used for the book [crafting interpterers](https://craftinginterpreters.com/):

```
// [type, value]
typedef struct {
    ValueType type;
    // The struct makes space for the largest of these types.
    union {
        bool boolean;
        double number;
        Obj *obj;
    } as;
} Value;
```

In the end, the values are pretty standard. You have a type that can store certain values that fit in a pointer width (like double, int, count, etc.) directly, then anything else on a "heap." This is just about how any value model will work.

### Constants

A lot of this document will discuss *constants* for values. Say we have a table constant. How does this get represented?

The general way to solve it is with a constant pool (well, strings can be different, we will ignore that). If a value can't easily be stored within the code itself (like an integer, which could just be immediately after the opcode), then we store it in a *constant pool*. That's a scary name, really it's just a big array of values. If we have a table constant, we can reference it by "constant index 159." Then, when we hit that line in the code, we just look at that index. Easy.

This constant pool is *known at compile time*. When a bytecode gets serialized, it has a static number of values in the constant pool. Each time you load it, you load it with the same values at the same indices. That means that whenever the program is compiled, it is *always* the same index into the constant pool, so it can be placed in the code when referenced.

ZAM does not do this. Its constants come directly from the Zeek script it was compiled from, so there was no serialization of values. This means that you cannot output a ZAM file then later load it from disk. We saw that above, where `ZInst` stores `ZVal c`. The instruction itself stores a pointer.

### `threading::Value`

Zeek's refcounting is not an atomic operation, so if data is shared between threads, Zeek uses a different value type. This is `threading::Value`. This type is essentially plain old data, even if it's "complex." For example, this is its `string` value:

```
        struct {
            char* data;
            int length;
        } string_val;
```

Notably not a `StringValPtr`.

# Compilation

A compiler works in a few phases:

1) Lexing, turning the file into a stream of tokens.
2) Parsing, turning the stream of tokens into a structured tree (the abstract syntax tree, or AST).
3) Semantic analysis, analyzing the program for correctness (eg type checking).
4) Generate intermediate code
5) Optimization

Since Zeek is a tree-walking interpreter (which means it is *not* a compiler without ZAM), it only does the first two steps (ish?). During parsing, it does certain type checking and argument checks to see if types align. This is a pretty simplified way of doing semantic analyses, since it means you can't get much context about anything before or after.

That's also why something like ZAM is useful: It introduces a *compiler* to generate the intermediate code (the bytecode), then we can get down to the optimization step. You can, of course, optimize an AST then tree-walk, but Zeek doesn't.

## Zeek's function-based approach

Zeek compiles *functions* and has no notion of a compilation unit (or an entire file that comprises a single AST piece). There is only a big list of functions, events, and other declarations, not a "compilation unit" that holds it together. This means that doing something on a per-file basis is an architectural shift.

ZAM doesn't deal with this. All it does is compile a new body and replace the body (in `ScriptOpt.cc`):

```
        new_body = ZAM.CompileBody();

        if ( reporter->Errors() > 0 )
            return;

        if ( analysis_options.dump_final_ZAM )
            ZAM.Dump();

        f->ReplaceBody(body, new_body);
        body = new_body;
```

This means ZAM still plays within the AST. That `CompileBody` function returns a `StmtPtr`. It's just inserting a new body, so it's still within the tree walking interpreter.

But, this also means that ZAM's implementation details leak into the interpreter. Here are a few examples in `Func.cc`:

```
    if ( bodies[0].stmts->Tag() == STMT_ZAM )
        captures_vec = std::make_unique<std::vector<ZVal>>();
    else {
        delete captures_frame;
        delete captures_offset_mapping;
        captures_frame = new Frame(captures->size(), this, nullptr);
        captures_offset_mapping = new OffsetMap;
    }
```

```
    if ( bodies[0].stmts->Tag() == STMT_ZAM ) {
        auto& captures = *type->GetCaptures();
        int n = f->FrameSize();

        ASSERT(captures.size() == static_cast<size_t>(n));

        auto cvec = std::make_unique<std::vector<ZVal>>();

        for ( int i = 0; i < n; ++i ) {
            auto& f_i = f->GetElement(i);
            cvec->push_back(ZVal(f_i, captures[i].Id()->GetType()));
        }

        CreateCaptures(std::move(cvec));
    }
```

These exceptions occur with captures. It's important to note that it's not simply a drop-in replacement.
