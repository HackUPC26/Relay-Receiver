# SafeHaven RelayServer

Standalone production service for SafeHaven v1. It runs the Node WebSocket relay
and builds/serves the browser receiver from the same deploy.

The relay implements [PROTOCOL.md](PROTOCOL.md), listens on one HTTP port, and
exposes:

- `/ws` for sender and receiver WebSocket connections
- `/` for the built receiver SPA from `receiver/dist`

## Repository Layout

```text
.
  relay.js          Node relay and static file server
  receiver/         Vite + React receiver source
  PROTOCOL.md       shared wire contract
  fly.toml          Fly.io deployment config
```

Generated directories are not committed:

```text
node_modules/
receiver/node_modules/
receiver/dist/
```

## Quick Start

```bash
npm install
npm run build
npm start
```

The default port is `8080`.

For receiver frontend development:

```bash
npm ci --prefix receiver
npm run dev:receiver
```

The Vite dev server runs at `http://localhost:5173` and proxies `/ws` to the
local relay at `ws://localhost:8080`.

## Configuration

| Variable | Default | Purpose |
|---|---:|---|
| `PORT` | `8080` | HTTP and WebSocket listen port |
| `MAX_EVENT_LOG` | `10000` | Per-session in-memory event log cap |
| `STRICT_VERSION` | unset | Set to `1` to close version mismatches with `4003` |

## Deployment

Fly.io deployment is configured in `fly.toml`. The app listens on
`0.0.0.0:${PORT}`, with Fly routed to internal port `8080`.

```bash
fly deploy
```

The build runs `npm run build`, which installs receiver dependencies and creates
`receiver/dist` before the relay starts.

## Documentation

- [Relay design](docs/RELAY_DESIGN.md)
- [Deployment notes](docs/DEPLOYMENT.md)
- [Migration from the hackathon prototype](docs/MIGRATION_FROM_HACKATHON.md)
- [Receiver README](receiver/README.md)
- [Receiver architecture](receiver/ARCHITECTURE.md)
- [Protocol](PROTOCOL.md)
