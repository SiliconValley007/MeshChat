# MeshChat — decentralized gossip + CRDT + E2EE P2P chat

**Why no backend?** Browsers connect directly with WebRTC. Signaling (the one-time exchange of connection descriptions) is done by you via QR/paste/link/Gist; after the first link, further offers/answers are routed _through the mesh itself_. Public STUN servers only reveal your public address and never see messages.

**WebRTC in one paragraph:** two browsers swap SDP descriptions, run ICE to find a path through NATs, then open an encrypted DataChannel between them.

## Signaling fallback chain

1. **QR** — link is compressed (deflate-raw + base64url) and drawn by the built-in QR encoder. Payloads over ~900 chars are split into `MC1|id|i|n|data` chunks shown as an auto-cycling QR; the scanner reassembles them.
2. **Paste** — copy/paste the blob (chunk lines may be pasted together).
3. **Link** — `#o=…` / `#a=…` in the URL; opening it auto-fills the offer.
4. **Gist** — enter a Gist ID; the app polls `api.github.com` every 60 s (read-only, you paste blobs yourself; degrades with a toast if blocked).

If pairing fails (ICE failure or 45 s without connecting) the app advances to the next method and regenerates the offer. After 4 failures it reports a likely NAT/firewall block.

## CRDT (hybrid OR-Set / LWW ordering)

Messages are replicated _state_, not a log:

```
element  = {id=SHA256(sender|hlc|seq|text), sender, l,c (HLC), q (per-sender seq), text}
add      : insert if id unknown and not tombstoned          (idempotent)
retract  : tombstone {id,sender} — only the author's tombstone counts (observed-remove)
order    : (l, c, hash(sender), id)  → identical on every replica
causality: Hybrid Logical Clock + version vector {sender → max q}
bound    : newest 500 elements; older replays are rejected deterministically
merge    = set union of adds − tombstones  (commutative, associative, idempotent)
```

Ids are re-hashed on receipt, so forged elements are dropped. Double-click your own message to retract it.

## Gossip

- **Push (rumor mongering):** a new message goes to ⌈log₂(n)⌉+1 peers sampled by score; on receipt each peer re-forwards with probability `max(0.4, 1−0.2·hops)` up to 6 hops. Only _newly merged_ elements are forwarded (duplicate suppression).
- **Pull (anti-entropy):** every 3–8 s (jittered) a peer sends `sv` (digest + version vector + a sample of its neighbor view) to 2 sampled peers. On mismatch the peer replies with its id set; both sides push what the other lacks and request what they miss. This gives eventual consistency and heals partitions when any link reappears.
- **Peer sampling:** views carry `[peer, neighbors]` entries; they feed discovery, routing and the graph. Divergence at equal version vectors triggers a visible warning and reconciliation.

## Topology modes

| Mode         | When            | Policy                                                                      |
| ------------ | --------------- | --------------------------------------------------------------------------- |
| Full mesh    | ≤ 6 peers       | up to 9 links each                                                          |
| Partial mesh | 7–10 peers      | 4 best-scoring links; low-score links pruned (`by` goodbye prevents redial) |
| Star         | > 10 live peers | lowest live ID is hub; others keep only the hub link                        |

Score (0–100) = `100·stability − RTT/10 − 70·loss − flap penalty`. RTT/loss come from 2 s pings. Dropped peers enter a priority retry queue (best score first, exponential backoff, dialed via mesh-routed signaling).

## Security

Each connection: fresh **ECDH P-256** key pair → HKDF-SHA256 (salt = `passphrase|room`) → separate send/receive chains → **AES-GCM**. The chain **ratchets every 50 packets** (old chain keys are discarded) and the ECDH private key is dropped right after derivation, so there are no long-term keys (forward secrecy). A passphrase acts as a pre-shared secret: a mismatch causes decrypt failures, shown as an explicit "encryption mismatch" warning, and the link closes after 3 failures. Without a passphrase the key exchange is unauthenticated (an active MITM on signaling could intercept); use one for real protection. Sender IDs are self-asserted (no identity signatures).

## Network view

The optional **Network** panel shows a live force-directed map of peers (layout computed in a Web Worker). Per-peer RTT, stability, loss and score still drive peer selection and pruning internally; NAT/ICE problems and mismatched passphrases surface as toasts.

## Reconnection

Dropped peers enter a priority retry queue (exponential backoff, checked every 2 s). On network change (`online`), tab resume/unlock (`visibilitychange`, `resume`, `pageshow`) the app re-opens the broker socket, pings every link, drops the ones that don't answer within 3.5 s, clears failure back-off and redials all known peers. The broker connection has its own keep-alive watchdog (reconnects after 45 s of silence). A peer that re-announces while we hold a stale link replaces it. Peer lists expire after ~60 s so closed tabs do not linger.

## Usernames

Optional name (max 16 characters, control characters and `<>` stripped) set in the Invite panel and stored in `localStorage` (`mc_name`). It travels inside each message (covered by the message hash, so relays cannot alter it) and in typing events; received bubbles show it instead of the short ID (hover for the ID). Names are not unique or authenticated — bubble colour is derived from the sender ID.

## Identity, history and cleanup

- Peer ID is kept per tab (`sessionStorage`), together with the room code, so a reload rejoins the same room as the same peer and old messages still show as "you". A new tab is a new peer.
- Switching to a different room clears the local chat, peer list and connections first, so conversations from two rooms never mix or leak between rooms.
- Names are exchanged when a link opens, on change, and via gossip views, so a peer is labelled immediately, not only after its first message.
- Unreachable peers are retried ~6 times, then dropped from the retry list and disappear from the map after about a minute.
- _Fresh room_ (checkbox before "Create private room"): one bit of the invite code marks the room as fresh, so every member enforces it: peers only send a joiner messages timestamped after it connected. Members who reload also start empty. Default rooms sync history (up to 500 messages).

## Composer (WhatsApp-style)

The message box auto-focuses on desktop. Enter sends, Shift+Enter adds a new line (on touch devices Enter adds a line and the Send button sends). Clicking Send or any other button does not steal focus if the box had it, and the box is not focused if it did not have focus before. Long messages grow the box up to 5 lines.

## Live map

The Network panel shows every peer with its state (connected, unstable, connecting, reconnecting with retry countdown, disconnected, via mesh), RTT/score/loss, animated links, broker and network status, and an event log.

## Limits

- Mesh is O(n²) connections; tuned for 2–10 peers, degrades to star beyond.
- STUN only, **no TURN**: symmetric NAT/CGNAT/strict firewalls may fail.
- Two peers that lose their only link must re-pair manually.
- History is in memory (500 messages); use _Export state blob_ to back up.
- Set the passphrase before connecting.

## Browser notes

Chrome/Edge/Android: everything. Firefox: everything except in-app QR _scanning_ (no `BarcodeDetector`; use the phone camera app or paste). iOS Safari: WebRTC/crypto work; in-app scanning depends on `BarcodeDetector` support, otherwise use the camera app (it opens the link). Needs Blob Workers and `ResizeObserver`; `CompressionStream` falls back to plain base64.

## Quick connect (private invite link)

Home shows only the chat. **Invite** (top right) slides open the room panel; **Network** shows an optional live map. "Create private room" makes a random 80-bit code (16 chars, e.g. `K7QM-2XPD-9HTR-C4WA`) and an invite link `#c=<code>`; others paste it and press Join. The code never leaves your devices: a public MQTT-over-WebSocket broker (HiveMQ → EMQX → Mosquitto fallback) only sees a hashed topic and AES-GCM ciphertext keyed from the code, so it cannot read the SDP/IPs or guess the room. The code also salts the P2P key derivation. 80 bits is infeasible to brute-force even offline; anyone you give the link to can join, so share it privately. If brokers are blocked, use _Advanced_ manual pairing.
