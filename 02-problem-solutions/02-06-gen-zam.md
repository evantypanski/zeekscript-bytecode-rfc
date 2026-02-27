# Problem 6: Gen ZAM

ZAM uses a tool called `gen-zam` in order to generate opcodes. It also has macro syntax etc. But, this is just hard to read.

For reference, here is a binary operator:

```
binary-expr-op Sub
op-type I U D T
vector
eval $1 - $2
#
eval-type T	auto v = $1->Clone();
		auto s = v.release()->AsTableVal();
		$2->RemoveFrom(s);
		$$ = s;
```

It's its own language! It has its own syntax, letters which are hard to know what they mean without prior knowledge with the tool, and absolutely no ability to have nice modern development tools (like a language server).

## Proposal: Use macros

Rust has a nice macro language. We could make a Rust macro which takes in types, arguments, `precheck`, etc. almost exactly the same. In turn, it will make an enum with all of its arguments, along with implementing some `Op` trait with a `eval` method (or similar).

This would all be in one place, though. Since the enum definition would happen all in one block, that single file may get huge. We could fix this by having a pseudo-DSL using... more macros :)

Rust gives *extremely* powerful [procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html). This would allow us to iterate over a token stream (just like `gen-zam` does) in order to create the Rust code within the file itself. We make a *derive macro* with [`proc_macro_derive`](https://doc.rust-lang.org/reference/procedural-macros.html#the-proc_macro_derive-attribute) and apply it to an enum type. Each element within the enum can use some attribute in order to define op-types, eval blocks, whether it has side effects, etc. The result gets textually written immediately after.

If the benefit is not immediately obvious, take this in `indexing.op`:

```
op IndexVecBoolSelect
classes VVV VCV
op-types V V V
set-type $$
eval	if ( $1->Size() != $2->Size() )
		ERROR("size mismatch, boolean index and vector");
	else
		{
		auto vt = cast_intrusive<VectorType>(Z_TYPE);
		auto v2 = $1;
		auto v3 = $2;
		auto v = vector_bool_select(std::move(vt), v2, v3);
		Unref($$);
		$$ = v.release();
		}
```

Then take how it could be generated with proc macros:

```
#[derive(IndexingOp)]
pub enum IndexOp {
  #[op(classes="VVV VCV", op-types="V V V" eval = "index_vec_bool_select")]
  IndexVecBoolSelect
}

fn index_vec_bool_select(ctx: &Context, op1: Val, op2: Val) -> Result<Val, String> {
  ...
}
```

The `eval` block just points to a function, and that function is *pure Rust*. These will still be different files. These will still be maintained in almost exactly the same way. We can even change the `classes` and `op-types` etc. to be more type-safe rather than parsing strings. The point is that this is *Rust* code. It is not some unhighlighted DSL with magic words to get it to do something in particular.

## Alternative: LLVM Tablegen

We could also just use an existing tool here. [LLVM tablegen](https://llvm.org/docs/TableGen/index.html) can be good for this purpose. But, we still hit the issue of essentially having a DSL without good LSP support. We still have giant blocks of `.td` files in almost the same way, but without the specificity that `gen-zam` gives. It's possible, but it's probably not the right fit, in my opinion.

Though, I have not tried it for this use case.
