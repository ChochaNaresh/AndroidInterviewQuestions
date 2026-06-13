# Android System Design — Short Answers (Quick Revision)

> Condensed answers for rapid review before an interview. For full explanations, code, and trade-offs see `android_system_design_interview.md`.

---

## 1. Design an Image Loading Library

Load images from URL/file/resource into an `ImageView` efficiently with a three-tier cache: **active resources** (weak refs to in-use bitmaps), an in-memory `LruCache<Key, Bitmap>` (~1/8 of heap), and a `DiskLruCache` of encoded bytes. The cache key must encode `url + targetW/H + transforms + format`. Avoid OOM by **downsampling** with `inSampleSize` (read bounds via `inJustDecodeBounds`) and reuse memory via a **bitmap pool** (`inBitmap`) to keep scrolling jank-free. Decode off the main thread, deliver on it, cancel stale requests on recycled views (tag the view), and bind to the lifecycle.

---

## 2. Design a File Downloader Library

Download large files with pause/resume/cancel and progress callbacks. The core trick is the HTTP **`Range: bytes=N-`** header: persist `downloadedBytes` (plus `ETag`/`Last-Modified`) so you resume from the offset after process death; a `206 Partial Content` confirms support. Stream the body to a `RandomAccessFile` (never hold the file in memory), **throttle** progress callbacks/DB checkpoints (~every 65 KB or 100 ms), and atomically rename `.tmp` → final on success. Use a dispatcher with a max-concurrency limit; use `WorkManager` for downloads that must survive app death.

---

## 3. Design WhatsApp

Real-time 1:1/group messaging where the **local DB (Room) is the single source of truth** and the UI never waits on the network. Transport is a persistent **WebSocket** for live delivery, with **FCM** waking the app to reconnect when the socket is down. Sending writes optimistically with `status=PENDING` (client-generated **UUID** for dedup/idempotency); a server-assigned monotonic `serverSeq` orders messages. Delivery receipts (sent → delivered → read) flow back over the socket. **E2E encryption** uses the Signal Protocol (X3DH handshake + Double Ratchet); media goes to a CDN out-of-band with only a reference + key in the message.

---

## 4. Design Instagram Stories

Ephemeral full-screen auto-advancing media expiring after 24h, with a tray showing unseen rings. The key to "instant" playback is **aggressive prefetch**: fetch the next segment and the next user's first media ahead of time, backed by an LRU media cache. Use `ViewPager2` (one page per user) with an inner segment controller, where a **single timer** drives both the segmented progress bar and auto-advance (pause on long-press, resume remaining time). Reuse a small **ExoPlayer pool** (current/next) rather than creating one per segment, and release players in `onStop`.

---

## 5. Design a Networking Library

A Retrofit/OkHttp-style declarative HTTP client: an annotated API interface backed by a proxy that builds requests, an **interceptor chain** (chain-of-responsibility for auth/logging/cache/retry), a transport (wrap OkHttp for connection pooling/HTTP-2/gzip), pluggable converters, and a dispatcher thread pool delivering on the main thread. Expose `suspend` functions via `suspendCancellableCoroutine` so cancellation propagates. Honor HTTP caching (`ETag`/`If-None-Match` → `304`) and map failures into a **sealed result** (Success/HttpError/NetworkError/ParseError).

---

## 6. Design Facebook Nearby Friends

Opt-in feature showing nearby friends where **battery efficiency is paramount**. On the client use `FusedLocationProviderClient` with **balanced** priority (not raw GPS), plus **geofences** and **activity recognition** to sample only when moving, and **batch uploads** via `WorkManager` rather than per-fix. The backend indexes the latest location with a **geohash/quadtree**, queries the user's cell + 8 neighbors, intersects with the friend list, and computes haversine distance. Privacy-first: bucketed/approximate distances, auto-expiring sharing (`expiresAt`), and server-side TTL on locations.

---

## 7. Design a Caching Library

A general-purpose tiered cache: a fast in-memory `LruCache` over a persistent `DiskLruCache`, with a pluggable serializer, injectable clock for TTL, and a pluggable eviction policy. Reads are **read-through** (memory → disk, promoting on hit); writes are **write-through** (memory + disk). Eviction is abstracted behind an interface (LRU default, LFU/FIFO pluggable). Thread-safety via synchronized memory tier and serialized disk writes. TTL can be **lazy** (check on read) or **active** (periodic sweep).

---

## 8. Location-Based App Design Problems

For ride-hailing/delivery/geofencing/fitness, interviewers probe **battery, accuracy, permissions, and background limits**. Building blocks: `FusedLocationProviderClient` for fixes, the **Geofencing API** for enter/exit (OS-monitored, cheap), **Activity Recognition** to sample only when moving, and `WorkManager` or a foreground service for background work. Request `FINE` only when needed (`BACKGROUND_LOCATION` is a separate, scrutinized grant; Android 14+ needs FGS type `location`). Battery strategy: lowest sufficient priority, larger intervals/displacement, motion-triggered wakes, batched uploads, stop when backgrounded.

---

## 9. Offline-First App Architecture

The **local DB (Room) is the single source of truth**; the network is a sync mechanism, not the read path. The UI observes `Flow` from Room (instant reads, no network spinner) and writes optimistically with a `pendingSync` flag or an **outbox** table. A `SyncWorker` (WorkManager, network-constrained, retrying) drains the outbox and does **delta sync** (pull only changes since the last cursor). Resolve conflicts via per-record `version` with last-write-wins/server-wins/merge. Trade-off: optimistic UI needs rollback logic when the server rejects a write.

---

## 10. Design an LRU Cache

O(1) `get`/`put` via a **HashMap + doubly-linked list**: the map gives O(1) key→node lookup; the list keeps recency order with most-recently-used at the head. On access, move the node to the head; on overflow, evict the tail. Space is O(capacity). A `LinkedHashMap(cap, 0.75f, accessOrder=true)` with `removeEldestEntry` is the shortcut, but interviewers usually want the manual map+DLL. Follow-ups: thread-safety (striped locks), eviction by byte size for bitmaps, and TTL.

---

## 11. Design an Analytics Library

Track events fire-and-forget and reliably batch-upload them across offline and process death. `track()` returns immediately, persisting events to a local DB with a UUID **idempotency key**. Upload in **batches** (size threshold, time interval, or on background) via **WorkManager** (retry with backoff, survives death); delete rows only after a server ack → **at-least-once** delivery with server-side dedup. Auto-attach common context (device, OS, app version, session id) and tie events to a user via `identify`.

---

## 12. Design a Logging Library

A Timber-style facade routing leveled, tagged logs to pluggable **sinks** ("trees"): Logcat in debug; file/crash-reporter/remote in release. The key performance trick is passing the message as a **lambda `() -> String`** so the string is built only if a sink will consume it. Wrap each sink call in `runCatching` so logging never crashes the app. File/remote sinks write off the main thread and run messages through a **PII redaction** step. Configure per build (`if (BuildConfig.DEBUG) plant(LogcatSink) else plant(CrashSink/FileSink)`).

---

## 13. Design the Uber App

Two clients (rider, driver) over one backend. Live location streams from the driver via `FusedLocationProvider` (high accuracy on-trip) over a **WebSocket**, relayed to the matched rider; the backend keeps a **geospatial index** (geohash/S2/quadtree) for nearest-driver matching. A server-authoritative **trip state machine** (REQUESTED → ASSIGNED → ARRIVING → STARTED → ENDED → PAID → RATED) is mirrored and persisted locally to survive reconnect/crash. Mobile concerns dominate: battery (foreground service, tuned intervals), network resilience (queue + reconnect with backoff), and **marker interpolation** so the car animates instead of teleporting.

---

## 14. HTTP Request vs Long-Polling vs WebSocket vs SSE

Four ways to get server data, by real-time-ness:

- **HTTP request** — req/response, short-lived; must poll for updates. Simple, cacheable, not real-time.
- **Long-polling** — server holds the request open until data/timeout, then client re-issues. Near real-time over plain HTTP, but reconnect overhead.
- **WebSocket** — persistent **full-duplex** TCP after an upgrade handshake; ideal for chat/multiplayer/trading. You manage reconnect/heartbeats; harder to scale.
- **SSE** — long-lived `text/event-stream`, **one-way server→client**, auto-reconnect; great for feeds/notifications when you don't need upstream.

---

## 15. How Voice and Video Calls Work

Real-time media over **WebRTC**: capture → **encode** (Opus audio; VP8/9/H.264/AV1 video) → transport over **UDP/RTP** (low latency beats reliability — drop late frames) → decode → render. A separate **signaling channel** (your own WebSocket server) exchanges **SDP** offers/answers and **ICE** candidates. NAT traversal uses **STUN** (discover public IP for direct P2P) and **TURN** (relay when direct fails), orchestrated by **ICE**. Group calls use an **SFU** (each client sends one stream up, server forwards) since P2P mesh is N². WebRTC adapts bitrate/resolution and uses jitter buffers + FEC for graceful degradation.

---

## 16. Data Syncing on Unstable Networks

Keep client/server consistent on flaky links: **local DB as source of truth**, an **outbox/mutation queue** that survives death, and **WorkManager** for retrying, network-constrained background sync. Use **idempotency keys** (client UUIDs) so retries don't duplicate, **delta sync** (only changes since the last cursor) to minimize bytes, and **chunked/resumable transfers** for large payloads. Add versioned **conflict resolution**, connectivity-triggered sync (`NetworkCallback`), and **exponential backoff with jitter** to avoid thundering herds. Result: instant optimistic writes, nothing lost, no duplicates, minimal data over the wire.

---

## 17. How "Where Is My Train" Works Without Internet

The core experience needs no live server call. The full **schedule, station list, and route geometry are stored locally** (SQLite), so the app computes expected position from the timetable. The device's **GPS works offline** (satellite, not network); the app maps the GPS fix onto the local route polyline to infer "between A and B." Historically it also used the connected **cell-tower ID** (available without data) plus crowdsourced tower→track mappings, and **dead reckoning** (interpolating against scheduled timings) between fixes. It syncs schedules and uploads anonymized data when online.

---

## 18. Database Normalization vs Denormalization

**Normalization** splits data into related tables so each fact is stored once (FKs) — minimal redundancy, strong integrity, faster/simpler writes, but reads need JOINs. **Denormalization** deliberately duplicates/pre-joins data — faster reads and simpler queries, but larger storage, update anomalies, and heavier writes. Normalize for write-heavy/OLTP; denormalize for read-heavy/analytics. On Android, local DBs often lean **slightly denormalized** for fast UI reads (e.g. a `lastMessageText` on the chat row) while keeping correctness-critical data normalized.

---

## 19. Hash vs Encrypt vs Encode

Three different purposes: **Encode** changes data *format* for transport (Base64/URL/UTF-8) — reversible, no key, **not security**. **Encrypt** turns plaintext into ciphertext with a **key** for confidentiality — two-way, decryptable with the key (AES, RSA). **Hash** is a **one-way** fixed-length digest for integrity/fingerprints and password storage (use slow salted bcrypt/scrypt/Argon2, not raw SHA-256; HMAC adds a key). One-liner: encode to be read by machines, encrypt to keep secret, hash to verify without storing.

---

## 20. Webhook vs Polling

**Polling**: the client repeatedly asks "any new data?" — simple, works behind NAT, no inbound endpoint, but wasteful and latency = the interval. **Webhook**: the server pushes an HTTP callback to a URL the client registered — real-time and efficient, but the client must expose a reachable public endpoint, verify signatures, and absorb retries/duplicates. Mobile devices can't host reliable webhooks (no stable public URL, OS background limits), so the mobile equivalent is **push notifications (FCM)** — the server pushes via Google's infra to wake the app.

---

## 21. Options for Real-Time Updates in an Android App

From cheapest to most real-time: **polling** (simple, wasteful), **long-polling**, **SSE** (one-way feeds), **WebSocket** (bidirectional, best for chat/presence; costs battery), **FCM push** (wakes a killed/backgrounded app to then sync — the standard, but throttled), **Firebase RTDB/Firestore listeners** (managed real-time sync with offline support; vendor lock-in), and **gRPC streaming** (efficient typed bidirectional over HTTP/2). On mobile, choose by app state: FCM when backgrounded/killed, WebSocket when foregrounded and bidirectional, SSE for one-way, polling for low urgency — battery and OS background limits dominate.

---

## 22. Options for Network Optimization in a Mobile App

- **Payload**: gzip/Brotli compression, efficient formats (Protobuf/FlatBuffers), pagination/field selection, image optimization (WebP/AVIF, right resolution, thumbnails first).
- **Avoid requests**: HTTP caching (`ETag`/`304`), offline-first local cache with delta sync, dedupe in-flight requests.
- **Connection**: keep-alive + HTTP/2 multiplexing, HTTP/3 (QUIC), CDN, TLS session resumption.
- **Scheduling/battery**: batch requests to align with radio wake windows, defer non-urgent work via WorkManager, prefetch on Wi-Fi, retry with backoff + jitter.

---

## 23. Firebase Remote Config in Android

A cloud key–value store to change app behavior/appearance **without a store release**. Define **in-app defaults**, the server overrides them, the SDK **fetches** (cached, throttled ~12h in prod) and you **activate** to make values live, then read via `getString/getBoolean/...`. Use cases: feature flags / kill-switches, A/B testing and gradual rollouts, audience targeting, and config tuning. Best practices: always ship sensible defaults, never store secrets (values are client-readable), and use realtime listeners when you need prompt changes.

---

## 24. Accurate Time in Android

The clock APIs differ: `System.currentTimeMillis()` is wall-clock epoch but **user-changeable and drifts**; `SystemClock.elapsedRealtime()` is **monotonic** time since boot (incl. sleep) for measuring durations; `uptimeMillis()` excludes deep sleep. Never trust `currentTimeMillis()` for security/business logic (token expiry, trials, ordering). Measure intervals with `elapsedRealtime()`. For accurate absolute time, fetch it from a **trusted source** (server `Date` header or NTP) once, compute an offset against `elapsedRealtime()`, and apply it locally. Store/transmit in UTC; inject a `Clock` for testability.

---

## 25. Query Optimization in SQLite

Biggest win: **indexes** on `WHERE`/`JOIN`/`ORDER BY` columns (but don't over-index — writes slow down). Select only needed columns (not `SELECT *`), use **`EXPLAIN QUERY PLAN`** to confirm index use vs full scan, and filter/paginate (`LIMIT`/keyset). Wrap batch writes in a **single transaction** (avoids per-write fsync), reuse compiled/parameterized statements, and avoid functions on columns in `WHERE` (defeats indexes). Enable **WAL mode** (Room default) for concurrent reads/writes, and run `ANALYZE`/`VACUUM` periodically.

---

## 26. WebSocket vs Socket.IO

**WebSocket** is a standardized transport **protocol** (RFC 6455): one persistent full-duplex TCP connection with raw frames — you build reconnection, acks, and rooms yourself. **Socket.IO** is a **library** with its own protocol layered on top, adding automatic reconnection, **long-polling fallback**, rooms/namespaces, event-based messaging with **ack callbacks**, and heartbeats. Key distinction: a raw WebSocket client **cannot** talk to a Socket.IO server — Socket.IO has its own handshake/framing. Use raw WebSocket for lightweight standards-based connections; Socket.IO when you want batteries-included features.

---

## 27. Symmetric vs Asymmetric Encryption

**Symmetric** uses one shared secret key (AES, ChaCha20) — very fast for bulk data, but has the **key distribution problem**. **Asymmetric** uses a public/private key pair (RSA, ECC, DH) — solves key distribution (anyone encrypts with your public key) and enables **digital signatures**, but is much slower and impractical for large payloads. In practice they **combine** as hybrid encryption (e.g. TLS, WhatsApp): use asymmetric to exchange a random symmetric session key, then symmetric for the actual bulk traffic.

---

## 28. SMS Retriever API in Android

Auto-read a one-time OTP **without the `READ_SMS` permission**, so no privacy/Play-policy issue. Flow: the app calls `SmsRetriever.getClient(ctx).startSmsRetriever()` (5-min window), sends the phone number to your server, and the server sends an SMS **specially formatted** with a `<#>` prefix and an **11-character app-hash suffix** (derived from package + signing cert). Google Play services matches the hash and delivers the message **only to your app** via a `BroadcastReceiver` (`SMS_RETRIEVED_ACTION`), which parses the code. The related **SMS User Consent API** shows a one-tap dialog when you don't control the SMS format.
