# Problem 9: Events

Events are currently handled as an event queue within C++. Various analyzers shove events into this queue, then the queue gets periodically drained. This is the case even for "meta" events: `zeek_init` is just an event that gets added to the queue, then later drained. It's just that the drain is triggered within setup, and has some special properties (ie with initialization errors).

This presents two problems:

1) Events are script-level constructs. This means that an event handler lives in the script layer, and its arguments are script values.
2) The core must retrieve the events it wants to raise somehow.

## Where is the event queue?

To solve 1, we simply move the event manager to the VM. The core then simply asks the VM to enqueue an event (exactly as it does now) and drain the event queue (again, exactly as it does now). In practice, all this means is that the new VM holds the keys, and the core can interact in almost exactly the same way.

Of course, the values that get enqueued as arguments should now be created using the new value API, thus creating `ZVal`, so that the event can be entirely handled within Rust.

So the solution to 1 is simple. Move the event queue to the script VM.

## How to get events?

The second problem has a simple solution. The core of it is that the C++ core needs to understand what event to enqueue. The Rust VM is the one that knows where events are. So each time an event is enqueued, we can just ask the Rust VM to look up an event by name, provide an optional result, then enqueue it if it is available. This could just be one enqueue call.

Zeek's core currently has `.bif` files (like `event.bif`) which produce the event definitions. These definitions are used throughout the core. In order to make minimal changes, this should be targetted. Then we keep exactly the same `.bif` files and event registration. For context, the generated code for an event defined in the `event.bif` file looks like:

```
	::zeek_init = zeek::event_registry->Register("zeek_init");
```

We can simply keep this, then hijack `event_registry` just as we proposed for the event queue. Eventing is a VM concern.

Note that the event pointer will not apply. Here is the definition for `zeek_init`:

```
zeek::EventHandlerPtr zeek_init;
```

This means we must change `EventHandler`. After this, it will be an opaque handle into the VM, which it gets from the `Register` call. If a fallback is necessary, it can turn the event handler into either a Rust VM managed handler or a legacy handler, then choose based on that.

> [!NOTE]
> While eventing is a VM concern (in Rust), *registering* events is a core concern. The VM does not (and will not) have a static list of events. The core may, but doesn't have to (since it can request based on event names from the VM). Anything related to events will be a VM concern except registering them. This extends to event groups, disabling events, marking them as used, and more.
