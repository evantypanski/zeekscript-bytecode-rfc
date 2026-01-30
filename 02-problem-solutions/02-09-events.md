# Problem 9: Events

Events are a bit tricky. Ideally, they are the first part to switch to `ZVal`-focused handling. But, we have to be careful about events which need extra consideration, like `schedule`. Furthermore, we may not guarantee that each invocation of `Drain` completely empties the queue, as it only runs two rounds!

Events are also a scripting concern, so the interpreter will handle them. The interpreter will know where each event is and hold the event queue.

## The queue drain problem

While previously, the scratch space was said to be destroyed "after each packet," this logic needs revisitted. The first point is that we may need a `ZVal` to live longer if its event has not been handled yet. For that, we will need a special handler after draining the event queue.

For this, we will do one final round: Gather up all `ZVal`s that are used in pending events (without dispatching!) and promote them to persistent space.

## Scheduled events

Some events can be scheduled:

```
schedule 30sec { myevent(x, y, z) };
```

In this case, we must keep `x`, `y`, and `z` in persistent space. We can determine this from the runtime: if there is code scheduled (like so) or async code, then promote any values to persistent space.

# Proposal

The proposal here is simple: after the event queue is drained, it must promote any needed events in scratch space to persistent space. Then, scheduled events have all values promoted immediately.

The event queue will move to the interpreter. Event handlers are simply pointers to values. This will be a `ZVal`. These `ZVal`, as discussed earlier, may point back into logacy script pointers, which invoke "legacy" events.
