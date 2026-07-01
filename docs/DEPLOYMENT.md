# Deployment

RelayServer deploys as one Fly.io app that serves both the WebSocket relay and
the built receiver SPA.

## Fly.io

`fly.toml` must route traffic to the same port the Node process listens on:

```toml
[env]
  PORT = '8080'

[http_service]
  internal_port = 8080
```

If Fly logs say it is probing `0.0.0.0:3000`, the deployed config is stale or
incorrect. Redeploy with the checked-in `fly.toml`.

## Build

The root build script owns the receiver production bundle:

```bash
npm run build
```

That runs:

```bash
npm ci --prefix receiver
npm run build --prefix receiver
```

The relay then serves `receiver/dist`.

## Registry

This repo pins public npm in `.npmrc` and `receiver/.npmrc`:

```ini
registry=https://registry.npmjs.org
```

Both lockfiles should resolve packages from `registry.npmjs.org`, not internal
corporate registries.

Check with:

```bash
rg 'npm.apple.com|artifacts.apple.com' package-lock.json receiver/package-lock.json
```

No output is expected.

## Common Checks

```bash
npm ci
npm run build
node --check relay.js
```

The generated `receiver/dist` directory is ignored by Git and should be rebuilt
during deploy.
