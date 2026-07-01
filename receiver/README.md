# SafeHaven Receiver

Browser dashboard for trusted contacts. In production it is built into
`receiver/dist` and served by the relay on the same origin as `/ws`.

The receiver implements the shared [protocol](../PROTOCOL.md) and uses the URL
fragment pairing value:

```text
https://<relay-host>/#<token>:<key>
```

The `token` is sent to the relay. The `key` is retained in the browser for the
deferred encryption transform seam.

## Quick Start

From the relay repo root:

```bash
npm ci --prefix receiver
npm run dev:receiver
```

Or from this directory:

```bash
npm install
npm run dev
```

The dev server runs at `http://localhost:5173` and proxies `/ws` to
`ws://localhost:8080`.

## Checks

```bash
npm run typecheck
npm run build
npm run preview
```

The production build writes `receiver/dist`, which is ignored by Git and built
by the relay repo root during deploy.

## Documentation

- [Architecture](ARCHITECTURE.md)
- [Relay README](../README.md)
- [Protocol](../PROTOCOL.md)
