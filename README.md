# SafeHaven v1 Relay (M1)

A tiny stateful WebSocket relay. One WebSocket per participant, keyed by a
session token. It fans out a single sender's frames to all receivers, retains
the full per-session event log in RAM, keeps the last video keyframe, and
replays both to any receiver that joins mid-incident. A contact opening the
link late still sees the timeline from the start.

This repository was extracted from the larger SafeHaven v1 project. The relay
wire contract is included in [PROTOCOL.md](PROTOCOL.md) so this GitHub repo is
self-contained.

## What it does

- Runs `http.createServer` + `WebSocketServer({ server })` on one port.
- Connect URL: `ws[s]://HOST/ws?role=<sender|receiver>&token=<urlencoded>&v=1`.
- Validation / close codes:
  - invalid `role` -> `4000`
  - missing `token` -> `4001`
  - second live sender on a token -> `4002`
  - version mismatch -> `4003` only when `STRICT_VERSION=1`
- Per-token `RoomState`: `{ sender, receivers, eventLog, lastVideoIDR }`.
  - Every text frame with top-level `type:"event"` is appended verbatim to
    `eventLog` in arrival order.
  - The last binary frame whose 16-byte header has the KEYFRAME flag (byte 2,
    bit 0) is retained as `lastVideoIDR`.
  - Audio and delta video are never retained.
  - A fresh `incident_start` clears the log as a new session boundary.
- Fan-out: sender text and binary frames are forwarded to all receivers
  verbatim.
- Join replay: on receiver join, the relay sends that receiver the entire
  `eventLog` in order, then `lastVideoIDR`, before any live frame.
- Live frames arriving during replay are buffered per connection and flushed in
  order once replay completes, so history never interleaves with live.
- Presence: receiver join/leave sends `{type:"presence", event, receivers}` to
  the sender only, with the current receiver count.
- Room teardown: when there is no sender and no receivers, the room and its
  whole in-memory log are dropped.

## What it deliberately does not do

- No broad payload reading. It reads only:
  - WebSocket frame type: text or binary.
  - Top-level `type` on text frames, to know whether a frame is a product event.
  - Binary KEYFRAME flag, to know whether a media frame should be retained.
- No inner payload or event parsing, except for the narrow `incident_start`
  check needed to clear the session log. That check reads only
  `payload.event_type` and never logs it.
- No disk persistence.
- No payload logging.
- No WebRTC, SDP, ICE, offer/answer, or per-viewer peer connections.
- No transformation of forwarded data. The sender and receiver own encryption;
  the relay forwards opaque bytes.

## Stale-sender detection

A hard network drop can leave a sender socket lingering without a clean close,
which would make a legitimate reconnect look like a second sender and trigger
`4002`.

To avoid that, the relay runs a ws-level ping/pong heartbeat every 30 seconds.
Each socket is marked dead if it misses a pong, and a dead sender is
`terminate()`d so the token frees up. When a new sender connects, the relay also
proactively reclaims the existing sender if its socket is closed, not open, or
already flagged dead by the heartbeat.

Net effect: duplicate live senders are still rejected, but a real reconnect
after a drop is accepted.

## Static hosting

- Production inside the SafeHaven v1 workspace: if `../receiver/dist` exists,
  the same port serves the built receiver SPA with an `index.html` fallback for
  deep links and `#<token>:<key>` URL fragments.
- Standalone production: deploy the receiver UI separately, or preserve the
  same relative `../receiver/dist` layout beside this repo.
- Development: if `../receiver/dist` is absent, the relay serves 404 for static
  assets and logs a one-time hint. Run the Vite dev server for the UI and point
  it at this relay's `/ws`.

## Run

```bash
cd relay
npm install

# optional: cp .env.example .env  (PORT, MAX_EVENT_LOG)
npm start

# strict version rejection instead of warn-and-allow:
STRICT_VERSION=1 npm start

# custom port:
PORT=9000 npm start
```

Requires Node >= 20, ESM (`"type":"module"`), and the `ws` package.

## Deploy on Fly.io

This repo includes `fly.toml` for the `safehaven-relayserver` app. The relay
listens on `0.0.0.0:${PORT}` and defaults to `8080`, so Fly is configured with:

```toml
[env]
  PORT = "8080"

[http_service]
  internal_port = 8080
```

If Fly logs say it is probing `0.0.0.0:3000`, the deployed config is stale or
was generated before this file existed. Redeploy from this repo so Fly uses the
checked-in `fly.toml`.

## Config

| Var | Default | Meaning |
|---|---:|---|
| `PORT` | `8080` | HTTP + WS listen port |
| `MAX_EVENT_LOG` | `10000` | Per-session event-log safety cap. On overflow, oldest events drop and a non-silent truncation line is logged. |
| `STRICT_VERSION` | unset | `1` closes with `4003` on `v != 1`; default is warn-and-allow. |

## Versioning note

The default version policy is warn-and-allow. Set `STRICT_VERSION=1` to
hard-close `4003` on a version mismatch. A missing `v` is also tolerated with a
warning by default.

## Quick local smoke test

1. Run `npm start` in this repo.
2. In another shell, run the mock sender from the SafeHaven v1 workspace:
   `../tools/mock-sender`. It prints a receiver URL.
3. Open that URL in the receiver UI and watch the incident populate, including
   the full back-timeline if you join late.
