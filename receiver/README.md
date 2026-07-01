# SafeHaven v1 Receiver

The contact-facing dashboard. This is a Vite + React + TypeScript app served by
the relay in production, so a contact opens the pairing link in a browser and
watches a protected person's live session over the single relay WebSocket.

The receiver implements the wire contract in [../PROTOCOL.md](../PROTOCOL.md).
If this README and the protocol disagree, the protocol wins.

## Run

From the relay repo root:

```bash
npm run dev:receiver
```

Or from this directory:

```bash
npm install
npm run dev
```

The dev server runs at:

```text
http://localhost:5173/#<token>:<key>
```

`token` and `key` are each 32 hex chars. The receiver splits on the first `:`;
`token` is sent to the relay and `key` is retained for the deferred encryption
boundary in `src/transport/inboundTransform.ts`. With no fragment, the
`TokenEntry` overlay prompts for the pairing string.

In dev, Vite proxies the same-origin `/ws` path to the local relay at
`ws://localhost:8080`. In production, the relay serves this bundle and handles
`/ws` on the same origin.

```bash
npm run typecheck
npm run build
npm run preview
```

## Architecture

```text
src/
  main.tsx                  Vite entry
  App.tsx                   Transport + media + incident state wiring
  styles.css                Global styles and keyframes

  transport/
    socket.ts               WebSocket lifecycle, reconnect, demux, close codes
    frame.ts                16-byte little-endian media header parser
    pairing.ts              #<token>:<key> fragment parsing
    inboundTransform.ts     Encryption boundary, identity passthrough today

  media/
    videoDecoder.ts         WebCodecs H.264 Annex-B to canvas
    audioPlayer.ts          PCM Int16 to Float32 Web Audio scheduling
    audioLevels.ts          Audio meter data from decoded PCM

  events/
    translateEvent.ts       Wire product events to internal entries
    incidentState.ts        Session, tier, GPS, trail, and log reducer
    labelMap.ts             AI label display mapping
    theme.ts                Palette and event colors

  components/               SessionHeader, RiskBanner, Tabs, VideoFeed,
                            AudioPanel, GPSMap, IncidentLog, TokenEntry, etc.
```

## Production build

The relay repo root owns the production build:

```bash
npm run build
```

That command runs `npm ci --prefix receiver` and then `npm run build --prefix
receiver`, producing `receiver/dist`. The generated `dist` directory is ignored
by Git and should be rebuilt during deploy.
