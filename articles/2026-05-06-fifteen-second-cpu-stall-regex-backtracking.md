# A 15-second CPU stall from one unclosed XML tag

*Posted 2026-05-06 · ~9 min read*

> Postmortem from running [llmapi.pro](https://llmapi.pro), an Anthropic-protocol-compatible relay, in production. We had a relay process that occasionally pegged a single core for fifteen seconds straight. The traffic looked normal. The fix was three characters in a regex. This is what happened, and the layered defense we built afterward.

## The symptom

Every once in a while — maybe one in a few thousand requests — the relay would just stop. From the outside it looked like a deadlock or a hung upstream. p99 latency would tick up; Cloudflare would occasionally surface 524s on requests that happened to land on the affected worker; the rest of the workers continued serving fine.

The puzzling part was that nothing else looked wrong. Upstream backends were healthy, network was fine, no spikes on database, no GC pauses long enough to explain the stall. The Node.js process was simply busy. Doing what, we did not yet know.

## Catching it

The shape of the problem made profiling tricky: each event was rare and short-lived. We could not reliably attach a profiler in time. So we made the relay self-instrument: a one-second async timer that, if any single in-flight request had been on the event loop for more than one second, dumped a stack snapshot to disk.

The first snapshot was unambiguous. The stack ended at a regex `exec` call, deep inside the request normalization path. The regex itself looked innocent.

```javascript
// Strips an Anthropic protocol-internal tag that some backends do not handle:
const SR = /<system-reminder>([\s\S]*?)<\/system-reminder>/g;
```

The input was a single user message. About fifty kilobytes of text. Containing — as it turned out — a `<system-reminder>` opening tag that had no matching close tag.

## Why it stalls

The regex above is the textbook example of catastrophic backtracking, but only on a specific kind of input.

When the engine tries to match `<system-reminder>([\s\S]*?)</system-reminder>`, it consumes `<system-reminder>` literally, then enters the lazy `[\s\S]*?` group. Lazy means: try to match as little as possible, then check the next part of the regex. If the next part fails, expand by one character and try again.

For a well-formed input — opening tag, content, closing tag — this is fast. The engine expands the lazy group up to the closing tag, finds it, and exits.

For an input with no closing tag, the engine has to walk the lazy group all the way to the end of the string, expanding by one character at a time, checking each position for `</system-reminder>`. That is O(N²) on input length. At fifty kilobytes, with the JavaScript regex engine's per-step overhead, fifteen seconds is exactly what you would predict.

The model — or, more often, an automated tool emitting messages on the user's behalf — had produced a message that opened a `<system-reminder>` and then, for whatever reason, did not close it. In a string-handling sense, the input was perfectly valid. As input to a backtracking regex without a length bound, it was a denial-of-service vector.

## The fix

Three layers, in order of where they live.

### Layer 1 — fix the regex

The narrow fix is to bound the quantifier. We replaced the lazy `*?` with a length-bounded character class that cannot scan to end-of-string:

```javascript
// Bounded quantifier: at most ~16 KB of inner content; anything longer
// is presumed malformed and skipped via a different code path.
const SR = /<system-reminder>([\s\S]{0,16384}?)<\/system-reminder>/g;
```

This stops the catastrophic case dead. A genuinely-large valid `<system-reminder>` block over 16 KB stops matching, but in practice we have not seen one — the tag is used for short Anthropic-internal hints, not bulk content.

A more thorough fix is to switch the matcher to a non-backtracking engine (`re2`-style). We do this on the JVM side of some other tooling, but for the Node.js relay the bounded quantifier is enough.

### Layer 2 — watchdog timer

The regex fix closes one specific door, but there are other regexes in the codebase, and we will accidentally write more. So the relay now has a per-request watchdog: a four-second hard ceiling for the synchronous portion of any handler. If the event loop blocks longer than that, the watchdog forcibly aborts the request, returns a 504-equivalent error to the client, and increments a counter that the alert pipeline reads.

This requires that the request handler split its blocking work into chunks and yield (`setImmediate`, `await Promise.resolve()`) at known points, so the watchdog can fire. We did not have that discipline before; we do now. Reviewing the existing handlers for blocking sections was about three hours of work.

### Layer 3 — nginx `next_upstream`

Even with the in-process fix and the watchdog, a stalled worker will surface upward. nginx's `proxy_next_upstream` directive lets the load balancer retry a hung request on a sibling worker after a configured timeout. Combined with `max_fails`, this means a single bad request that defeats both Layer 1 and Layer 2 still produces a recoverable user-visible outcome.

```nginx
location / {
  proxy_pass http://relay_pool;
  proxy_next_upstream error timeout;
  proxy_next_upstream_timeout 8s;
  proxy_next_upstream_tries 2;
}
```

Each layer is independent. None of them alone would have kept us safe. Together they form a shape we now apply to other parts of the system.

## Operational lessons

A few things we now treat as durable rules:

1. **Any regex run on user-controlled input must have a bounded quantifier.** No exceptions. Code review for any regex change includes "what is the worst-case input."
2. **Catastrophic backtracking does not look like a regex bug at first.** It looks like a generic latency tail. If you cannot explain a tail, suspect the regex layer first when the input shape can be adversarial.
3. **A per-request synchronous watchdog is cheap insurance.** Implement it before you need it. The implementation is small; the wiring of yield points across handlers is the actual work.
4. **Layered defenses produce graceful failure modes; single-layer defenses produce binary success/failure.** A single fix to the regex would have closed this exact vulnerability and left every other regex in the codebase open. Three layers means the next class of failure has at least two backstops.

## What we did not do

We did not blame the input. The fact that some client emitted an unclosed `<system-reminder>` tag is a thing that happens. The relay's job is to handle the input the wire delivers, not the input we wish the wire would deliver. If a malformed message can stall a worker for fifteen seconds, that is a relay bug, not a client bug.

## Closing

The fix took an hour. Finding it took an afternoon. Building the layered defense around it took two days. Of those two days, the regex change was by far the smallest piece — most of the time went into wiring the watchdog correctly across handlers, exercising the nginx failover under controlled load, and making sure the alert pipeline actually paged when the synthetic test fired.

If you operate a relay that does any normalization on incoming text, this is the easiest class of bug to ship and one of the more painful classes to debug from cold. Cheap to prevent, expensive to find.

---

*llmapi.pro is an independent, Claude-compatible API relay; we are not affiliated with Anthropic. The Claude Code CLI's `ANTHROPIC_BASE_URL` environment variable is documented and supported by Anthropic for pointing at compatible endpoints.*
