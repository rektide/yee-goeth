# yee-goeth

> Graceful teardown for elements. *Here goeth nothing.*

The symmetric counterpart to [`yee-there`](https://github.com/rektide/yee-there). Where yee-there lets an element signal "I am arriving, here is my hold," yee-goeth lets an element signal "I am leaving — give me a moment to clean up."

## Why

Most interesting custom elements hold resources:

- a network connection (cast, moq, WebTransport, fetch streams)
- a decoder / encoder pipeline
- a subscription with a remote registry
- an open file, socket, or stream
- in-flight async work that shouldn't be cut mid-stream

Today, when such an element is `disconnectedCallback`'d or removed from a yee-mix's query results, the cleanup is best-effort and immediate. There's no general mechanism for *"this will take a moment; let me finish what I'm doing."* Service-worker `ExtendableEvent` exists; nothing equivalent exists for general DOM teardown.

yee-goeth fills that gap with a `YeeGoethEvent` (extends `ExtendableEvent`) dispatched on teardown. `waitUntil(promise)` extends teardown until the promise settles — flush this GOP, send this UNSUBSCRIBE, close this connection cleanly, ack this final message.

## The shape

```ts
class YeeGoethEvent extends ExtendableEvent {
  static readonly type = 'yee-goeth'
  // What triggered teardown — informs how graceful to be.
  readonly reason: 'disconnected' | 'unapplied' | 'ejected' | 'pagehide'
  constructor(init: YeeGoethEventInit) { super('yee-goeth', init) }
}

interface YeeGoethable extends EventTarget {
  // Element dispatches yee-goeth on teardown; waitUntil() extends teardown
  // until the element's resources are released.
}
```

The aggregator side — typically a yee-mix host or the page itself:

```ts
class YeeGoethGate extends EventTarget {
  // Returns a promise that resolves when all teardown holds on the
  // observed subtree have cleared. Re-arms when new yee-goeth events fire.
  settled(): Promise<void>
  // For hard-shutdown: force-settle regardless of in-flight holds.
  force(): void
}
```

## Reason taxonomy

The `reason` field lets elements pick the right level of grace:

- `disconnected` — element left the DOM. Standard teardown.
- `unapplied` — yee-mix removed the element from its target set (applier still alive).
- `ejected` — page is force-closing (pagehide, browser closed). Minimal cleanup.
- `pagehide` — same as ejected but page-initiated.

An element may use different teardown paths per reason. A moq subscription on `ejected` just drops the socket; on `disconnected` it sends UNSUBSCRIBE first.

## Where it lands in the yee family

This is the missing half of the **transactional apply/unapply** contract that cast-yee, moqy, and command-auto-yee all need:

- cast-yee on `unapplied` waits for the receiver to ack the close.
- moqy `<moq-subscribe>` on `unapplied` waits for UNSUBSCRIBE ack; on `ejected` drops the socket.
- command-auto-yee on `unapplied` waits for in-flight invocations to drain.
- yeetable on `unapplied` waits for the final payload to flush.

yee-goeth makes these uniform. yee-mix's `unapply()` becomes: dispatch `yee-goeth`, gate further target processing on `settled()`.

## Why the archaic name

*Goeth* — third-person archaic of *goes*. yee-there says "are we there yet"; yee-goeth says "and so it goes." Pair names that scan.

## Status

Baseline. README + design contract only — implementation and tickets under beads prefix `goeth`.

## Related

- [`yee-there`](https://github.com/rektide/yee-there) — symmetric counterpart (arrival).
- [`yee`](https://github.com/rektide/yee) — primary consumer; gates unapply on yee-goeth.
- [`cast-yee`](https://github.com/rektide/cast-yee), [`moqy`](https://github.com/rektide/moqy) — first real consumers with stateful teardown.

## License

MIT
