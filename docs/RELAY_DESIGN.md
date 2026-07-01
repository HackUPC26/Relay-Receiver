# Relay Design

The SafeHaven relay is a stateful WebSocket service. It is intentionally small:
one HTTP server, one `WebSocketServer`, and token-keyed in-memory room state.

## Responsibilities

- Accept sender and receiver sockets at `/ws`.
- Validate `role`, `token`, and protocol version.
- Enforce one live sender per token.
- Fan out sender text and binary frames to all receivers.
- Retain the per-token product event log and latest video keyframe.
- Replay retained history to receivers that join late.
- Notify the sender when receivers join or leave.

## Room State

Each token maps to:

```text
sender
receivers
eventLog
lastVideoIDR
```

`eventLog` stores sender-origin text frames whose top-level `type` is `event`.
The relay appends those frames verbatim and replays them verbatim.

`lastVideoIDR` stores the latest binary frame whose protocol header has the
KEYFRAME flag. Audio and delta video are never retained.

The room is dropped when no sender and no receivers remain.

## Replay Ordering

When a receiver joins, the relay sends:

1. all retained event frames in arrival order
2. the retained video keyframe, if present
3. live frames that arrived while replay was in progress

Live frames are buffered per joining receiver until replay finishes. History and
live traffic must not interleave for that receiver.

## Privacy Boundaries

The relay does not decode media, persist events, or log payloads. It reads only:

- WebSocket frame type
- top-level text envelope `type`
- the narrow `payload.event_type === "incident_start"` check needed to clear a
  prior session log
- binary KEYFRAME flag in the cleartext media header

Payload encryption can be added at the sender/receiver transform seams without
changing relay routing.

## Stale Sender Handling

A hard network drop can leave a sender socket open long enough to block a valid
reconnect. The relay runs a ping/pong heartbeat and terminates sockets that stop
responding. When a sender connects, the relay also reclaims an existing sender
socket if it is closed, not open, or already flagged dead.

Duplicate live senders are rejected with `4002`; valid reconnects are accepted
after stale socket reclamation.

## Close Codes

| Code | Meaning |
|---:|---|
| `4000` | invalid role |
| `4001` | missing token |
| `4002` | sender already connected |
| `4003` | version mismatch when `STRICT_VERSION=1` |
