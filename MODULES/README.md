# Nostr Collectives - Modules

This directory contains the runnable components of the Nostr Collectives prototype.

| Module | Description |
|--------|-------------|
| **[NosColl-client](./NosColl-client/)** | Web client (Vite + Lit + TypeScript) |
| **[caped-bucket](./caped-bucket/)** | CAP-enforcing Nostr relay (Node.js) |

## Quick Start (Docker)

```bash
cp .env.example .env    # edit as needed
docker compose up --build
```

The client will be available at `http://localhost:<CLIENT_PORT>` and the relay at `ws://localhost:<RELAY_PORT>`.

## Configuration

All settings are in the `.env` file (see [.env.example](.env.example)):

| Variable | Description | Default |
|----------|-------------|---------|
| `CLIENT_PORT` | Host port for the web client | `8080` |
| `RELAY_PORT` | Host port for the relay | `3334` |
| `DEFAULT_RELAYS` | Comma-separated relay URLs the client connects to | `ws://localhost:3334` |
| `EVENT_TTL` | How long the relay keeps events (ms) | `86400000` (24h) |
| `RELAY_NAME` | Relay name shown in NIP-11 info | `NosColl Relay` |
| `RELAY_DESCRIPTION` | Relay description (NIP-11) | |
| `RELAY_PUBKEY` | Relay operator pubkey (NIP-11) | |

### Remote Deployment

For a server accessible at `relay.yourdomain.com`, update your `.env`:

```env
DEFAULT_RELAYS=wss://relay.yourdomain.com
RELAY_PORT=3334
CLIENT_PORT=8080
```

The client fetches its relay list from `/config.json` at startup. In Docker, this file is generated from `DEFAULT_RELAYS` by the container entrypoint — no rebuild needed when relay URLs change.

## Local Development (without Docker)

```bash
# Terminal 1: Start the relay
cd caped-bucket
yarn install
yarn start

# Terminal 2: Start the client dev server
cd NosColl-client
yarn install
yarn dev
```

The client defaults to `ws://localhost:3334` when no `/config.json` override is present.
