# Receiver Architecture

The receiver is a Vite + React + TypeScript app. It owns browser-side transport,
media decode, event translation, and the contact dashboard UI.

## Source Layout

```text
src/
  main.tsx                  Vite entry
  App.tsx                   app wiring and session lifecycle
  styles.css                global styles
  transport/                pairing, WebSocket, frame parser, transform seam
  media/                    WebCodecs video, Web Audio PCM, audio levels
  events/                   protocol event translation and incident state
  components/               header, risk banner, tabs, video, audio, map, log
```

## Data Flow

1. `pairing.ts` parses `#<token>:<key>`.
2. `socket.ts` connects to same-origin `/ws?role=receiver&token=...&v=1`.
3. Text event payloads pass through `InboundTransform`.
4. `translateEvent.ts` converts protocol events into internal incident entries.
5. Binary video frames go through WebCodecs into a canvas.
6. Binary PCM frames go through Web Audio scheduling and the audio meter.

## Development And Production

In development, Vite serves the app at `localhost:5173` and proxies `/ws` to the
local relay on `localhost:8080`.

In production, the relay serves the static Vite build from `receiver/dist` and
handles `/ws` on the same origin. The receiver code does not hardcode a relay
host.

## Browser Degradation

If WebCodecs or H.264 support is unavailable, the video panel degrades to a
"video unavailable" state. Audio, GPS, AI labels, tier banner, and timeline stay
live.

## Encryption Seam

`transport/inboundTransform.ts` is an identity passthrough today. It is the
future browser-side decrypt boundary and is constructed with the `key` from the
pairing URL fragment.
