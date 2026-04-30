# NIP-46 Lab

A complete browser-based testbench for [NIP-46](https://github.com/nostr-protocol/nips/blob/master/46.md) (Nostr Connect / Remote Signing). One self-contained HTML file — no build, no server, no install. Open it and start testing your bunker.

![screenshot](docs/screenshot.png)

## Features

**Both connection flows**
- `bunker://` (signer-initiated) — paste a URI from nsec.app, nsecBunker, Amber, etc.
- `nostrconnect://` (client-initiated) — generate a URI with QR code, secret-validated per spec

**All 9 NIP-46 RPC methods**
`connect`, `ping`, `get_public_key`, `sign_event`, `nip04_encrypt`, `nip04_decrypt`, `nip44_encrypt`, `nip44_decrypt`, `switch_relays` — plus a `raw` method for testing extensions.

**Live wire log** — every kind 24133 event in/out, color-coded (req/res/auth/err/sys), click any entry to expand and see the decrypted JSON-RPC payload plus the raw event.

**Auth challenge handling** — when the signer responds with `auth_url`, a banner surfaces the link and the request stays open for the second response.

**Auto-detected transport encryption** — defaults to NIP-04 (most compatible with deployed bunkers), auto-upgrades to NIP-44 when detected. Manual override available.

**Reliability layer**
- `online` / `offline` events, plus Network Information API (`navigator.connection`) — type, downlink, RTT
- `visibilitychange`, `pagehide`, `pageshow` (with `persisted` flag for bfcache)
- Wall-clock skip detector for OS-level suspension that doesn't fire visibility events
- WebSocket `onclose` hook → exponential backoff reconnect (1s → 30s, jittered)
- Heartbeat ping every 45s while connected; force-reconnect after 2 misses
- bfcache restore always force-reconnects (sockets are dead)
- Network-type changes trigger a verification pass (wifi → cellular often kills sockets silently)

**Sign and publish** — after `sign_event`, one click publishes the signed event to damus / nostr.band / nos.lol so you can verify it propagated.

## Usage

Just open `index.html` in any modern browser. No install step.

If you want to host it: drop it on any static host, GitHub Pages, Netlify, Vercel, etc.

```bash
# local serve (any static server works)
python3 -m http.server 8080
# then open http://localhost:8080
```

### Trying it with a real bunker

1. Open [nsec.app](https://nsec.app), create or import a key
2. Click **Connect app** → copy the `bunker://...` URI
3. Paste it into the **bunker://** tab in NIP-46 Lab
4. Click **connect**
5. Approve the connection in nsec.app when the auth challenge appears
6. Try every RPC method from the middle column

### Testing the client-initiated flow

1. Switch to the **nostrconnect://** tab
2. Click **generate URI**
3. Either copy the URI or scan the QR with a signer app
4. Approve in your signer — the lab will auto-detect the connection and fetch your pubkey

## Stack

Pure HTML + JavaScript module, no build step. Loads from CDN:
- [`nostr-tools`](https://github.com/nbd-wtf/nostr-tools) — keys, events, NIP-04, NIP-44, relay client
- [`qrcode`](https://github.com/soldair/node-qrcode) — QR generation

## Spec compliance

Built against the current [NIP-46 spec](https://github.com/nostr-protocol/nips/blob/master/46.md):
- Distinguishes `client-pubkey` / `remote-signer-pubkey` / `user-pubkey`
- Calls `get_public_key` immediately after `connect` per spec
- Validates the `secret` returned in `connect` response for `nostrconnect://` flow
- Subscribes with `since: now - 10s` to avoid stale ephemeral events from non-compliant relays
- Ignores duplicate `auth_url` and duplicate replies per request id
- Handles `auth_url` as `result: "auth_url"` with URL in the `error` field, keeping the request open

## License

MIT
