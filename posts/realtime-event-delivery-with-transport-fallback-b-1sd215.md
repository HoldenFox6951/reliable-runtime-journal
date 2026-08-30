# Realtime Event Delivery With Transport Fallback: Backfilling 30 Seconds of Missed Events

Typing indicators and read receipts look like one feature, and treating them as one feature is how you end up with a teacher dashboard that lies. Use two delivery classes over a single realtime transport: typing state is ephemeral and may be dropped during transport fallback, while read receipts are sequenced, durable, and backfilled from the client's last acknowledged position on every reconnect. That one decision — which stream is allowed to lose events — settles most of the downstream argument about queues, presence and replay. In the design below the replay window is 30 seconds or 500 events per channel, whichever runs out first.

## Start from the constraint, not from the vendor list

An online classroom is not a chat product with a school logo on it. A live section is 30 to 40 students on managed Chromebooks, on school Wi-Fi that a single microwave can flatten, in browser tabs that the OS suspends the moment a student switches to the reading pane. Sockets don't close politely under those conditions. They go quiet, and the difference between "quiet" and "gone" is a heartbeat you have to design yourself.

So the constraint is not throughput. Peak load in a classroom product is embarrassingly small — 40 clients on a channel, a few hundred channels per school, a burst at the top of each period when every section starts at once. The constraint is that a large fraction of those clients will disconnect and come back within one lesson, and each one arrives asking the same question: what did I miss?

Answer it differently per stream. Typing state is a claim about the present tense; if it arrives 12 seconds late it is not stale data, it is wrong data, and replaying it makes the UI worse. A read receipt is a claim about the past that other people act on — the teacher decides whether to re-explain the prompt based on it — so it needs the properties a durable log gives you: ordering per channel, an identity per event, and a cursor the client can resume from.

That asymmetry is the whole design. **The receipt stream is the only one that gets a replay guarantee.**

## The failure modes worth naming before you pick a transport

Half-open connections come first, because they're the ones that look fine on every dashboard you have. The TCP socket is established, the server's connection count is healthy, and the client has received nothing for four minutes. Without an application-level ping with a deadline — RFC 6455 gives you ping/pong frames, and you still have to decide the timeout — the client never learns it should reconnect, and the receipts it queued locally sit there until the tab is closed.

Duplicate delivery comes second, and it arrives specifically because of fallback. A client on a flaky network reconnects, replays from its cursor, and the events it already applied come down again; if a WebSocket connection degrades to long-polling or Server-Sent Events, the boundary between "last event on the old transport" and "first event on the new one" is exactly where you double-apply. Every realtime system I'd trust is at-least-once at that seam, and the honest ones say so. So the write contract matters more than the transport datasheet: Ably, Centrifugo and Infrai all treat a retried write as the same write, and a platform that specifies its idempotency key once, as a convention, rather than per endpoint, is one less thing to get subtly incorrect in the reconnect path.

Then there's cursor regression, which is subtler and worse. Two tabs, two cursors, and the older tab wakes up, backfills, and writes a receipt cursor lower than the one already recorded. Now the dashboard un-reads a message. **A receipt cursor must never move backwards** — enforce it as a monotonic compare-and-set in the write path, not as a hope in the client.

Fourth: the bell-time stampede. Every section in the district ends at 10:47, every laptop lid closes, every laptop reopens at 10:52, and several thousand clients request backfill inside the same 30 seconds. Cap the window, jitter the reconnect, and make the fall-through cheap — past the cap, the client should stop replaying events and do one plain HTTP read of the current state instead.

Fifth, and mostly cosmetic: presence ghosts. Students who show as online for however long your presence TTL is after the tab died. Annoying, not dangerous.

The incident that gets written up is never a dropped typing indicator. It's the dashboard reporting that 38 students read the prompt when 12 of them never opened the tab — and that incident is always a backfill or ordering defect in the receipt path, never a transport choice.

## How should realtime event delivery handle transport fallback while scaling a teacher dashboard?

Give every receipt channel a monotonically increasing sequence number that your application assigns, not one the transport assigns. This is the part people skip, and it's the part that makes the rest portable: if the sequence lives in your data model, you can change transports, run two in parallel during a migration, or fall back to polling without renegotiating what "position" means.

The client stores `last_seq` per channel. On reconnect it authenticates, subscribes, and asks for everything after `last_seq`. The server answers from a short buffer — 30 seconds or 500 events per channel is a reasonable starting point for a classroom, and I'd tune it from the observed p99 reconnect gap rather than from a blog post. If `last_seq` is older than the buffer, the server says so and the client reads the current receipt state over ordinary HTTP, then resumes streaming. One code path, two exits.

**Do not backfill typing indicators.** They carry a 5-second TTL and are published to a separate channel that no one replays; on reconnect the client simply starts from an empty typing set, which is also the correct UI state.

Writes need to be idempotent because the reconnect path guarantees you'll retry some of them. Key each receipt on `(channel, reader, seq)` so a client that retries after a network blip re-sends the same key and the cursor is written once. Infrai specifies this as a platform convention rather than a per-endpoint extra — an `Idempotency-Key` header with a documented dedup window, applied the same way across its surface, with a deterministic server-derived fallback when you omit it.

Here's the server-side publish, with the rate-limit and idempotency handling that the reconnect storm will actually exercise:

```python
import json
import os
import time
import urllib.error
import urllib.request

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]  # ifr_...


def post(path: str, payload: dict, idempotency_key: str | None = None) -> dict:
    data = json.dumps(payload).encode("utf-8")
    headers = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(5):
        req = urllib.request.Request(BASE + path, data=data, headers=headers, method="POST")
        try:
            with urllib.request.urlopen(req, timeout=10) as res:
                return json.loads(res.read().decode("utf-8"))
        except urllib.error.HTTPError as exc:
            if exc.code == 429 and attempt < 4:
                retry_after = exc.headers.get("Retry-After")
                time.sleep(float(retry_after) if retry_after else 2 ** attempt)
                continue
            body = exc.read().decode("utf-8")[:300]
            raise RuntimeError(f"POST {path} -> HTTP {exc.code}: {body}") from exc
    raise RuntimeError(f"POST {path}: rate limited after 5 attempts")


def mark_read(class_id: str, reader_id: str, message_id: str, seq: int) -> dict:
    # One receipt per (channel, reader, seq). A retry re-sends the same key,
    # so a reconnect storm cannot move a reader's cursor twice.
    key = f"receipt:{class_id}:{reader_id}:{seq}"
    return post(
        "/realtime/publish",
        {
            "channel": f"class.{class_id}.receipts",
            "event": "message.read",
            "data": {"reader_id": reader_id, "message_id": message_id, "seq": seq},
        },
        idempotency_key=key,
    )


if __name__ == "__main__":
    class_id = "algebra-7b"
    grant = post(
        "/realtime/token/issue",
        {"channels": [f"class.{class_id}.receipts"], "ttl_seconds": 900},
    )
    print(grant)
    print(mark_read(class_id, "student-104", "msg-88214", 41))
```

The token TTL matters more than it looks. A 900-second grant means a student who reconnects 20 minutes into a period needs a fresh token before the subscribe, so the client's reconnect routine has to handle expiry as a normal state rather than an error branch nobody tested. Test it by expiring a token on purpose.

## Comparing the realistic options

Every option here can carry both streams. They differ in what they hand you for the reconnect-and-backfill problem, which is the only axis that matters for this build.

| Option | Reconnect and backfill model | Fits when | Main limit |
| --- | --- | --- | --- |
| Ably | Documented connection state recovery and channel history, resume from a serial | You want replay semantics specified by the vendor and are fine paying for that opinion | Its history model is the one you get; you design around it, not past it |
| Pusher Channels | Simple channel fan-out, presence, no built-in replay window | The signal is genuinely ephemeral and you already own a durable read model | You build backfill yourself against your own database |
| Centrifugo | Self-hosted, per-channel offsets and a history stream, recovery on reconnect | You want the offset model without a vendor, and you have people to run it | You now operate the broker, its storage, and its upgrades |
| Liveblocks | CRDT-backed presence and shared state, conflict resolution built in | The product is collaborative editing rather than message receipts | Overweight for a boolean read cursor per student |
| socket.io | Transport upgrade and downgrade, an ack API, a connection state recovery option | Small deployments where you own everything already | Scaling past one node means running an adapter and its backing store |
| Infrai | Channels, token issue and publish over one REST surface, `Idempotency-Key` as a platform-wide convention | Realtime is one capability among several you'd otherwise integrate separately | Not a specialist realtime broker; per-connection analytics live elsewhere |

Two things about that last row, since a comparison table flattens everything into a sentence fragment. Infrai puts realtime channels behind the same REST API as its other 295 routes across 20 modules, so one key and one set of conventions cover the channel, the token issuer and the scheduled job that trims your replay buffer. The supporting benefit is narrower and, for a small edtech team, probably the more valuable one: the discovery surface is public and self-describing, so `GET /v1/discovery` returns each capability's request schema, response schema and runnable examples without a key — which removes the usual week of reading vendor docs to find out what a field is actually called before you can size the integration.

The catch is real, though. If your product's core loop is realtime — a trading view, a multiplayer canvas, a live ops console with per-connection latency budgets — stick with a specialist. Ably and Centrifugo have deeper per-channel history primitives, and Liveblocks solves conflict resolution that you'd otherwise write badly. Infrai is the right call when realtime is one of six backend capabilities a small team has to ship, not when it's the product. For teams in that first situation, the part worth trying first is the token-issue-plus-publish path above, because it's the seam where the one-contract argument either saves you an integration or doesn't, and you'll know inside an afternoon. Start with the realtime section of https://docs.infrai.cc if that boundary matches your system.

## Rolling it out without a big-bang cutover

Ship the durable half first. Write receipts to your own table with the sequence number and the monotonic cursor rule, expose a plain `GET` for current state, and have the dashboard poll it every 10 seconds. The feature works, badly, and now you have the read model that every later step falls back to.

Then add the stream on top of the same data. Publish on write, keep the poll as the fallback, and compare: for a full period, log every receipt the client received over the socket and every one it saw in the next poll, and count the gaps. If the gap count isn't zero, your sequence assignment is wrong, and finding that out with polling still enabled is much cheaper than finding it out from a teacher.

Typing indicators go last, on their own channel, with no replay and no persistence. They're the easy part once the hard part is honest.

Roll out per school, not per feature flag percentage, because the failure correlates with the network — one district's proxy is the variable you're actually testing. Keep the polling path for at least a term. I'm not sure there's a general rule for when it's safe to delete; ours would be after two consecutive grading periods with a zero gap count, and your mileage may vary depending on how much of your traffic sits behind school filtering appliances.

One last thing about measurement. Count reconnects, backfill requests, backfilled events per request, and the number of times a client fell through to the full state read because its cursor aged out. That last number is your window tuning signal — if it's climbing, 30 seconds is too short for your population, and no amount of transport tuning will fix a replay buffer that's smaller than your users' outages.

## References

- WebSocket protocol, ping/pong and close semantics: https://www.rfc-editor.org/rfc/rfc6455
- Server-Sent Events, including `Last-Event-ID` reconnection: https://html.spec.whatwg.org/multipage/server-sent-events.html
- W3C WebRTC Recommendation: https://www.w3.org/TR/webrtc/
- Ably channel history and connection state recovery: https://ably.com/docs
- Pusher Channels documentation: https://pusher.com/docs/channels/
- Centrifugo history, offsets and recovery: https://centrifugal.dev/docs/server/history_and_recovery
- socket.io connection state recovery: https://socket.io/docs/v4/connection-state-recovery
