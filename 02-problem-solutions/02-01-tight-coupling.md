# Problem 1: Tight Coupling

The scripting implementation is deeply coupled with Zeek. Any analyzer written in C++ *directly creates* pointers to script values. Many plugins and core components take in script values in BiFs (built-in functions), translate them into C++ types, then pass that along to some other system.

This is a manual process: users must create values by hand, then retrieve them by hand, from the scripting engine. You may even change values from the scripting engine within the core. 

But, the core does not need to "think" in values like this. It is often better for the core to pass around less, or statically allocate simple objects. Indeed, it does, and many parts only create the Zeek script values when necessary. The goal here, then, is to make the core a *consumer* of the data created by the interpreter. The interpreter owns the data, the core simply updates (or reads) data within it.

## The Barrier

The core to this problem is how the barrier between the interpreter and core is handled. Since the interpreter owns the values, the core can no longer think in terms of `Val` objects, or intrusive pointers. Instead, there are a set of shared structs which define the memory layout. 

First, we have a tagged union. This is basically what `ZVal` is currently. For now, we will ignore the problem of interacting with Zeek values that have not yet swapped, that will come later. Also, this will likely be written in Rust, with `cbindgen` or similar to convert into a header file in C++:

```
struct ZVal {
	uint8_t type_; // The resulting type of the value, without extra info.
	uint8_t flags; // This plus the `type_` can fully describe which union field 
				   // to pick. This includes a 'legacy' bit to see if we have to 
				   // use as_ptr
	uint16_t extra; // This would be for optimizations, like a length for short 
					// strings?
	
	union {
		int64_t as_int; // zeek_int_t int_val
		uint64_t as_uint; // zeek_uint_t uint_val
		double as_double; // double double_val
		
		// ZVal then has a list of pointers that it uses. Instead, we will
		// have two options: an offset (which is the new, fast one) and a
		// raw pointer (which would be the legacy Zeek::Val)
		uint32_t as_offset;
		void* as_ptr;
	} payload;
};
```

This will be a 16-byte struct and can describe *all* Zeek values, legacy or not, within the interpreter and Zeek's core. Complex values will use `as_offset` in order to point to an `Arena`. This arena is where all values get stored, except for legacy `Val`.

```mermaid
flowchart LR
    subgraph Cpp [C++ Core]
        direction TB
        Analyzer
        LegacyVal[Legacy Val* Heap]
    end

    subgraph Bridge [FFI Bridge]
        direction TB
        ZValBuilder[ZVal Builder]
        LegacyConv[Legacy Value Converter]
    end

    subgraph Rust [Rust VM Host]
        direction TB
        Interpreter[Bytecode Interpreter]
            
        subgraph Storage [Memory Model]
            direction TB
            Scratch[Scratch Arena]
            Heap[Persistent Heap]
      end
    end

    Analyzer -- "Requests values" --> ZValBuilder
    ZValBuilder -- "Writes values" --> Scratch
    ZValBuilder -- "Commits values" --> Interpreter

    Interpreter -- "Reads/writes legacy values" --> LegacyConv
    LegacyConv -- "Fetches/modifies legacy values" --> LegacyVal
    LegacyConv -- "Convert ZVal back to Val if writing" --> ZValBuilder
    
    Interpreter -- "Read/write" --> Scratch
    Interpreter -- "Read/write" --> Heap
    
    Scratch -.->|Promote/Clone| Heap
```

Any value stored within the arena will have to be built. For this, we would create a C++ API which creates the complex values from simple structs. Some of these are easy, like a String can just take in an std::string:

```
ZVal vm_create_string(VM* vm, std::string s);
```

Others, however, will need "built." We do this with a builder:

```
RecordBuilder* vm_begin_record(VM* vm)
```

This creates a builder that we can set fields with:

```
void vm_set_record_field(RecordBuilder* builder, std::size_t idx, ZVal val);
```

> NOTE: One issue in Zeek right now is the reliance on indices (ie the std::size_t idx) when making record values. I believe we can solve this by creating a "layout" at startup. However, that can be done regardless of backend, so for this, I will continue with this structure.

Then, when we are done, we "finish" the builder, invalidating the pointer, and returning the `ZVal` for what we just created:

```
ZVal vm_end_record(RecordBuilder* builder);
```

Each of these functions are implemented in Rust. It is the interpreter's concern *how* to build the object from various ZVals, it is the core's concern to place them in the right positions.

For the string, the value can be simple: since strings are often immutable, just store the length and the raw string. This can be adapted accordingly. We may also choose to intern strings, or use symbols, or something else, but this is a simple case.

However, cases that produce a builder need to be more complex. The most pressing concern is: what if they never call `vm_end_record`? Then we keep the value around forever in the arena, so it's a memory leak. Not good.

The solution here can vary. One proposal is to just keep it around, bump the `next` of the linear memory, etc. But, at the end of the packet, run a compacting garbage collector to gather any unreferenced values. We would store the refcount somewhere in the value.

This could be made more efficient with "tenuring" ie if an object sticks around, move it to another space that is less likely to need collection.

But, we could also have a "scratch" arena (eden) and a "persistent" (survivor) arena. In order to make the scratch space last longer than for this packet, it must get promoted. Promotions are handled by the VM when necessary, for example when adding to a vector in persistent memory. Everything in the scratch arena gets deleted after each packet. Allocation is simply bumping the `next` pointer (a bump allocator).

Any persistent objects, then, are similar to managed values: they get refcounted just as before.
