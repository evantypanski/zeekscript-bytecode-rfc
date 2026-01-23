# Problem 3: Debugging

When debugging is not a concern from the start, the solutions end up tacked-on. Here, we will discuss two different considerations for debugging: the *user* debugging their Zeek scripts, and the *developer* debugging the output bytecode and the interpreter behavior. These will use similar mechanisms, so they are combined here.

## Debugging Zeek Scripts

Recently, the Zeek team [(re)discovered a script debugger](https://github.com/zeek/zeek/pull/4837). While this is almost certainly not used significantly, the functionality is notably "nifty." With a new bytecode, we should have this logic from day one, then advertise it better so that users can enjoy it.

First, the bytecode will already have source locations in the printable form (I will assume it is present, the debug information will be optional). We will now harden this proposal.

The bytecode structure included:

```
struct Bytecode {
  // ...
  debug_info: Option<DebugInfo>
}
```

We will create a new `Span` struct to use within `DebugInfo`:

```
struct Span {
  start_offset: u32,
  end_offset: u32,
}
```

The start and end offsets can be used in order to find line and column information. They are simply indexes into a Zeek script file. For this, a mechanism to cache line numbers is necessary (see how Clang does it, for example. I [wrote about that](https://etyp.dev/posts/errors-pls/) once, the algorithm is relatively simple).

Then, a separate `DebugInfo` struct will correlate instructions to their spans:

```
struct DebugInfo {
  // For now, one Bytecode is one DebugInfo. This may change!
  file: String,
  // This is 1:1 mapped with Bytecode::instrs
  spans: Vec<Span>,
  // Keep some information on locals, so that we can map a name to its frame
  // slot. This will store a name, slot, and span, most likely.
  locals: Vec<LocalVarInfo>,
}
```

This is all that is necessary for most cases! Each instruction points back to the start and end indices of what it got compiled from, so we have everything that we need in order to provide source locations for runtime errors, compile errors, and assertions.

> [!NOTE]
> This example uses a one-file-per-Bytecode approach. However, this might not be the best way. In the future, we could imagine a world where a module is compiled into its own Bytecode, with everything included. Then, we would need a map from an index to string filenames in `DebugInfo` (ie `Vec<FileIdx>`). Then a span will hold a `FileIdx` type that will use that. This likely isn't necessary to start.

Then this can print extra information as comments when pretty printing the bytecode:

```
CONSTANTS:
  #0: String("test") ; local x <test.zeek:2:3-2:15>

test.zeek:1:0 (function my_func)
  0000: LOAD_CONST r0, #0 ; local x = <CONSTANT 0>
```

Though, the bytecode will be unlikely to be printed by many people outside of the Zeek team.

### Debugger

This section will only discuss the infrastructure that will be necessary for a debugger to operate. While the debugging itself is important, it is not vital to plan from the beginning. The goal here is to create a generic mechanim that all sorts of debugging can utilize.

The main structure will be within the virtual machine. We will structure this as "hooks" (separate from Zeek hooks) that will execute *before* and *after* each instruction. This could be:

```
struct VMState {
  frames: Vec<Frame>,
  ip: usize,
  // etc.
}

struct VM {
  // ...
  state: VMState,
  debug_hooks: Vec<Box<dyn DebugHook>>,
  debug: bool,
}

// Optionally split this into one that takes the immutable VMState for safety
trait DebugHook {
  fn before_instr(&mut self, vm_state: &mut VMState, ip: InstrIdx);
  fn after_instr(&mut self, vm_state: &mut VMState, ip: InstrIdx);
}
```

Then you can implement any number of debug hooks. Some might just be gathering metrics (ie opcodes, which indexes are used most, etc.). Some might implement debugger functionality.

This is very expensive, so the `debug` flag will prevent anything from getting added to `debug_hooks`, and prevent checking to run them.

For example, you might have a hook that keeps a set of breakpoints:

```
struct DebuggerHook {
  breakpoints: HashSet<InstrIdx>,
}
```

Where we add breakpoints from some user commands (like `b test.zeek:5` would find the instruction for `test.zeek:5` and add it to the `breakpoints` set). Then, we just pause for the user in `before_instr`:

```
impl DebugHook for DebuggerHook {
  fn before_instr(&mut self, vm_state: &mut VMState, ip: InstrIdx) {
    if self.breakpoints.contains(&ip) {
      vm.break_debug(ip);
    }
  }

  fn after_instr(&mut self, vm: &mut VMState, ip: InstrIdx) {}
}
```

From this point, the VM should contain all of the information necessary. Notably the frame is important, along with the call stack. Those will be exposed as methods in the VM, then hooks may call those as needed.

## Debugging the Interpreter

We should make registering a hook very easy, and good logging for potential debugging here.

Notably, one issue will be lowering Zeekscript into the bytecode. For this, we will be in C++ (for easy access to the AST). Unfortunately, this will have to remain as-is, but some mechanism like `dump` methods and good logging of various stages (similar to Spicy) would be very helpful.

For the bytecode, it would be helpful to:

1) Get the line that a given instruction compiled to
2) Track opcodes as they execute, with extra info like function locations

The first will just use the `DebugInfo` to look up the instruction index. If available, we should also allow dumping the code from the source file. This would make it so you don't have to leave the debugger to find the exact file used.

The second will just be a debug hook: before anything executes, just pretty-print the value. We can optionally include the `Frame`, but that's a lot of information.

We also separated the VM state out from the VM so that we can modify the frames themselves. This is powerful! We can print the bytecode to get an instruction index, then modify it to a particular value in order to test something. These sorts of one-off debug hooks are vital for testing and reproducing.

# Proposal

Almost all debugging should be done with a vector of `DebugHook`s. These are conditionally enabled so that it doesn't take runtime. They may modify the values.
