# Realtime — actions, channels & timers (0.20.0)

Until 0.20.0 a plugin had two speeds: a frontend write to the doc store, and a backend `onSchedule` tick
reading it back, floored at 10 s. Fine for a leaderboard, useless for anything that takes turns — a move
checked the instant it's made, every participant seeing it within a second, a different view per user
(your hand, your turn), a clock that runs out in seconds. **Actions** cover the request path, **channels**
the push path, and a one-shot **timer** covers the clock. ARCHITECTURE §7.9 is the full spec; this file is
the plugin-author's view of it.

**This is not "plugins author routes."** The host owns the URL, the method, authentication, CSRF, the body
cap, the rate limits and the error shape, and checks access before any plugin code runs — a plugin
registers a *function*, the host hands it a caller it already resolved. Same shape as `SearchProvider`
refining what the host already filtered, just at request time instead of search time.

Both manifest blocks (`actions`, `channels`) are documented in `manifest.md`; this file covers the
backend/frontend APIs and the test kit.

## Actions — request-time calls into your backend

```java
ctx.actions().handle("move", call -> {
    synchronized (lock) {                                   // handlers, timers, callbacks all run concurrently
        Game game = load(ctx.store(), call.scope());        // the store is the truth; memory is a cache
        if (!game.turn().equals(call.caller().userId())) {
            throw new ActionRejectedException("not-your-turn");         // → 422 with `code`
        }
        game.apply(call.payload(Move.class));
        ctx.store().put(call.scope(), "game:state", game);  // under data.backendPrivate: no client reads it raw
        ctx.channels().channel(call.scope(), "game").publish("moved", game.publicView());
        return game.publicView();                           // → JSON body; null → 204
    }
});
```

```ts
try {
  const board = await ctx.actions!.call<Board>('move', { cell: 3 });
} catch (e) {
  if (isActionRejected(e)) notice(t(`error.${e.problem.code}`));        // the plugin's own code, 422
  else if (isRateLimited(e)) retryIn(e.problem.retryAfterMs ?? 1000);   // 429, or a busy plugin's 503
  else throw e;                                                         // a bug or an outage
}
```

**Register every name in `register(ctx)`.** `Actions.handle(name, handler)` throws
`IllegalArgumentException` for a name the manifest doesn't declare — a typo caught at startup, not at the
first production call.

### What `ActionCall` gives you

```java
Caller   caller();       // host-resolved — never read from the payload
Scope    scope();        // host-checked visible to this caller; never USER
JsonNode payload();      // client input — validate it
<T> T    payload(Class<T> type);    // escaping this on a bad shape is a 400 bad-request
Instant  receivedAt();   // stamped before queueing — what orders a buzzer, not when the handler ran
DocStore store();        // ctx.store(), except USER "me" resolves to the caller
```

- **`caller()` is never anonymous** — the host refuses an anonymous call (401) before any handler runs.
- **`store()`'s `USER` exception is the one place a backend reaches a user partition** (§7.4's rule bends
  exactly here) — but that partition stays the *user's*: they can overwrite it over HTTP, and
  `backendOwned`/`keyFloors`/`backendPrivate` are all ignored there. **Never authoritative.** An accepted
  move, a hidden hand — anything the backend vouches for — goes in a **shared** scope under
  `data.backendPrivate`, not in the caller's own partition.
- `Caller(userId, role)` — to show it as a person, resolve `userId` through `ctx.users()`/`identity`
  (§8.8), same as every other id this contract hands you. Never persist a name.

### The check order, and what each refusal looks like

The host checks, in order, before any handler code runs:

| # | Check | Refusal |
|---|---|---|
| 1 | plugin declares `actions` | 404 `not-found` |
| 2 | the block declares `name` | 404 `unknown-action` |
| 3 | signed in | 401 `unauthorized` |
| 4 | the name's `callableBy` | 403 `forbidden` |
| 5 | body ≤ 16 KiB (`Actions.MAX_BODY_BYTES`) — checked **before** the scope, which travels inside the body | 413 `payload-too-large` |
| 6 | the scope exists and this caller may see it | 404 `not-found` — absence, never "you may not" |
| 7 | per-user / per-plugin rate budget | 429 `too-many-requests`, `Retry-After` + `retryAfterMs` |
| 8 | a handler is registered for the name | 503 `action-unavailable` — the author's wiring bug |

Then the handler runs, and its outcome is the answer:

- its return value → JSON body; `null` → 204
- `ActionRejectedException(code[, detail])` → **422** `action-rejected`, with `code` as a top-level field —
  the plugin's UI reads `code` to explain itself in the reader's language; `detail` is for logs, never shown
- a payload `payload(Class)` cannot convert, escaping the handler → 400 `bad-request`
- anything else thrown → 500, logged to the plugin's log
- the plugin's queue is full → 503 `action-busy` (`retryAfterMs`); the handler overran its deadline → 504
  `action-timeout` (it **keeps running** — Java can't stop it — but the caller is already answered)

Every refusal type is `https://mosaicast.dev/problems/<slug>`, exposed as `PROBLEM_TYPES` in TS.
`isActionRejected(e)` narrows to the 422; `isRateLimited(e)` narrows to the 429-or-busy-503 pair. Both size
caps (the action body, a channel event's data below) are measured on **UTF-8 JSON**, not character count.

### Concurrency — the plugin's problem, not the host's

Handlers run on a bounded executor of the plugin's own, never the request thread, **concurrently with each
other, with `runAfter` timers and with channel join/leave callbacks**. The host promises no ordering beyond
`receivedAt()` (stamped on arrival, before queueing — that's what decides who buzzed first). **The plugin
synchronizes its own state.** In-memory state between calls is a legitimate cache; anything that must
survive a restart belongs in the store, under `data.backendPrivate` if it's not meant to be read raw.

## Channels — live push events

```java
ctx.channels().authorize((caller, ref) -> isPlayer(ref, caller.userId()));   // none registered → every subscribe refused
ctx.channels().onLeave((ref, caller) -> games.markAway(ref, caller.userId()));

ctx.channels().channel(scope, "game:42").publish("moved", view);              // everyone subscribed, right now
ctx.channels().channel(Scope.site(), "inbox").publishTo(bob, "invited", i);   // one user, on a shared channel
```

```ts
let state: Game | null = null;
const load = async () => { state = await ctx.actions!.call<Game>('state'); render(state); };
const sub = ctx.channels!.subscribe('game:42', (e) => {
  if (e.event === 'moved' && state && (e.data as Game).version > state.version) render((state = e.data as Game));
}, { onReset: load, onStatus: (s) => showStatus(s) });
await load();
// on unmount: sub.close();
```

**The recipe is always the same: subscribe first, then fetch state through an action, then apply events,
discarding anything your own version numbers say you've already seen.** `onReset` means exactly "fetch
again" — a long disconnect, a host restart, anything the shell can't replay from its short per-subscription
buffer. A late subscriber receives **nothing from before** — events are never persisted, a publish with
nobody subscribed is a no-op — which is what makes a channel a *notification of change*, not a source of
truth, and what makes a typing indicator safe to push through it.

- **`Channel.publish(event, data)`** reaches every current subscriber; **`publishTo(userId, event, data)`**
  reaches one user's subscriptions on a *shared* channel only — a no-op if they're not subscribed (nothing
  reaches a browser that didn't ask and wasn't admitted). `subscribers()` is the distinct user set right
  now, one entry per user regardless of tabs.
- **`ChannelRef(scope, name)`** is the channel's identity, scoped per plugin — two plugins' `inbox` on the
  same scope are different channels. `scope` is never `USER` (a channel is for several people to share).
- **Without an `authorize` callback registered, every subscribe is refused.** The host's own checks
  (signed in, `subscribableBy`, scope visibility, the `perUserSubscriptions` cap) run first; the authorizer
  is where *membership* lives — the host has no model of who's playing in your game.
- **`onJoin`/`onLeave` fire per *user*, not per tab** — join on a user's first subscription to a channel,
  leave on their last one ending, for whatever reason (closed by the browser, refused, kicked, or ended
  because access changed).
- **`Channel.kick(userId)`** ends every subscription that user holds on this channel; their browser sees
  `refused`, terminally — no auto-reconnect. Whether they can subscribe again is the authorizer's call next
  time.

### Lifecycle: `closed` vs `refused`, and which is terminal

`ChannelStatus` = `'connecting' | 'open' | 'refused' | 'closed'`.

- **`refused` is terminal.** The floor, the authorizer, the subscription cap, a `kick`, or being signed out
  (then without even a request). The shell does not retry it.
- **`closed` is also terminal**, but for a different reason: sign-out, account erasure, a role change, a
  site-access change, or the plugin being disabled — access *changed*, it wasn't denied to begin with.
- **A 429 is not terminal.** The subscription stays `connecting` and the shell retries after
  `retryAfterMs`.
- A subscription is **bound to a browser session** — a personal access token can't open one, because
  nothing would end it on revoke.

Channel and action-event names share the same grammar (`^[a-z0-9][a-z0-9-]{0,63}$`); a **channel's own
name** is looser (`^[A-Za-z0-9:_-]{1,128}$`) since plugins build structured ids (`game:42`) out of it.
`Channel.MAX_EVENT_BYTES` is 8 KiB of serialized JSON; `publish` throws `IllegalArgumentException` over that
or on a bad event name — a broken event is the backend's bug, caught in its own tests, never a browser's
problem to diagnose.

**How events travel is the host's business, not the contract's.** v1 is Server-Sent Events, one stream per
tab multiplexing every plugin's channels; nothing in the SDK names a transport, and a future WebSocket
transport could replace it without a `platformApi` bump.

## Timers — `ctx.runAfter`

```java
Scheduled timer = ctx.runAfter(Duration.ofSeconds(30), () -> timeOut(ctx, scope));
// later, if the turn ends early:
timer.cancel();   // idempotent; false if it already ran or was already cancelled
```

One-shot, **in memory only**, no manifest declaration and no floor — `onSchedule`'s 10 s floor exists to
stop a *periodic* misconfiguration from becoming a busy loop, and a one-shot timer can't become one. A zero
delay runs as soon as possible; a negative or `null` delay is `IllegalArgumentException`, never clamped.

**A restart drops every pending timer.** Persist the deadline itself (a turn's end time, in the store) and
re-arm the timer from it in `register(ctx)` — the same "memory is a cache, the store is the truth" rule
actions follow. The host caps pending timers per plugin (`IllegalStateException` past it), and stopping the
plugin cancels all of them. Timers run on the same executor as action handlers and channel callbacks —
same concurrency rules apply.

## Testing

### Java — `FakePluginContext.withActions(...)` / `.withChannels()`

```java
FakePluginContext ctx = new FakePluginContext().withActions("join", "move").withChannels();
plugin.register(ctx);

Caller alice = Callers.fan();
ctx.actions().call(alice, "join", Scope.episode("s2e04"), Map.of());

ActionRejectedException e = assertThrows(ActionRejectedException.class,
        () -> ctx.actions().call(alice, "move", Scope.episode("s2e04"), Map.of("cell", 3)));
assertEquals("not-your-turn", e.code());

FakeSubscription sub = ctx.channels().subscribe(alice, new ChannelRef(Scope.episode("s2e04"), "game"));
assertThrows(SubscriptionRefusedException.class,
        () -> ctx.channels().subscribe(Callers.fan(), new ChannelRef(Scope.episode("s2e04"), "game")));
```

- `Callers.fan()` / `.podcaster()` / `.admin()` mint a fresh user each call (pass a `UUID` to be the same
  person twice).
- `FakeActions` is **synchronous by design**: a call runs its handler on the test thread and returns the
  answer directly — no executor, no deadline, no rate limit, since those are the host's to test, not the
  plugin's. Where the host would refuse *before* a handler runs (undeclared name, below floor, no
  handler registered, body too large), it throws `IllegalStateException` naming the status it would have
  answered. `ActionRejectedException` and any other handler throw propagate as thrown — a real 400/500 case
  looks the same in a test as it would need to be handled in production.
- `withCallableBy(Role)` / `withNameFloor(name, Role)` stand in for the manifest's `callableBy` fields, and
  enforce the same raise-only rule the manifest validator does (`IllegalArgumentException` on a lowering
  floor). `handled()` lists every name with a registered handler — assert `register()` wired all of them.
- `FakeChannels.subscribe(caller, ref)` runs the floor, then the plugin's `authorize` callback, then
  `onJoin` if it's that user's first subscription — all **synchronously**; a `publish` delivers before it
  returns. A refusal throws `SubscriptionRefusedException`, whose `reason()` is `BELOW_FLOOR`,
  `NO_AUTHORIZER` (nothing registered — every subscribe refused, same as production), or `AUTHORIZER` (it
  answered `false`).
- `FakeSubscription.events()` is everything delivered to it while open, oldest first; `isOpen()`,
  `wasRefused()`. `FakeChannels.published(ref)` is everything a channel ever published, delivered or not —
  useful for asserting a `publish` happened even with nobody subscribed to receive it.
  `FakeChannels.endAccess(userId)` stands in for the host ending a user's subscriptions on an access change.
- **Timers**: `FakePluginContext.now()` reads a fake clock fixed at a `START` instant until
  `advanceTimers(Duration)` moves it forward, running every `runAfter` task that falls due along the way, in
  due-time order — a turn clock tested without sleeping. `pendingTimers()` counts what's still waiting, for
  asserting a timer was armed or cancelled.
- **`InMemoryDocStore.withBackendPrivate(pattern...)`** declares a backend-private key the same way the
  manifest does, for asserting a floor actually bites — pair with `asUser(UUID, Role)`/`asAnonymous()` the
  same way `keyFloors` tests do.

### TypeScript — `makeMockActions` / `makeMockChannels`

```ts
const actions = makeMockActions({
  state: () => ({ version: 1, board: [] }),
  move:  (p) => ((p as { cell: number }).cell > 8 ? actionRejection('off-board') : null),
});
const channels = makeMockChannels();
const ctx = makeMockCtx({ actions, channels });
await mount(ctx);

expect(actions.calls.map((c) => c.name)).toEqual(['state']);
channels.emit('game:1', 'moved', { version: 2 });          // delivers synchronously
expect(root.textContent).toContain('version 2');
channels.reset('game:1');                                   // the component should fetch state again
```

- `makeMockCtx` keeps `actions`/`channels` **`null` unless you pass them** — same "undeclared capability"
  default every other optional surface gets, so a component written against a plugin with neither still has
  to handle their absence.
- `makeMockActions(handlers, { scope?, signedIn? })` answers the way the host does: signed out → 401
  `unauthorized` without a request; an undeclared name → 404 `unknownAction`; an oversized body → 413
  `payloadTooLarge`; otherwise the handler runs. Its return is the answer (`undefined`/`null` → 204,
  resolved `null`); throw or return `actionRejection(code, detail?)` or `rateLimited(retryAfterMs?)` to drive
  the two narrowed error paths. Payload and answer both make a JSON round trip, so a `Date` arrives as the
  wire would send it. `calls` records every call, refused ones included.
- `makeMockChannels({ scope?, signedIn? })` is synchronous too: `subscribe` reports `connecting` then `open`
  before returning (or straight to `refused` if signed out or the name was already `refuse`d). `emit`
  delivers to every live subscription on a name before returning; `reset` fires `onReset`; `refuse` ends
  live subscriptions as `refused` **and** refuses later subscribes to that name; `end` ends them as `closed`
  (an access change). `subscriptions` records every one ever opened, closed and refused included.

### Also new in the test kit

`PluginContext` gained three abstract methods (`actions()`, `channels()`, `runAfter(...)`) — **a hand-rolled
`PluginContext` implementation of your own must add them**; `FakePluginContext` already has them, so a test
built on it needs no change beyond the version bump.
