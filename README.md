# NIP-46 Lab

A complete browser-based testbench for [NIP-46](https://github.com/nostr-protocol/nips/blob/master/46.md) (Nostr Connect / Remote Signing). One self-contained HTML file — no build, no server, no install. Open it and start testing your bunker.

![screenshot](docs/screenshot.png)

## Three modes

Toggle in the top header:

- **client** — connect to an existing bunker as a remote-signing app
- **bunker** — act as a remote signer yourself; expose a `bunker://` URI or dial out to a `nostrconnect://` client
- **relay test** — diagnostic tool for nostr relays (NIP-11, AUTH, ephemeral kinds, NIP-46 echo)

## Features

### Client mode

**Both connection flows**
- `bunker://` (signer-initiated) — paste a URI from nsec.app, nsecBunker, Amber, etc.
- `nostrconnect://` (client-initiated) — generate a URI with QR code, secret-validated per spec

**All 9 NIP-46 RPC methods**
`connect`, `ping`, `get_public_key`, `sign_event`, `nip04_encrypt`, `nip04_decrypt`, `nip44_encrypt`, `nip44_decrypt`, `switch_relays` — plus a `raw` method for testing extensions.

**Live wire log** — every kind 24133 event in/out, color-coded (req/res/auth/err/sys), click any entry to expand and see the decrypted JSON-RPC payload plus the raw event.

**Auth challenge handling** — when the signer responds with `auth_url`, a banner surfaces the link and the request stays open for the second response.

**Auto-detected transport encryption** — defaults to NIP-44 (modern, authenticated), auto-detects NIP-04 from inbound payloads for compatibility. Manual override available.

**Reliability layer**
- `online` / `offline` events, plus Network Information API (`navigator.connection`) — type, downlink, RTT
- `visibilitychange`, `pagehide`, `pageshow` (with `persisted` flag for bfcache)
- Wall-clock skip detector for OS-level suspension that doesn't fire visibility events
- WebSocket `onclose` hook → exponential backoff reconnect (1s → 30s, jittered)
- Heartbeat ping every 45s while connected; force-reconnect after 2 misses
- bfcache restore always force-reconnects (sockets are dead)
- Network-type changes trigger a verification pass (wifi → cellular often kills sockets silently)

**Sign and publish** — after `sign_event`, one click publishes the signed event to damus / nostr.band / nos.lol so you can verify it propagated.

### Bunker mode

Run NIP-46 Lab as the remote signer itself — useful for testing your own client app, demoing the protocol, or short-lived signing sessions in a browser tab.

- **Identity** — paste an `nsec` / hex secret key, or generate an ephemeral one
- **Listen on relays** — multiple relays in parallel, with auto-reconnect on drop
- **Generate `bunker://` URI** — pubkey + relays + rotatable secret + QR code, ready for any client
- **Dial out via `nostrconnect://`** — paste a client URI and the bunker connects to it
- **Approval gate** — auto-approve all (test mode) or manually approve / reject every request
- **Per-client tracking** — sees connected clients, granted permissions, encryption mode, request counts
- **Encryption auto-detect** — replies in NIP-04 or NIP-44 depending on what the client sent
- **All 8 RPC methods implemented** — `connect`, `ping`, `get_public_key`, `sign_event`, `nip04_encrypt/decrypt`, `nip44_encrypt/decrypt`, `switch_relays`
- **Stats panel** — live counters for requests, signatures, encryptions, decryptions, rejections

⚠️ Browser-tab signing keys live in memory only — do not paste production `nsec`s into any web app you don't fully trust. The bunker mode is for testing.

### Relay test mode

A focused check that answers one question: **can this relay carry NIP-46 bunker traffic?** Other relay capabilities (NIP-01 publish, NIP-11, AUTH, etc.) are out of scope.

Four sequential checks, each a hard prerequisite for bunker operation:

1. **WebSocket reachable** — `wss://` handshake succeeds
2. **Accepts kind 24133** — publishes a realistic encrypted kind 24133 event and verifies `OK true` from the relay (some relays blanket-reject the ephemeral 2xxxx range)
3. **Forwards 24133 by `#p` tag** — subscribes with `{ kinds: [24133], "#p": [pk] }`, waits for EOSE, publishes to self, and confirms the relay actually delivers the event
4. **Full encrypted RPC round-trip** — two ephemeral keypairs (client + bunker) on the same socket: client sends an encrypted JSON-RPC `ping`, bunker decrypts, replies with encrypted `pong`, client validates id and result. Measures end-to-end latency.

The verdict at the top is binary: **supports bunker** or **will not work as bunker relay**, with the failing reason called out. A raw protocol trace shows every frame in/out for debugging.

Quick presets are included for nsec.app, damus, nostr.band, nos.lol, nostr.mom.

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
