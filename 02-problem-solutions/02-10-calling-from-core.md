# Problem 10: Calling script functions from the core

Currently, there is no mechanism in the bytecode VM to call a script function from Zeek's C++ core. This is definitely important for hooks; I [just added](https://github.com/zeek/zeek/pull/5223) a hook call from C++ core recently:

```
static const auto timing_out_hook = zeek::id::find_func("connection_timing_out");
if ( ! timing_out_hook || timing_out_hook->Invoke(GetVal())->IsOne() ) {
```

But, some script functions are called from the C++ core:

```
Discarder::Discarder() {
    check_ip = id::find_func("discarder_check_ip");
    check_tcp = id::find_func("discarder_check_tcp");
    check_udp = id::find_func("discarder_check_udp");
    check_icmp = id::find_func("discarder_check_icmp");

    discarder_maxlen = static_cast<int>(id::find_val("discarder_maxlen")->AsCount());
}

// ...

        try {
            discard_packet = check_ip->Invoke(&args)->AsBool();
        }

        catch ( InterpreterException& e ) {
            discard_packet = false;
        }
```

I propose that we have a multi-step process:

1) We allow calling hooks and functions from Zeek's core, just as today. However, we split it up, so you must use `bool vm_call_hook(ZVal hook_ptr, ...)` and `ZVal vm_call_function(ZVal fn_ptr, ...)` (or something like that). The hook returns whether or not it broke from the hook. The function returns `ZVal`.
2) We deprecate calling *functions* within Zeek. I see no good reason for this functionality.
3) We continue forward with hooks wherever needed, but the core should never need to rely on scripts to yield values.

Of course, it's not immediately clear if the core *absolutely* must suspend its execution to invoke scripts without breaking people's code. This needs testing. But, it seems like a failure of separation of concerns that the core can ask script *functions* to provide a value. It should only be able to ask the script layer if it wants to do "X predefined action" not "please provide the result of this call."
