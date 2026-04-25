# Drain deploy: how to swap a relay without tearing live SSE

*Posted 2026-04-25 · ~8 min read*

> This is a postmortem-style note from running [llmapi.pro](https://llmapi.pro), an Anthropic-protocol-compatible relay, in production. This entry covers a deploy incident that cost us thirty minutes of midafternoon uptime and produced a technique we now call drain-deploy.

## The setup

We run the relay behind a stateless nginx layer. The relay itself is a Node.js process — nothing exotic. For most of year one, deploys were straightforward: `systemctl restart relay`. It worked. New connections picked up the new binary immediately, old connections finished, and we moved on.

That stopped working the moment we added a long-running Claude Code session that could stretch to forty minutes. A rolling restart mid-session meant that session's SSE stream broke, Claude Code surfaced a reconnect loop, the user lost context, and we got a support email with a timestamp we couldn't argue with.

The incident was embarrassing precisely because the fix was obvious in hindsight. We just hadn't built for it.

## What "tearing" actually means here

When nginx routes a request to a backend and that backend process dies, nginx returns a 502 before the SSE stream finishes. From Claude Code's perspective, the connection drops and it attempts to reconnect. For a model that's mid-response — streaming tokens back — the reconnect may land on a new relay process that has no knowledge of the in-flight response. The model continues generating on the backend, but the relay that asked for it is gone. Tokens get lost. The session degrades.

The window where this happens is not long: maybe ten to thirty seconds for a typical response. But if your p99 response time is elevated or you're handling long tool-use chains, that window grows. And the probability of hitting it during a deploy is a function of your connection arrival rate — not your deploy schedule.

## The naive restart sequence (what we did first)

```
1. nginx routes new requests to relay-A
2. we restart relay-A → 502s flood in for ~5 seconds during startup
3. nginx health checks pass, new connections restore
4. relay-A is now running new binary
```

This is fine for stateless services. For a streaming relay that holds an SSE stream open to a client, step 2 is where the incident starts.

## The drain sequence (what we do now)

```
1. relay-A: stop accepting NEW upstream connections (drain mode)
2. wait: let existing SSE streams finish or hit their timeout
3. nginx: remove relay-A from the upstream pool
4. deploy: restart relay-A
5. nginx: add relay-A back to the pool
6. relay-A: re-enable accepting new connections
```

This is a standard load-balancer drain sequence. The relay-side implementation is where it gets specific.

## Relay-side drain mode

We added a `/health/drain` endpoint that the deploy script hits before touching the process. When drain mode is active, the relay:

- Stops polling nginx for new work — no new `await fetch()` calls to the upstream
- Completes any in-flight requests already received
- Closes its server socket to new incoming connections only after all active SSE streams have ended, or after a hard timeout of 120 seconds (whichever comes first)

The hard timeout exists because an upstream model bug once produced a response that simply never ended — a tool-use loop that the model never broke out of. We needed a ceiling. 120 seconds is an upper bound on any single response we intend to serve; anything that runs longer is already a protocol error.

```javascript
// Simplified drain state machine
const drainState = { active: false, deadline: null, activeStreams: 0 };

async function handleDrain() {
  drainState.active = true;
  drainState.deadline = Date.now() + 120_000;

  // Wait for active streams to drain, or deadline
  while (drainState.activeStreams > 0 && Date.now() < drainState.deadline) {
    await sleep(500);
  }

  server.close();
}
```

The `activeStreams` counter is incremented when we begin streaming a response to a downstream client, and decremented when that stream's SSE terminates. It's an in-process counter — no Redis required, since a relay instance only tracks its own streams.

## The nginx piece

nginx's `max_fails` and `fail_timeout` directives handle upstream health, but drain deploy is a graceful operation, not a failure. We use `nginx -s reload` after updating the upstream weight to zero rather than relying on health-check fail counters. The reload is fast enough that new connection attempts don't pile up.

The full deploy script (called from CI, triggered by a new image tag):

```bash
# 1. Signal drain on relay instance 1
curl -X POST http://relay-1:3000/health/drain

# 2. Wait up to 120s for active streams to clear
for i in $(seq 1 24); do
  remaining=$(curl -s http://relay-1:3000/health/streams)
  [ "$remaining" -eq 0 ] && break
  sleep 5
done

# 3. Remove from nginx upstream, reload
sed -i '/relay-1/d' /etc/nginx/upstreams.conf
nginx -s reload

# 4. Restart the relay process
systemctl restart relay

# 5. Restore nginx upstream
echo "server relay-1:3000;" >> /etc/nginx/upstreams.conf
nginx -s reload

# 6. Clear drain mode — relay now accepts new connections
curl -X DELETE http://relay-1:3000/health/drain
```

Steps 3 and 5 are the two nginx reloads. They are cheap. The relay restart in step 4 happens with zero live traffic because steps 1–2 have already drained all streams.

## The keep-alive complication

SSE streams are long-lived by design. A keep-alive heartbeat runs every fifteen seconds on our relay — a `: heartbeat\n\n` comment line sent over each active stream. This keeps proxies and load balancers from closing idle connections.

The problem: if you drain a relay and the heartbeat stops before the SSE stream has formally ended, the client sees a silent gap. Some HTTP clients reconnect; others wait. Either way, the drain has introduced a behavior change visible to the client.

The fix: the heartbeat continues through the drain window unchanged. The heartbeat is a no-op from a protocol perspective — Claude Code ignores comment lines in an SSE stream. We treat drain mode as "finish what you've started," and the heartbeat is part of that.

## What we got wrong the first time

Our first drain implementation called `server.close()` immediately upon receiving the drain signal. This closed the server socket and aborted all active streams before they could finish. The result was identical to a hard restart — we had implemented a drain that behaved like a hard restart.

The fix was the state machine above: track active streams, wait for them to reach zero, then close.

## Edge case: the reconnect during drain

What happens if Claude Code's client reconnect logic fires during the drain window — between step 1 (drain active) and step 5 (relay restarted)? The client makes a fresh connection to nginx. nginx, if not yet reloaded, routes it to relay-A in drain mode.

We handle this by having drain-mode relays send a `503 Service Unavailable` on new incoming HTTP requests with a `Retry-After: 5` header. Claude Code's reconnect handler respects `Retry-After`. The nginx reload in step 3 happens within seconds of the drain signal, so the window is narrow — but the 503 ensures we don't silently drop a reconnecting client.

## Results

After implementing drain deploy:

- Deploys that occur during peak traffic produce zero 502s on active SSE streams
- The 120-second hard timeout handles the rare tool-use loop case without manual intervention
- Claude Code reconnect loops dropped from a mean of 2.3 per deploy (measured over six deploys pre-fix) to zero

The remaining surface area for disruption is the nginx reload in steps 3 and 5. Each reload drops approximately 0–1 connections during the reload window (typically under 100ms). We've measured this at roughly 0.3% chance of a dropped connection per reload event under moderate load. We have not yet addressed this further; it would require moving to a zero-downtime reload mechanism at the nginx layer, which is a separate project.

## What we'd do differently

We would build the drain endpoint on day one. The state machine is twelve lines of JavaScript. The nginx piece is another twenty. The cost of getting it wrong — a support email with a session-loss timestamp — is asymmetric.

The incident that triggered this was not a complex failure. It was a routine operation that we hadn't instrumented for graceful behavior. That turned out to be the whole problem.

--- *[← Back to journal index](../README.md)*
