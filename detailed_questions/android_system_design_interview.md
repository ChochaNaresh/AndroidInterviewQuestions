# Android System Design — Interview Preparation Guide

Android system design interviews evaluate whether you can take a vague, open-ended prompt ("Design an image loading library", "Design WhatsApp") and drive it into a concrete, defensible architecture under time pressure. Unlike backend system design, the focus is on the **client**: memory and battery constraints, the threading model, caching layers, offline behaviour, the network stack, lifecycle correctness, and clean module boundaries. The backend is treated as a black box you call over the network — you should mention it, but the depth lives on the device.

This guide covers every design and concept question from the Outcome School / Amit Shekhar Android System Design list. For each **design** question it follows a repeatable structure: functional & non-functional requirements, high-level components, data model, key APIs, caching/threading strategy, and trade-offs, with Kotlin sketches where they clarify the design. For each **concept** question it gives a comprehensive comparison you can speak to confidently.

A reusable framework to open any design answer with:

1. **Clarify scope** — ask what's in/out (e.g. "single-image or batch? GIF support? disk cache size limit?").
2. **Requirements** — list functional and non-functional (latency, memory ceiling, offline, battery).
3. **High-level design** — name the components and draw the data flow.
4. **Deep-dive** — pick the 2–3 hardest parts (memory, threading, cache eviction) and go deep.
5. **Trade-offs** — call out what you optimised for and what you sacrificed.

---

## Table of Contents

**Design questions**

1. [Design an Image Loading Library](#1-design-an-image-loading-library)
2. [Design a File Downloader Library](#2-design-a-file-downloader-library)
3. [Design WhatsApp](#3-design-whatsapp)
4. [Design Instagram Stories](#4-design-instagram-stories)
5. [Design a Networking Library](#5-design-a-networking-library)
6. [Design Facebook Nearby Friends](#6-design-facebook-nearby-friends)
7. [Design a Caching Library](#7-design-a-caching-library)
8. [Location-Based App Design Problems](#8-location-based-app-design-problems)
9. [Offline-First App Architecture](#9-offline-first-app-architecture)
10. [Design an LRU Cache](#10-design-an-lru-cache)
11. [Design an Analytics Library](#11-design-an-analytics-library)
12. [Design a Logging Library](#12-design-a-logging-library)
13. [Design the Uber App](#13-design-the-uber-app)

**Concept questions**

14. [HTTP Request vs Long-Polling vs WebSocket vs SSE](#14-http-request-vs-long-polling-vs-websocket-vs-sse)
15. [How Voice and Video Calls Work](#15-how-voice-and-video-calls-work)
16. [Data Syncing on Unstable Networks](#16-data-syncing-on-unstable-networks)
17. [How "Where Is My Train" Works Without Internet](#17-how-where-is-my-train-works-without-internet)
18. [Database Normalization vs Denormalization](#18-database-normalization-vs-denormalization)
19. [Hash vs Encrypt vs Encode](#19-hash-vs-encrypt-vs-encode)
20. [Webhook vs Polling](#20-webhook-vs-polling)
21. [Options for Real-Time Updates in an Android App](#21-options-for-real-time-updates-in-an-android-app)
22. [Options for Network Optimization in a Mobile App](#22-options-for-network-optimization-in-a-mobile-app)
23. [Firebase Remote Config in Android](#23-firebase-remote-config-in-android)
24. [Accurate Time in Android](#24-accurate-time-in-android)
25. [Query Optimization in SQLite](#25-query-optimization-in-sqlite)
26. [WebSocket vs Socket.IO](#26-websocket-vs-socketio)
27. [Symmetric vs Asymmetric Encryption](#27-symmetric-vs-asymmetric-encryption)
28. [SMS Retriever API in Android](#28-sms-retriever-api-in-android)

---

## 1. Design an Image Loading Library

Build a library like Glide / Coil / Picasso: load images from a URL (or resource/file) into an `ImageView` efficiently, with caching, transformations, and correct behaviour during view recycling.

### Requirements

**Functional**
- Load image from URL, file, resource, URI into an `ImageView` (or as a `Bitmap`/`Drawable`).
- Placeholder while loading, error image on failure.
- Resize/transform (center-crop, circle-crop, rounded corners).
- Memory cache + disk cache.
- Cancel a request when the target view is recycled or detached.

**Non-functional**
- Must not cause `OutOfMemoryError` — bitmaps are the #1 OOM source on Android.
- Smooth scrolling in a `RecyclerView` (no jank on the main thread).
- Thread-safe; loads off the main thread, delivers on the main thread.
- Lifecycle-aware (don't deliver to a destroyed Activity/Fragment).

### High-level components

```text
request() ──> Request (url, target, transforms, placeholder)
                 │
                 ▼
         ┌──────────────────┐
         │  Engine          │  orchestrates the lookup chain
         └──────────────────┘
             │  1. ActiveResources (in-use bitmaps, weak refs)
             │  2. MemoryCache (LruCache of Bitmaps)
             │  3. DiskCache (LruDiskCache of encoded bytes)
             │  4. Network fetch (OkHttp)
             ▼
         Decode + downsample (BitmapFactory)
             ▼
         Transform (center-crop, etc.)
             ▼
         Deliver on main thread ──> ImageView
```

### The three-tier cache

1. **Active resources** — bitmaps currently displayed, held by weak references with reference counting. Prevents the same in-use bitmap from being evicted from the LruCache and re-decoded.
2. **Memory cache** — `LruCache<Key, Bitmap>` sized as a fraction of the app heap (e.g. 1/8 of `maxMemory`). Holds recently used but not currently displayed bitmaps.
3. **Disk cache** — encoded original (and/or transformed) bytes via `DiskLruCache`. Survives process death; sized in MB.

### Cache key

The key must encode everything that affects the decoded pixels: `url + targetWidth + targetHeight + transformations + decodeFormat`. Two `ImageView`s of different sizes loading the same URL get different memory-cache entries but can share the disk-cached original bytes.

### Memory: the hard part

Decoding a full-resolution image into an `ImageView` that's much smaller wastes memory. A 4000×3000 ARGB_8888 bitmap = 4000 × 3000 × 4 bytes ≈ **48 MB**. Downsample to the target size:

```kotlin
fun decodeSampled(data: ByteArray, reqW: Int, reqH: Int): Bitmap {
    val opts = BitmapFactory.Options().apply { inJustDecodeBounds = true }
    BitmapFactory.decodeByteArray(data, 0, data.size, opts) // reads dimensions only
    opts.inSampleSize = calcInSampleSize(opts.outWidth, opts.outHeight, reqW, reqH)
    opts.inJustDecodeBounds = false
    opts.inPreferredConfig = Bitmap.Config.RGB_565 // 2 bytes/px if no alpha needed
    return BitmapFactory.decodeByteArray(data, 0, data.size, opts)
}

fun calcInSampleSize(w: Int, h: Int, reqW: Int, reqH: Int): Int {
    var sample = 1
    while (w / (sample * 2) >= reqW && h / (sample * 2) >= reqH) sample *= 2
    return sample // power of two: decoder is fastest
}
```

**Bitmap pool** — reuse bitmap memory via `BitmapFactory.Options.inBitmap` so the GC doesn't churn. When a bitmap is no longer referenced, return its allocation to a pool keyed by size; the next decode of a compatible size reuses it. This is what keeps scrolling smooth — the dominant cost of jank is GC pauses from constant bitmap allocation.

### Threading

- A bounded thread pool for network + decode work (Glide separates "source" and "disk-cache" executors).
- `inJustDecodeBounds` and the actual decode run on a background thread.
- Delivery posts back to the main thread via a `Handler(Looper.getMainLooper())`.

### View recycling correctness

Tag the target view with the request. When a view is rebound in a `RecyclerView`, cancel the in-flight request for the previous URL so a slow response doesn't draw the wrong image into a recycled cell:

```kotlin
fun into(target: ImageView) {
    val tag = target.getTag(R.id.img_req) as? RequestFuture
    tag?.cancel() // cancel stale request on this recycled view
    val future = engine.load(request) { bitmap -> target.setImageBitmap(bitmap) }
    target.setTag(R.id.img_req, future)
}
```

### Lifecycle awareness

Bind requests to the host's lifecycle (Glide uses a headless `RequestManagerFragment`/lifecycle observer). Pause requests on `onStop`, resume on `onStart`, clear on `onDestroy` to avoid leaking the Activity and to avoid delivering to a dead view.

### API sketch

```kotlin
ImageLoader.with(context)
    .load("https://example.com/a.jpg")
    .placeholder(R.drawable.ph)
    .error(R.drawable.err)
    .transform(CircleCrop())
    .override(width, height)
    .into(imageView)
```

### Trade-offs
- `RGB_565` halves memory but loses alpha and color fidelity — fine for opaque photos, wrong for icons with transparency.
- Caching transformed bitmaps speeds redraws but multiplies disk usage; caching only originals saves space but re-transforms each time.
- Aggressive memory cache = fewer decodes but higher OOM risk on low-end devices; size it from `ActivityManager.getMemoryClass()`.

**📚 Reference:** https://outcomeschool.com/blog/android-image-loading-library-optimize-memory-usage, https://outcomeschool.com/blog/android-image-loading-library-use-bitmap-pool-for-responsive-ui, https://outcomeschool.com/blog/android-image-loading-library-solve-the-slow-loading-issue

---

## 2. Design a File Downloader Library

Build a PRDownloader-style library: download large files reliably with pause/resume, progress callbacks, and parallel/chunked downloads.

### Requirements

**Functional**
- Download a file from a URL to a destination path.
- Progress callbacks (bytes downloaded / total).
- Pause, resume, cancel.
- Resume after process death / network loss.
- Concurrent downloads with a configurable limit.

**Non-functional**
- Resilient to flaky networks (retry with backoff).
- Low memory (stream to disk, never hold the whole file in memory).
- Correct on a single source of truth for progress (avoid double-counting).

### Resume mechanism — HTTP Range requests

The core trick is the HTTP `Range` header. Persist how many bytes are already written; on resume, ask the server only for the rest:

```text
Range: bytes=1048576-      # request from byte 1,048,576 onward
```

The server replies `206 Partial Content` with `Content-Range`. If it returns `200`, it doesn't support ranges and you must restart from zero. Persist a small metadata record (`url`, `tempPath`, `totalBytes`, `downloadedBytes`, `eTag`) so resume survives app restarts. Validate with `ETag`/`Last-Modified` — if the remote file changed, discard the partial and restart.

### Components

```text
DownloadRequest ──> DownloadDispatcher (queue + thread pool, max concurrent)
                         │
                         ▼
                   DownloadTask (Runnable)
                     ├─ open HttpURLConnection / OkHttp with Range header
                     ├─ stream body -> RandomAccessFile (seek to offset)
                     ├─ emit progress (throttled)
                     └─ persist progress to DownloadModel (Room/SQLite)
```

### Streaming to disk

```kotlin
fun download(task: DownloadModel) {
    val conn = client.newCall(
        Request.Builder().url(task.url)
            .header("Range", "bytes=${task.downloaded}-").build()
    ).execute()
    val raf = RandomAccessFile(task.tempPath, "rw").apply { seek(task.downloaded) }
    conn.body!!.byteStream().use { input ->
        val buf = ByteArray(8 * 1024)
        var read: Int
        while (input.read(buf).also { read = it } != -1) {
            if (task.isCancelled) return
            raf.write(buf, 0, read)
            task.downloaded += read
            maybeEmitProgress(task)          // throttle to ~every 65 KB or 100 ms
            if (shouldCheckpoint()) db.save(task) // persist offset periodically
        }
    }
    rename(task.tempPath, task.finalPath)    // atomic move on success
}
```

### Progress throttling

Don't post a callback on every 8 KB read — that floods the main thread. Throttle by bytes (every ~65 KB) or by time (every ~100 ms / when the percentage changes). PRDownloader uses a sync threshold for both progress callbacks and DB checkpoints.

### Parallel / multi-part downloads

For a large file you can split it into N ranges, download them on N threads into the same file at different offsets (using `RandomAccessFile.seek`), and merge by simply having each writer own its byte region. Trade-off: more throughput on high-bandwidth links, more connections/overhead and complexity on mobile — often a single stream with resume is enough.

### Reliability
- Retry transient failures (timeouts, 5xx) with exponential backoff.
- Use `WorkManager` for downloads that must survive app death / continue in background, with a network constraint (`UNMETERED` for large files).
- Atomic finalize: download to `.tmp`, rename only on full success so consumers never see a half file.

### Trade-offs
- Multi-part is faster but not all servers/CDNs support ranges and it uses more battery/connections.
- Frequent DB checkpoints = better resume granularity but more I/O; balance the checkpoint interval.

**📚 Reference:** https://github.com/amitshekhariitbhu/PRDownloader

---

## 3. Design WhatsApp

A real-time 1:1 and group messaging app with delivery receipts, presence, media, and end-to-end encryption.

### Requirements

**Functional**
- 1:1 and group messaging.
- Delivery states: sent (✓), delivered (✓✓), read (blue ✓✓).
- Online/last-seen presence, typing indicators.
- Media (images, video, voice notes, documents).
- Offline send (queue) and offline read (local DB is source of truth).
- End-to-end encryption.

**Non-functional**
- Low latency delivery, scale to billions of users.
- Reliable ordering, exactly-once display (dedup).
- Battery- and bandwidth-efficient on mobile.

### Transport

A persistent **WebSocket** (WhatsApp historically used a customized XMPP) for bidirectional real-time messaging while the app is foregrounded/connected. When the socket isn't connected (app killed/backgrounded), the server delivers via **push (FCM)** a wake-up, and the client reconnects to pull queued messages.

### Client architecture (single source of truth = local DB)

```text
UI (Compose) ── observes ──> Local DB (Room: messages, chats, contacts)
                                  ▲
                                  │ writes immediately (optimistic)
       OutboxQueue ── WebSocket ──┤
                                  │ inbound messages
       FCM wake ──> reconnect ────┘
```

The UI **never** waits on the network. Sending writes the message to the local DB with `status = PENDING` and shows it instantly; a background sender drains the outbox over the socket and updates the status (`SENT → DELIVERED → READ`) as acks arrive.

### Data model (client)

```text
Message(id PK, chatId, senderId, body/mediaRef, timestamp,
        status [PENDING|SENT|DELIVERED|READ|FAILED], serverSeq)
Chat(id PK, type [1:1|group], lastMessageId, unreadCount, mutedUntil)
Contact(id PK, name, phone, publicKeyBundle, lastSeen, presence)
```

- `id` is a **client-generated UUID** so the message exists locally before the server assigns a `serverSeq`. This also enables idempotent dedup: if a message is delivered twice, the UUID collapses it.
- Ordering uses a server-assigned monotonic `serverSeq` per chat, not client clocks.

### Delivery receipts

Each message carries acks back through the same socket:
- Server receives → `SENT`.
- Recipient device receives → server relays a delivery ack → `DELIVERED`.
- Recipient opens the chat → read ack → `READ`.

### Group messaging

Server-side fan-out: the sender sends one message; the server replicates it to each member's queue. For E2E, WhatsApp uses the **Sender Keys** model — the sender encrypts once with a group session key distributed pairwise, avoiding N separate encryptions per message.

### End-to-end encryption (Signal Protocol)

- Each user has an identity key pair plus prekeys uploaded to the server.
- Sender establishes a shared session via **X3DH** key agreement, then uses the **Double Ratchet** for forward secrecy (a new message key per message).
- The server only ever sees ciphertext; it cannot read messages.
- See [Symmetric vs Asymmetric Encryption](#27-symmetric-vs-asymmetric-encryption) — the handshake is asymmetric, the message body is symmetric (AES).

### Media

Upload media to blob storage / CDN out-of-band; the message carries only a reference + encryption key + thumbnail. Recipients download lazily. This keeps the messaging path small and fast.

### Presence & typing

Lightweight ephemeral events over the socket (not persisted): `typing`, `online`, `last_seen`. Heavily rate-limited to save battery.

### Trade-offs
- Persistent socket = instant delivery but battery cost; mitigated by letting the OS kill it and relying on FCM to wake.
- Client-generated IDs add dedup complexity but enable true optimistic UI.
- E2E means no server-side search/backup of plaintext — backups are separately encrypted.

---

## 4. Design Instagram Stories

Ephemeral, full-screen, auto-advancing media (image/video) that expires after 24 hours, with a tray of who-has-stories at the top of the feed.

### Requirements

**Functional**
- Horizontal tray of users who posted stories; ring indicates unseen.
- Tap opens a full-screen viewer; auto-advance through a user's stories, then to the next user.
- Tap right/left to skip forward/back; long-press to pause; swipe down to dismiss.
- Segmented progress bar at the top.
- Mark seen; expire after 24h.

**Non-functional**
- **Instant** playback — no spinner between segments. Prefetch is the whole game.
- Smooth 60fps gestures.
- Bounded memory and disk.

### Components

```text
StoriesTrayViewModel ──> tray list (user, hasUnseen, thumbnailUrl)
StoryViewerActivity
  ├─ ViewPager2 (user-level)  → one page per user
  │    └─ inner segment controller (per-story progress + auto-advance)
  ├─ MediaPlayer pool (ExoPlayer for video, ImageLoader for images)
  ├─ Prefetcher (next user + next segment)
  └─ SeenRepository (local DB + sync to server)
```

### Prefetching — the key to "instant"

The reason Stories feel instant is aggressive prefetch:
- When the tray loads, prefetch the **first segment** of the first few users.
- While viewing user N's story, prefetch user N+1's first media and the current user's next segments.
- Use the image/video cache (LRU on disk) so already-seen media isn't refetched.

```kotlin
fun onSegmentShown(userIndex: Int, segmentIndex: Int) {
    prefetch(stories[userIndex].segments.getOrNull(segmentIndex + 1)) // next segment
    prefetch(stories.getOrNull(userIndex + 1)?.segments?.firstOrNull()) // next user
}
```

### Segment timing & progress

Each segment has a fixed duration (images ~5s, video = clip length). A single timer drives both the progress bar animation and auto-advance. Pause on long-press (cancel the timer, hold animation); resume restores remaining time.

```kotlin
class SegmentController(private val durations: List<Long>) {
    private var index = 0
    fun start(onTick: (Float) -> Unit, onComplete: () -> Unit) { /* ValueAnimator */ }
    fun pause(); fun resume()
    fun next() { if (++index < durations.size) start(...) else onUserComplete() }
    fun previous() { if (--index >= 0) start(...) else onUserPreviousNeeded() }
}
```

### Data model

```text
Story(userId, segments: List<Segment>, expiresAt)
Segment(id, mediaUrl, type [IMAGE|VIDEO], durationMs)
SeenState(segmentId, seenAt)   // local; synced so ring clears across devices
```

### Memory & player reuse
- Reuse a small pool of `ExoPlayer` instances (current, next) rather than creating one per segment — player creation is expensive.
- Release players in `onStop`; recreate on resume.
- Cap the disk media cache; honor expiry (drop expired media).

### Trade-offs
- Heavy prefetch wastes data if the user dismisses early — bound it to "next 1–2 users", and gate large video prefetch on Wi-Fi.
- Storing seen-state only locally is simpler but won't sync across devices; syncing adds a write path.

---

## 5. Design a Networking Library

A Retrofit/Volley-style HTTP client wrapper: declarative API definitions, async execution, JSON parsing, interceptors, caching, and error handling.

### Requirements

**Functional**
- GET/POST/PUT/DELETE, headers, query/path params, request/multipart body.
- Async with callbacks / coroutines / Flow.
- Pluggable serialization (JSON via Moshi/Gson/kotlinx).
- Interceptors (auth, logging, retry).
- Response & error model; cancellation.

**Non-functional**
- Thread-safe, connection pooling/reuse, gzip.
- Configurable timeouts, retries, caching.
- Testable (swap the transport).

### Layered design

```text
API interface (annotations) ─> ServiceProxy (dynamic Proxy builds Requests)
                                     │
                              RequestQueue / CallFactory
                                     │
                              Interceptor chain  ── auth ── logging ── cache ── retry
                                     │
                              Transport (OkHttp / HttpURLConnection)  ── ConnectionPool
                                     │
                              Converter (bytes <-> model)
                                     │
                              Dispatcher (thread pool) ─> deliver on main thread
```

### Interceptor chain (chain of responsibility)

Each interceptor can inspect/modify the request, proceed, then inspect/modify the response — the design pattern that makes the stack extensible:

```kotlin
interface Interceptor { fun intercept(chain: Chain): Response }

class AuthInterceptor(val tokens: TokenStore) : Interceptor {
    override fun intercept(chain: Chain): Response {
        val req = chain.request.newBuilder()
            .header("Authorization", "Bearer ${tokens.access}").build()
        var res = chain.proceed(req)
        if (res.code == 401) {                 // refresh + retry once
            val fresh = tokens.refreshBlocking()
            res = chain.proceed(req.newBuilder().header("Authorization", "Bearer $fresh").build())
        }
        return res
    }
}
```

### Threading & async

A bounded `Dispatcher` thread pool executes calls; results post back to the main thread. For modern Android, expose `suspend` functions backed by `suspendCancellableCoroutine` so cancellation propagates to the underlying call.

```kotlin
suspend fun <T> Call<T>.await(): T = suspendCancellableCoroutine { cont ->
    enqueue(object : Callback<T> {
        override fun onResponse(r: T) = cont.resume(r)
        override fun onFailure(e: Throwable) = cont.resumeWithException(e)
    })
    cont.invokeOnCancellation { cancel() }
}
```

### Caching

Honor HTTP caching semantics (`Cache-Control`, `ETag`, `Last-Modified`) with a disk response cache; conditional GETs (`If-None-Match`) save bandwidth when the server returns `304`. Allow a force-network / force-cache override per request.

### Error model

Map transport/HTTP/parse failures into a sealed result so callers handle every case:

```kotlin
sealed interface NetResult<out T> {
    data class Success<T>(val data: T, val fromCache: Boolean) : NetResult<T>
    data class HttpError(val code: Int, val body: String?) : NetResult<Nothing>
    data class NetworkError(val cause: IOException) : NetResult<Nothing>
    data class ParseError(val cause: Throwable) : NetResult<Nothing>
}
```

### Trade-offs
- Building on OkHttp (don't reinvent sockets/HTTP-2/connection pooling) vs raw `HttpURLConnection` — in practice you wrap OkHttp; the "library" is the ergonomic layer above it.
- Reflection/codegen for the API proxy: reflection is simpler, codegen (annotation processing) is faster and crash-safe at runtime.

---

## 6. Design Facebook Nearby Friends

Opt-in feature showing which friends are near you, with periodic location sharing and proximity notifications.

### Requirements

**Functional**
- Opt-in location sharing; granular controls (precise/approximate, duration).
- Show friends nearby on a list/map with approximate distance.
- Notify when a friend comes nearby.
- Stop sharing easily; privacy-first.

**Non-functional**
- **Battery efficiency is paramount** — naive continuous GPS drains the battery.
- Privacy: minimal precision, user consent, easy off-switch.
- Scale proximity queries on the backend (geohash/quadtree).

### Client: battery-aware location collection

- Use **Fused Location Provider** (`FusedLocationProviderClient`) with a balanced priority, not raw GPS.
- Use **Geofences** and **Activity Recognition** instead of polling: only sample frequently when the user is moving; back off when stationary.
- Batch and upload locations periodically (e.g. every few minutes) via `WorkManager`, not on every fix.
- Significant-location-change style triggers to wake only when the user has actually moved.

```kotlin
val request = LocationRequest.Builder(Priority.PRIORITY_BALANCED_POWER_ACCURACY, 5 * 60_000L)
    .setMinUpdateDistanceMeters(100f)   // only when moved 100m+
    .setWaitForAccurateLocation(false)
    .build()
```

### Backend proximity (mention briefly)

Server indexes the latest location of each user with a **geohash** or **quadtree**. To find nearby friends, query the user's geohash cell + 8 neighbors, intersect with the friend list, compute haversine distance, filter by radius. Push a notification when a friend enters the radius.

### Data model

```text
Client: SharingState(enabled, precision, expiresAt), FriendLocation(friendId, lat, lng, updatedAt, distance)
Server: UserLocation(userId, geohash, lat, lng, ts, ttl)  // TTL so stale locations vanish
```

### Privacy
- Approximate distance buckets ("within 1 mile") rather than exact coordinates by default.
- Auto-expire sharing (`expiresAt`); locations have a TTL server-side.
- Encrypt in transit; never share with non-friends.

### Trade-offs
- Higher update frequency = fresher "nearby" data but worse battery; balance with motion-triggered sampling.
- Showing precise vs bucketed distance trades usefulness against privacy.

---

## 7. Design a Caching Library

A general-purpose, multi-tier cache (memory + disk) with pluggable eviction, TTL, and serialization — usable for objects, responses, or bitmaps.

### Requirements

**Functional**
- `put(key, value)`, `get(key)`, `remove(key)`, `clear()`.
- Tiered: fast in-memory tier backed by a persistent disk tier.
- Eviction policy (LRU default, pluggable) and size limits (count or bytes).
- Per-entry TTL / expiry.
- Thread-safe; pluggable serializer for disk.

**Non-functional**
- O(1) get/put for the memory tier.
- Bounded memory and disk footprint.
- Survives process death (disk tier).

### Architecture

```text
Cache<K,V>
 ├─ MemoryCache (LruCache, size by count or byte weight)
 ├─ DiskCache   (DiskLruCache: journal + files, size in bytes)
 ├─ Serializer<V> (to/from bytes for disk)
 └─ Clock (injectable for testable TTL)
```

### Read/write flow (read-through, write-through)

```kotlin
suspend fun get(key: K): V? {
    memory.get(key)?.takeIf { !it.isExpired() }?.let { return it.value }   // tier 1
    disk.get(key)?.let { entry ->                                          // tier 2
        if (entry.isExpired()) { disk.remove(key); return null }
        memory.put(key, entry)            // promote to memory
        return entry.value
    }
    return null
}

suspend fun put(key: K, value: V, ttlMs: Long = Long.MAX_VALUE) {
    val entry = Entry(value, expiresAt = clock.now() + ttlMs)
    memory.put(key, entry)
    disk.put(key, serializer.encode(entry))   // write-through
}
```

### Eviction abstraction

```kotlin
interface EvictionPolicy<K> {
    fun recordAccess(key: K)
    fun recordInsert(key: K)
    fun evictionCandidate(): K?    // who to drop when over capacity
}
// LRU, LFU, FIFO implementations plug in here.
```

### Thread-safety
- `LruCache` is internally synchronized for the memory tier.
- Serialize disk writes (single-writer or per-key locks); reads can be concurrent.
- Use `ConcurrentHashMap` + striped locks for hot maps; or run all mutations on a single dispatcher.

### TTL strategies
- **Lazy expiry** — check on read, discard if expired (simple, default).
- **Active expiry** — a periodic sweep removes expired entries (frees memory proactively).

### Trade-offs
- Write-through (write memory + disk on every put) keeps tiers consistent but doubles write cost; write-back batches disk writes for throughput at the risk of loss on crash.
- LRU vs LFU: LRU is cheap and good for recency; LFU survives one-off scans better but needs frequency counters.

---

## 8. Location-Based App Design Problems

A catch-all category: ride-hailing, food delivery, geofenced reminders, fitness trackers. Interviewers probe **battery, accuracy, permissions, and background limits**.

### Core building blocks

| Concern | API / Approach |
|---|---|
| Get location | `FusedLocationProviderClient` (fuses GPS, Wi-Fi, cell, sensors) |
| Continuous updates | `LocationRequest` with a `Priority` (high accuracy ↔ low power) |
| Enter/exit a region | **Geofencing API** (OS-monitored, battery-cheap) |
| User moved? | **Activity Recognition** (still/walking/driving) — sample only when moving |
| Background work | `WorkManager` (deferrable) or foreground service (ongoing nav) |

### Permission model (modern Android)
- `ACCESS_COARSE_LOCATION` / `ACCESS_FINE_LOCATION` — request fine only when needed; the user can grant **approximate** only.
- `ACCESS_BACKGROUND_LOCATION` is a separate, later request and heavily scrutinized on Play.
- On Android 14+, foreground-service-type `location` is required for location work in a foreground service.

### Battery strategy (the recurring theme)
1. Choose the lowest priority that meets the use case (`BALANCED` ≈ ~100m, fine for "nearby"; `HIGH_ACCURACY` only for turn-by-turn).
2. Increase the update interval and minimum displacement.
3. Use geofences and activity recognition to wake only on meaningful change.
4. Batch uploads; don't hit the network per fix.
5. Stop updates when the screen is off / app backgrounded unless navigation is active.

### Example: geofenced reminder

```kotlin
val geofence = Geofence.Builder()
    .setRequestId("home")
    .setCircularRegion(lat, lng, 150f /* meters */)
    .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_ENTER)
    .setExpirationDuration(Geofence.NEVER_EXPIRE)
    .build()
geofencingClient.addGeofences(request, pendingIntent) // OS fires PendingIntent on entry
```

### Trade-offs
- Geofences are battery-cheap but limited in count (~100/app) and have latency/accuracy caveats.
- High-accuracy continuous GPS gives the best UX for navigation but is the worst for battery.

---

## 9. Offline-First App Architecture

Design an app that works fully offline and syncs when connectivity returns — the local database is the single source of truth.

### Principle: local DB is the source of truth

The UI reads from and writes to the **local database** (Room). The network is a sync mechanism, not the read path. This gives instant reads, instant (optimistic) writes, and graceful degradation offline.

```text
   UI  ⇄  ViewModel  ⇄  Repository  ⇄  Local DB (Room)  ── exposed as Flow
                                            ▲
                              SyncEngine ───┘
                                  │  pull (server → local)
                                  │  push (outbox → server)
                              WorkManager (network-constrained, retrying)
```

### Read path
Queries return `Flow<List<T>>` from Room; the UI observes and re-renders automatically whenever sync updates the DB. No spinner tied to the network.

### Write path (optimistic + outbox)
1. Write to the local DB immediately with a `pendingSync` flag (or enqueue a mutation in an **outbox** table).
2. Show the result instantly.
3. A `SyncWorker` drains the outbox to the server when online, then clears the flag.
4. On server confirmation, reconcile (replace temp IDs with server IDs).

```text
Outbox(localId, entityType, operation [CREATE|UPDATE|DELETE], payload, createdAt, retryCount)
Entity(... , syncState [SYNCED|PENDING|FAILED], updatedAt, serverVersion)
```

### Sync engine
- **Delta sync**: pull only changes since the last sync token / `updatedAt` cursor, not the whole dataset.
- **WorkManager** for reliable, constrained, retrying background sync (survives process death, respects Doze).
- Trigger sync on connectivity regained (`NetworkCallback`), on app foreground, and periodically.

### Conflict resolution
- **Last-write-wins** (timestamp/version) — simplest, can lose data.
- **Server-wins / client-wins** — by policy.
- **Merge / CRDTs** — for collaborative data; complex but lossless.
- Use a per-record `version` to detect conflicts (optimistic concurrency).

### Trade-offs
- Optimistic UI is great UX but needs rollback logic when the server rejects a write.
- LWW is simple but silently drops concurrent edits; merging is correct but costly.
- Local-first storage means you ship a migration story (Room migrations) and must handle schema drift.

**📚 Reference:** Android architecture guidance on offline-first apps (developer.android.com).

---

## 10. Design an LRU Cache

Implement a Least-Recently-Used cache with O(1) `get` and `put`.

### Approach: HashMap + Doubly-Linked List

- A **HashMap** gives O(1) key → node lookup.
- A **doubly-linked list** maintains recency order: most-recently-used at the head, least at the tail.
- On `get`/`put`, move the node to the head. On overflow, evict the tail.

```kotlin
class LRUCache<K, V>(private val capacity: Int) {
    private class Node<K, V>(val key: K, var value: V) {
        var prev: Node<K, V>? = null
        var next: Node<K, V>? = null
    }
    private val map = HashMap<K, Node<K, V>>()
    private val head = Node<K, V>(/*dummy*/ Any() as K, Any() as V) // sentinel
    private val tail = Node<K, V>(Any() as K, Any() as V)
    init { head.next = tail; tail.prev = head }

    private fun remove(n: Node<K, V>) { n.prev!!.next = n.next; n.next!!.prev = n.prev }
    private fun addFront(n: Node<K, V>) {
        n.next = head.next; n.prev = head; head.next!!.prev = n; head.next = n
    }

    @Synchronized fun get(key: K): V? {
        val n = map[key] ?: return null
        remove(n); addFront(n)            // mark most-recently-used
        return n.value
    }

    @Synchronized fun put(key: K, value: V) {
        map[key]?.let { it.value = value; remove(it); addFront(it); return }
        if (map.size == capacity) {
            val lru = tail.prev!!          // evict least-recently-used
            remove(lru); map.remove(lru.key)
        }
        val n = Node(key, value); addFront(n); map[key] = n
    }
}
```

(In real Android code you'd use `androidx.collection.LruCache`, which is `LinkedHashMap`-backed with `accessOrder=true` and a `sizeOf` hook; the question is testing whether you understand the underlying data structure.)

### Alternative: `LinkedHashMap`
`LinkedHashMap(capacity, 0.75f, accessOrder = true)` plus an overridden `removeEldestEntry` gives an LRU in a few lines — good to mention, but interviewers usually want the manual map + DLL version.

### Complexity
- `get`: O(1). `put`: O(1). Space: O(capacity).

### Trade-offs / follow-ups
- **Thread-safety**: synchronize, or use a concurrent variant; locking the whole cache serializes access — striped locks help under contention.
- **Eviction by size, not count**: weight each entry by bytes (bitmaps) and evict until under the byte budget.
- **TTL**: add expiry timestamps and treat expired entries as misses.

---

## 11. Design an Analytics Library

A library to track events (screen views, taps, custom events) and reliably batch-upload them to a backend, even across network loss and app restarts.

### Requirements

**Functional**
- `track(event, properties)`, `screen(name)`, `identify(userId)`.
- Attach common context (device, OS, app version, session id) automatically.
- Buffer events locally, batch-upload.
- Reliable delivery: survive offline + process death; no loss, minimal duplication.

**Non-functional**
- Near-zero impact on the calling thread (fire-and-forget).
- Battery/bandwidth efficient (batch, not per-event).
- Privacy/consent aware.

### Architecture

```text
track() ──(enqueue, non-blocking)──> EventQueue (in-memory ring)
                                         │ flush on size/time/background
                                         ▼
                                   Local DB (Room: pending events)
                                         │
                                   UploadWorker (WorkManager, batched, retrying)
                                         ▼
                                   Backend  ── ack ──> delete uploaded rows
```

### Reliability via persistence + batching
- Persist events to a local DB the moment they're tracked (don't lose them on a crash).
- Upload in **batches** triggered by: queue size threshold (e.g. 20 events), a time interval, or app going to background.
- Use **WorkManager** so uploads retry with backoff and survive process death; require network.
- Delete rows only after a successful server ack → at-least-once delivery (dedup server-side by event id).

```kotlin
fun track(name: String, props: Map<String, Any> = emptyMap()) {
    scope.launch(Dispatchers.IO) {                  // never block the caller
        dao.insert(Event(
            id = UUID.randomUUID().toString(),       // idempotency key for dedup
            name = name, props = props.toJson(),
            context = commonContext(), ts = clock.now()
        ))
        if (dao.count() >= BATCH_SIZE) scheduleUpload()
    }
}
```

### Common context & sessions
- Auto-attach device model, OS version, app version, locale, network type, and a **session id** (regenerated after N minutes of inactivity or on app restart).
- `identify` ties anonymous events to a user once they log in.

### Trade-offs
- Larger batches = fewer requests/better battery but more data lost if a batch can't be persisted; cap batch size.
- At-least-once + server dedup is simpler and more robust than trying for exactly-once on the client.
- Sampling high-volume events reduces cost at the price of fidelity.

---

## 12. Design a Logging Library

A Timber-style logging facade: tagged, leveled logs routed to pluggable sinks (Logcat in debug, crash reporter / file / remote in release), with PII safety and low overhead.

### Requirements

**Functional**
- Levels: VERBOSE/DEBUG/INFO/WARN/ERROR.
- Pluggable sinks ("trees"/"sinks"): Logcat, file, crash reporter, remote.
- Auto or explicit tags; lazy message construction.
- Per-build configuration (verbose in debug, errors-only in release).

**Non-functional**
- Minimal overhead on hot paths (no string building when the level is disabled).
- Thread-safe; never crash the app from logging.
- PII redaction / no secrets in logs.

### Architecture (facade + pluggable sinks)

```kotlin
interface LogSink { fun log(level: Int, tag: String?, msg: String, t: Throwable?) }

object Log {
    private val sinks = CopyOnWriteArrayList<LogSink>()
    fun plant(sink: LogSink) { sinks += sink }

    inline fun d(t: Throwable? = null, msg: () -> String) = dispatch(DEBUG, t, msg)
    inline fun e(t: Throwable? = null, msg: () -> String) = dispatch(ERROR, t, msg)

    @PublishedApi internal inline fun dispatch(level: Int, t: Throwable?, msg: () -> String) {
        if (sinks.isEmpty()) return
        val text = msg()                  // lambda → built only if there are sinks
        sinks.forEach { runCatching { it.log(level, autoTag(), text, t) } } // never throw
    }
}
```

- `msg: () -> String` is the key performance trick: the message string is only built if a sink will consume it — no wasted allocations on disabled levels.
- `runCatching` ensures a misbehaving sink never crashes the app.

### Sinks
- **LogcatSink** — debug builds only.
- **FileSink** — append to a rotating file (size/age-based rotation) on a background dispatcher; useful for field diagnostics and bug-report attachment.
- **CrashSink** — forward WARN/ERROR (and recent breadcrumbs) to Crashlytics/Sentry.
- **RemoteSink** — batch + upload selected logs (reuse the analytics upload machinery).

### Threading & PII
- File/remote sinks write off the main thread (single-thread dispatcher / queue) so I/O never blocks UI.
- Run messages through a redaction step (regex/format rules) to strip tokens, emails, card numbers before they reach persistent or remote sinks.

### Build configuration
```kotlin
if (BuildConfig.DEBUG) Log.plant(LogcatSink())
else { Log.plant(CrashSink()); Log.plant(FileSink(rotateBytes = 5_000_000)) }
```

### Trade-offs
- File logging aids debugging but costs I/O and disk; rotate and cap size.
- Remote logging is powerful for diagnostics but raises privacy/cost concerns — sample and redact.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-android-interview-activity-7413072570452717568-K2Pb

---

## 13. Design the Uber App

A ride-hailing app: riders request rides, the system matches a nearby driver, both see live location, with trip lifecycle, pricing, and payments.

### Requirements

**Functional**
- Rider: set pickup/destination, see ETA & fare estimate, request, watch the driver approach, pay, rate.
- Driver: go online, receive ride requests, navigate, complete trips.
- Live location tracking on both sides.
- Trip state machine; surge pricing; payments.

**Non-functional**
- Low-latency matching and location updates.
- Battery-efficient continuous tracking (driver app especially).
- Reliable on flaky mobile networks; resume trips after reconnect.

### Two clients, one backend

```text
Rider App ─┐                          ┌─ Driver App
           │   ── WebSocket/FCM ──>    │
           ▼                            ▼
       Backend: Matching, Trip Service, Location Service (geo-index),
                Pricing, Payments, Notifications
```

### Live location

- Driver app streams location (FusedLocationProvider, high accuracy while on a trip) over a **WebSocket** to the backend.
- Backend relays the driver's location to the matched rider in real time; rider's map animates the car.
- Backend maintains a **geospatial index** (geohash / S2 / quadtree) of online drivers for nearest-driver matching.

### Trip state machine

```text
REQUESTED → DRIVER_ASSIGNED → DRIVER_ARRIVING → DRIVER_ARRIVED
          → TRIP_STARTED → TRIP_ENDED → PAID → RATED
          (CANCELLED at most pre-start stages)
```

The client mirrors server-authoritative state; the server is the source of truth for the trip. The client persists current trip state locally so it can restore the in-trip UI after a crash/reconnect.

### Matching (backend, brief)
On a request, query the driver geo-index for nearby online drivers, rank by ETA/rating, dispatch the offer; first to accept wins; fall back to the next on timeout/decline.

### Mobile concerns (where the interview lives)
- **Battery**: drivers stream location for hours — use the highest interval/displacement that still feels live; foreground service with `location` type for ongoing trips.
- **Network resilience**: queue location updates and trip events; reconnect the socket with backoff; reconcile state on reconnect.
- **Maps**: render route polylines, animate marker interpolation between updates (don't teleport), draw ETAs.
- **Offline/edge**: persist trip so the rider/driver UI survives backgrounding and process death.

### Data model (client)
```text
Trip(id, state, driverId?, riderId, pickup, dest, fareEstimate, polyline, updatedAt)
DriverLocation(tripId, lat, lng, bearing, ts)
```

### Trade-offs
- WebSocket gives smooth live tracking but costs battery/connection; FCM wakes the app when the socket is down.
- Frequent location pushes = smoother car movement vs more bandwidth/battery; interpolate on the client to send fewer updates.

**📚 Reference:** https://github.com/amitshekhariitbhu/ridesharing-uber-lyft-app

---

## 14. HTTP Request vs Long-Polling vs WebSocket vs SSE

Four ways a client gets data from a server, ordered by how "real-time" they are.

| | HTTP Request | Long-Polling | WebSocket | SSE |
|---|---|---|---|---|
| Direction | Client→Server (req/resp) | Client→Server, server holds | Full-duplex, bidirectional | Server→Client (one-way) |
| Connection | Short-lived per request | Held open until data/timeout, then re-issued | Single persistent TCP | Single persistent HTTP |
| Protocol | HTTP | HTTP | `ws://`/`wss://` (upgraded from HTTP) | HTTP (`text/event-stream`) |
| Real-time | No (poll to refresh) | Near real-time | Real-time | Real-time (downstream) |
| Overhead | New connection/headers each call | Repeated reconnects + headers | Low after handshake | Low; auto-reconnect built-in |
| Browser/Android | Universal | Universal | Native (OkHttp on Android) | Native in browsers; libs on Android |

### HTTP Request (request/response)
Standard: client asks, server answers, connection closes (or is pooled). To get updates you must **poll** repeatedly. Simple and cacheable; not real-time; wasteful if you poll fast for rare changes.

### HTTP Long-Polling
Client sends a request; the server **holds it open** until data is available (or a timeout), then responds. The client immediately re-issues. Simulates push over plain HTTP. Better than tight polling, but each cycle still pays connection/header overhead and there are gaps between responses.

### WebSocket
After an HTTP upgrade handshake, you get a **persistent, full-duplex** TCP channel — both sides push messages anytime with minimal framing overhead. Ideal for chat, multiplayer, live trading, collaborative editing. Cost: you manage connection lifecycle, reconnection, heartbeats, and it's harder to scale/load-balance than stateless HTTP. On Android: OkHttp's `WebSocket`.

### Server-Sent Events (SSE)
A long-lived HTTP response of `text/event-stream` where the **server streams events** to the client. **One-way** (server→client) only, text-only, with built-in auto-reconnect and event IDs. Great for live feeds, notifications, dashboards. Lighter than WebSocket when you don't need client→server streaming. Native `EventSource` in browsers; on Android use OkHttp-SSE.

### When to choose
- Occasional fetch / CRUD → **HTTP request**.
- Need server→client updates, simple, text → **SSE**.
- Need bidirectional, low-latency → **WebSocket**.
- Stuck on plain HTTP infra → **Long-polling**.

**📚 Reference:** https://outcomeschool.com/blog/http-request-long-polling-websocket-sse, https://www.youtube.com/watch?v=8ksWRX4xV-s

---

## 15. How Voice and Video Calls Work

Real-time peer-to-peer audio/video, typically over **WebRTC**.

### Pipeline
1. **Capture** — mic + camera produce raw audio/video frames.
2. **Encode** — compress with codecs (audio: Opus; video: VP8/VP9/H.264/AV1). Compression is mandatory — raw video is far too large for networks.
3. **Transport** — send over **UDP** (via RTP/SRTP). UDP is preferred over TCP because for real-time media, **low latency beats reliability** — a late-but-correct retransmitted frame is useless; you'd rather drop it.
4. **Decode** — the peer decompresses frames.
5. **Render** — play audio, draw video.

### Signaling (out of band)
WebRTC needs a **signaling channel** (your own server, often WebSocket) to exchange:
- **SDP offers/answers** — describe codecs, media tracks, parameters.
- **ICE candidates** — possible network paths to reach each peer.
Signaling sets up the call; the media itself flows peer-to-peer.

### NAT traversal: STUN & TURN
Most devices are behind NAT/firewalls and have no public address.
- **STUN** server tells a peer its public IP:port so peers can attempt a direct connection.
- **TURN** server **relays** media when a direct P2P connection is impossible (strict NAT) — reliable but adds latency and server cost.
- **ICE** is the framework that gathers candidates and picks the best working path.

### Group calls
P2P mesh doesn't scale (N² streams). Use an **SFU** (Selective Forwarding Unit) — each client sends one stream up; the SFU forwards streams to participants. (MCU mixing is heavier and rarer.)

### Quality adaptation
WebRTC continuously adapts bitrate/resolution to measured bandwidth and loss, uses jitter buffers to smooth timing, and FEC/PLC to conceal lost packets — that's why calls degrade gracefully instead of freezing.

### Android specifics
- `WebRTC` library (`PeerConnectionFactory`, `PeerConnection`, `VideoTrack`/`AudioTrack`, `SurfaceViewRenderer`).
- Permissions: `RECORD_AUDIO`, `CAMERA`.
- Manage audio focus and routing (speaker/earpiece/Bluetooth).

**📚 Reference:** https://outcomeschool.com/blog/voice-and-video-call

---

## 16. Data Syncing on Unstable Networks

How to keep client and server data consistent when connectivity is intermittent.

### Strategy

1. **Local DB as source of truth** — UI reads/writes locally (Room); never block on the network. (See [Offline-First](#9-offline-first-app-architecture).)
2. **Outbox / mutation queue** — every local write also enqueues an operation to push later. Survives process death (persisted).
3. **WorkManager for sync** — deferrable, network-constrained, **retries with exponential backoff**, runs even after the app is killed and respects Doze. The right tool for "send when the network is back".
4. **Idempotency keys** — each mutation carries a client-generated UUID so retries don't create duplicates server-side.
5. **Delta sync** — pull only changes since the last cursor/`updatedAt`/sync token, not the full dataset — critical on slow links.
6. **Chunking & resumable transfers** — for large payloads/files use ranged/chunked uploads so a dropped connection resumes instead of restarting.
7. **Conflict resolution** — versioned records with last-write-wins, server-wins, or merge depending on the data.
8. **Connectivity awareness** — trigger sync on `ConnectivityManager.NetworkCallback` when the network returns; back off when on metered/poor links.
9. **Timeouts + retry/backoff + jitter** — short, sensible timeouts; exponential backoff with jitter to avoid thundering herds when many clients reconnect at once.

### Result
Writes feel instant (optimistic), nothing is lost, retries don't duplicate, and only minimal data crosses the wire — robust on 2G, in tunnels, and on flapping connections.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-interview-activity-7369017963754029056-G8Gv

---

## 17. How "Where Is My Train" Works Without Internet

The app shows live-ish train running status and location without an active data connection.

### Key techniques

1. **Offline bundled / downloaded data** — the full train schedule, station list, and route geometry are stored locally (SQLite). Knowing the timetable, the app can compute the expected position at any time without the network.
2. **GPS-based location, not cell-tower mapping** — the device's own GPS works without internet (GPS is satellite, not network). The app reads the phone's coordinates offline.
3. **Map GPS to the nearest station/route segment** — match the current GPS position against the locally stored route polyline to infer "between station A and B" and progress along the segment.
4. **Crowdsourced cell-tower / GPS heuristics** — historically these apps used the **cell tower ID** the phone is connected to (available without data) plus crowdsourced mappings of tower IDs to track positions, combined with the schedule, to estimate location even when GPS is weak inside a moving train.
5. **Dead reckoning against the timetable** — between location fixes, interpolate position using scheduled timings and average speeds.
6. **Sync when online** — when connectivity is available, refresh schedules and upload anonymized location data to improve the crowdsourced model.

### Takeaway
The "magic" is **precomputed offline data + on-device sensors (GPS + cell tower) + interpolation against the schedule** — no live server call is needed for the core experience.

**📚 Reference:** https://outcomeschool.substack.com/p/how-where-is-my-train-works-without

---

## 18. Database Normalization vs Denormalization

Two opposing strategies for structuring relational data.

### Normalization
Organize data to **minimize redundancy** by splitting into multiple related tables (per normal forms: 1NF, 2NF, 3NF…). Each fact is stored **once**; relationships use foreign keys.

- **Pros**: no duplicate data, smaller storage, no update anomalies (change a value in one place), strong integrity.
- **Cons**: reads need **JOINs** across tables, which can be slower; more complex queries.

### Denormalization
Deliberately **introduce redundancy** — duplicate or pre-join data into fewer tables — to optimize reads.

- **Pros**: faster reads (fewer/no JOINs), simpler read queries, great for read-heavy / reporting workloads.
- **Cons**: data duplication → larger storage, **update anomalies** (must update every copy), risk of inconsistency, heavier writes.

### Trade-off summary

| | Normalized | Denormalized |
|---|---|---|
| Redundancy | Minimal | Intentional |
| Read speed | Slower (JOINs) | Faster |
| Write speed | Faster, simpler | Slower (update all copies) |
| Storage | Less | More |
| Integrity | Strong | Easier to corrupt |
| Best for | Write-heavy, OLTP | Read-heavy, analytics, caches |

### On Android (Room/SQLite)
Local mobile DBs often lean **slightly denormalized** for fast UI reads (e.g. store a `lastMessageText` on the chat row instead of joining the messages table on every list render), accepting the small write cost of keeping it in sync. Normalize where correctness matters; denormalize the hot read paths.

**📚 Reference:** https://outcomeschool.com/blog/database-normalization-vs-denormalization

---

## 19. Hash vs Encrypt vs Encode

Three frequently-confused operations with completely different purposes.

| | Encode | Encrypt | Hash |
|---|---|---|---|
| Purpose | Data representation / transport | Confidentiality | Integrity / fingerprint |
| Reversible? | Yes (public scheme) | Yes (with the key) | **No** (one-way) |
| Needs a key? | No | Yes | No (HMAC uses a key) |
| Output | Different format, same info | Ciphertext | Fixed-length digest |
| Examples | Base64, URL-encoding, UTF-8 | AES, RSA, ChaCha20 | SHA-256, bcrypt, Argon2 |

### Encode
Transform data into another **format** for safe transport/storage — **not** for security. Anyone can decode it; there's no secret. Example: Base64-encoding a binary blob to put it in JSON. **Never** "encode for security."

### Encrypt
Transform plaintext into ciphertext using a **key** so only key-holders can read it. **Two-way** — decryptable with the right key. Use for confidentiality: messages, stored credentials, tokens at rest/in transit. (Symmetric vs asymmetric → [Q27](#27-symmetric-vs-asymmetric-encryption).)

### Hash
A **one-way** function mapping input to a fixed-length digest. You cannot reverse it. Use for integrity checks, deduplication, and **password storage** — store the hash, never the password; on login, hash the input and compare. For passwords use **slow, salted** hashes (bcrypt/scrypt/Argon2), not fast SHA-256, to resist brute force. **HMAC** adds a key to a hash for authenticated integrity.

### One-liner
- **Encode** to be *read by machines* (format).
- **Encrypt** to keep data *secret* (reversible with a key).
- **Hash** to *verify* data without storing it (irreversible fingerprint).

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-activity-7249769459790323712-ZF0O

---

## 20. Webhook vs Polling

Two ways for system A to learn about events in system B.

### Polling
Client **repeatedly asks** "any new data?" on a schedule.
- **Pros**: simple, client controls timing, no inbound endpoint needed, works behind NAT/firewalls.
- **Cons**: wasteful (most polls return nothing), latency between poll interval, server load scales with poll frequency × clients. Tighter interval = fresher data but more waste.

### Webhook ("push")
Server **calls the client** (an HTTP callback URL the client registered) the moment an event happens.
- **Pros**: real-time, efficient (no wasted requests), server-driven.
- **Cons**: client must expose a reachable public endpoint, handle security (verify signatures), and absorb retries/duplicates; harder for mobile clients which lack a stable public URL.

### Comparison

| | Polling | Webhook |
|---|---|---|
| Initiator | Client | Server |
| Latency | Up to the poll interval | Near-instant |
| Efficiency | Low (empty polls) | High |
| Infra | None special | Public callback endpoint |
| Reliability | Client retries naturally | Server must retry; needs idempotency |

### On mobile
Phones can't host webhooks reliably (no stable public URL, OS background limits). So the mobile equivalent of a webhook is **push notifications (FCM)** — the server pushes via Google's infrastructure, which then wakes the app. Use polling for non-urgent refreshes; FCM-push for real-time. (See [Q21](#21-options-for-real-time-updates-in-an-android-app).)

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_softwareengineer-activity-7265962230553223168-fsPx

---

## 21. Options for Real-Time Updates in an Android App

Ways to get fresh data to an Android client, from cheapest to most real-time.

1. **Polling (short)** — periodic `GET` on a timer. Simple, works everywhere; wasteful, latency = interval. Good for low-urgency dashboards.
2. **Long-polling** — server holds the request until data arrives. Near real-time over plain HTTP; reconnect overhead. (See [Q14](#14-http-request-vs-long-polling-vs-websocket-vs-sse).)
3. **Server-Sent Events (SSE)** — one-way server→client stream over HTTP, auto-reconnect. Great for feeds/notifications you only consume.
4. **WebSocket** — full-duplex persistent connection. Best for chat, live collaboration, trading, presence. Costs battery + connection management.
5. **FCM (Firebase Cloud Messaging) push** — server pushes through Google's infra, which wakes the app even when killed/backgrounded. The standard for notifications and for **waking** an app to then sync. Subject to throttling; not guaranteed-instant for high-frequency streams.
6. **Firebase Realtime Database / Firestore listeners** — managed real-time sync with offline support; the SDK keeps a socket and pushes snapshot changes. Fast to build; vendor lock-in and cost at scale.
7. **gRPC streaming** — bidirectional streams over HTTP/2; efficient binary, good for typed real-time APIs.

### Choosing on mobile
- App backgrounded / killed and you need delivery → **FCM** (often FCM wakes the app, then it opens a socket or pulls a delta).
- App foregrounded, bidirectional, low-latency → **WebSocket**.
- One-way live feed → **SSE**.
- Low urgency → **polling**.
Battery, OS background limits (Doze, background execution limits), and connection cost dominate the decision on Android.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_androiddev-android-activity-7287460425212866560-dJNB

---

## 22. Options for Network Optimization in a Mobile App

Techniques to make mobile networking faster, cheaper, and more battery-efficient.

### Payload
- **Compression** — gzip/Brotli request & response bodies.
- **Efficient formats** — Protocol Buffers / FlatBuffers over verbose JSON for hot paths.
- **Pagination & field selection** — request only what's needed (GraphQL/sparse fieldsets); avoid over-fetching.
- **Image optimization** — server-side resizing, WebP/AVIF, correct resolution per device, thumbnails first.

### Caching & avoiding requests
- **HTTP caching** — `Cache-Control`, `ETag`/`If-None-Match`, `Last-Modified` → `304 Not Modified` saves the body.
- **Local cache / offline-first** — serve from Room/disk; sync deltas only.
- **Deduplicate** in-flight identical requests.

### Connection
- **Connection reuse / keep-alive & HTTP/2 multiplexing** — avoid repeated TLS handshakes; one connection, many streams.
- **HTTP/3 (QUIC)** — faster setup, better on lossy mobile links.
- **CDN** — serve static assets from the edge.
- **DNS / TLS optimizations** — session resumption, fewer round trips.

### Scheduling & battery
- **Batch requests** — coalesce many small calls; align with radio wake windows.
- **WorkManager / JobScheduler** — defer non-urgent work to good network/charging windows; batch to avoid waking the radio repeatedly (the radio staying on is a major battery drain).
- **Prefetch** predictively on Wi-Fi; gate heavy transfers on `UNMETERED`.
- **Retry with exponential backoff + jitter**; honor timeouts.

### Resilience
- Graceful degradation, optimistic UI, resumable/chunked uploads & downloads on flaky links.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7287470994447900672-XYM9

---

## 23. Firebase Remote Config in Android

A cloud service to change app behavior and appearance **without publishing an update**.

### What it is
A cloud-stored set of **key–value parameters**. You define defaults in the app; the server can override them. The app fetches and applies new values at runtime, so you can tune behavior remotely.

### Use cases
- **Feature flags** — turn features on/off remotely; kill-switch a broken feature.
- **A/B testing** & gradual rollouts (integrates with Firebase A/B Testing / Analytics).
- **Targeting** — different values by app version, country, language, user audience, or random percentage.
- **Config tuning** — endpoints, thresholds, UI text/colors, refresh intervals.

### How it works
1. Set **in-app defaults** (used offline / before first fetch).
2. The SDK **fetches** values from the server (cached locally; throttled — production `minimumFetchInterval` typically ~12h, lower in debug).
3. **Activate** fetched values to make them live.
4. Read with `getString/getBoolean/getLong/getDouble`.

```kotlin
val remoteConfig = Firebase.remoteConfig
remoteConfig.setConfigSettingsAsync(
    remoteConfigSettings { minimumFetchIntervalInSeconds = 3600 }
)
remoteConfig.setDefaultsAsync(R.xml.remote_config_defaults)

remoteConfig.fetchAndActivate().addOnCompleteListener { task ->
    val newFeatureEnabled = remoteConfig.getBoolean("new_checkout_enabled")
    if (newFeatureEnabled) showNewCheckout()
}
```

### Best practices
- Always ship sensible defaults (the app must work before/without a fetch).
- Don't store secrets — values are client-readable.
- Fetch on app start/foreground; activate on next launch or immediately depending on UX.
- Use realtime Remote Config listeners to react to changes promptly when needed.

### Trade-offs
- Remote control without store review is powerful but values are public and add a network/SDK dependency; throttling means changes aren't instant unless you use realtime updates.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineering-tech-activity-7288408601944043520-pHZX

---

## 24. Accurate Time in Android

Getting trustworthy time is harder than it looks because the user (and clock drift) can change device time.

### The clock APIs

| API | Measures | Resets on reboot? | User-changeable? | Use for |
|---|---|---|---|---|
| `System.currentTimeMillis()` | Wall-clock (epoch, UTC) | No (persisted) | **Yes** | Display dates, timestamps to send to server |
| `SystemClock.elapsedRealtime()` | Time since boot (incl. deep sleep) | **Yes** | No | Measuring durations/intervals |
| `SystemClock.uptimeMillis()` | Time since boot (excl. deep sleep) | Yes | No | Timing while awake (animations) |

### The core lesson
- **Never trust `System.currentTimeMillis()` for security/business logic** (token expiry, "free trial ended", event ordering) — the user can change the device clock, and it drifts.
- **Use `elapsedRealtime()` for measuring intervals** — it's monotonic and unaffected by clock changes (e.g. "show this for 30 seconds", rate limiting).
- **For accurate absolute time, get it from a trusted source**: the server (read `Date` header / an API timestamp) or an **NTP** server, then compute an offset against `elapsedRealtime()` and apply that offset locally. This gives accurate, tamper-resistant wall-clock time without polling NTP constantly.

```kotlin
// Sync once: serverTime obtained from a trusted server response
val offset = serverEpochMs - SystemClock.elapsedRealtime()
fun trustedNow(): Long = SystemClock.elapsedRealtime() + offset   // accurate epoch
```

### Other notes
- Use `Instant`/`Clock` (java.time) for testable time; inject a `Clock` so tests aren't flaky.
- Account for time zones with the user's zone, but store/transmit in UTC.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7298341712634990592-tgyR

---

## 25. Query Optimization in SQLite

How to make SQLite queries fast on Android (Room sits on top of SQLite).

### Techniques

1. **Indexes** — create indexes on columns used in `WHERE`, `JOIN`, and `ORDER BY`. The single biggest win for read performance. In Room: `@Entity(indices = [Index("userId")])` or `@ColumnInfo(index = true)`. Beware over-indexing — each index slows writes and costs space.

2. **Select only needed columns** — `SELECT col1, col2` not `SELECT *`; fetching unused columns (especially BLOBs) wastes I/O and memory. In Room, project into a small POJO.

3. **Use `EXPLAIN QUERY PLAN`** — inspect whether a query uses an index ("SEARCH … USING INDEX") or does a full table scan ("SCAN"). Optimize the scans.

4. **Filter and paginate** — add `WHERE` to limit rows; use `LIMIT`/`OFFSET` or keyset pagination (Paging 3) instead of loading everything.

5. **Transactions for batch writes** — wrap many inserts/updates in a single transaction. Without it, each write is its own transaction (its own fsync), which is dramatically slower. `@Transaction` in Room or `db.runInTransaction { }`.

6. **Compiled / parameterized statements** — reuse prepared statements (`SupportSQLiteStatement`); Room does this for you and it avoids re-parsing SQL and SQL injection.

7. **Avoid expensive operations in WHERE** — functions on a column (`WHERE lower(name)=…`) prevent index use; store a normalized column or use a covering index.

8. **WAL mode** — Write-Ahead Logging (`PRAGMA journal_mode=WAL`, Room's default) lets reads and writes happen concurrently, improving throughput.

9. **VACUUM / ANALYZE** — `ANALYZE` updates statistics the planner uses; `VACUUM` reclaims space and defragments occasionally.

10. **Right data types & normalization balance** — store dates as integers, denormalize hot read paths where joins are the bottleneck (see [Q18](#18-database-normalization-vs-denormalization)).

```sql
EXPLAIN QUERY PLAN
SELECT id, title FROM message WHERE chatId = ? ORDER BY ts DESC LIMIT 50;
-- Want: SEARCH message USING INDEX index_message_chatId (chatId=?)
```

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_outcomeschool-softwareengineer-tech-activity-7299289876846284801-3gfH

---

## 26. WebSocket vs Socket.IO

A common confusion — one is a protocol, the other is a library on top of it.

### WebSocket
A **standardized transport protocol** (RFC 6455) providing a single, persistent, full-duplex TCP connection between client and server, established via an HTTP upgrade handshake. It's low-level: you get raw message frames and must build everything else (reconnection, acknowledgements, rooms, fallbacks) yourself.

### Socket.IO
A **library/framework** (with its own wire protocol layered on top of WebSocket) that adds developer conveniences:
- **Automatic reconnection** with backoff.
- **Fallback transport** (HTTP long-polling) when WebSocket isn't available — so it connects through restrictive proxies/firewalls.
- **Rooms and namespaces** for grouping/broadcasting.
- **Event-based messaging** with **acknowledgement callbacks**.
- **Heartbeats** (ping/pong) for liveness.

### Key distinction
A raw WebSocket client **cannot** talk to a Socket.IO server (and vice versa) directly — Socket.IO has its own handshake and framing protocol on top of WebSocket. You need a Socket.IO client to connect to a Socket.IO server.

### Comparison

| | WebSocket | Socket.IO |
|---|---|---|
| Type | Protocol (standard) | Library + custom protocol |
| Transport | WebSocket only | WebSocket, with long-polling fallback |
| Reconnection | Manual | Built-in |
| Rooms/broadcast | Manual | Built-in |
| Acks | Manual | Built-in |
| Interop | Standard, any WS client | Needs a Socket.IO client |
| Overhead | Minimal | Slightly more (extra framing) |

### When to use which
- Need a lightweight, standards-based connection and you'll handle reconnection yourself → **WebSocket** (OkHttp on Android).
- Want batteries-included reconnection, fallbacks, rooms, acks → **Socket.IO** (use the official `socket.io-client-java`/Android client, and a Socket.IO server).

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_outcomeschool-softwareengineer-tech-activity-7371027069046030336-OcIC

---

## 27. Symmetric vs Asymmetric Encryption

Two families of encryption, often combined in practice.

### Symmetric encryption
**One shared secret key** encrypts and decrypts.
- **Examples**: AES, ChaCha20, 3DES.
- **Pros**: very fast, efficient for large data.
- **Cons**: the **key distribution problem** — both parties need the same key, and getting it to the other side securely is hard.

### Asymmetric encryption (public-key)
A **key pair**: a **public key** (shareable) and a **private key** (secret). Data encrypted with the public key can only be decrypted with the private key (and vice versa for signatures).
- **Examples**: RSA, ECC (Elliptic Curve), Diffie-Hellman (key exchange).
- **Pros**: solves key distribution — anyone can encrypt to you using your public key; enables **digital signatures** (sign with private, verify with public) for authenticity/non-repudiation.
- **Cons**: much **slower**, impractical for large payloads.

### Comparison

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared secret | Public + private pair |
| Speed | Fast | Slow |
| Key distribution | Hard (shared secret) | Easy (public key) |
| Good for | Bulk data | Key exchange, signatures, small data |
| Examples | AES, ChaCha20 | RSA, ECC, DH |

### How they combine (hybrid encryption — e.g. TLS/HTTPS)
The best of both: use **asymmetric** to securely exchange a random **symmetric session key**, then use the fast **symmetric** key to encrypt the actual data. This is exactly how TLS, WhatsApp, and most secure systems work — asymmetric for the handshake/key-agreement, symmetric for the bulk traffic.

**📚 Reference:** https://www.linkedin.com/posts/amit-shekhar-iitbhu_symmetric-vs-asymmetric-encryption-activity-7308873320894996481-a43E

---

## 28. SMS Retriever API in Android

Read a one-time SMS verification code automatically **without requesting the `READ_SMS` permission**.

### The problem it solves
OTP autofill traditionally required `READ_SMS`, a sensitive permission Google restricts (and which lets you read *all* the user's SMS). The **SMS Retriever API** lets you receive just your own verification message — no permission, no privacy concern, no Play policy issue.

### How it works
1. **App** starts the SMS Retriever client and begins listening:
   ```kotlin
   SmsRetriever.getClient(context).startSmsRetriever()  // 5-minute window, no permission
   ```
2. **App** sends the user's phone number to your **server**.
3. **Server** sends an SMS that includes the OTP **and is specially formatted** for the SMS Retriever API:
   - Begins with `<#>` (or contains it),
   - Ends with an **11-character app hash string** that uniquely identifies your app,
   - Is ≤ 140 bytes, contains a one-time code.
   Example:
   ```text
   <#> Your code is 123456
   FA+9qCX9VSu
   ```
4. **Google Play services** matches the app hash, extracts the message, and delivers it to **your app only** via a broadcast — Android verifies it's destined for your app, so no permission and no access to other messages.
5. **App** receives it in a `BroadcastReceiver` (`SmsRetriever.SMS_RETRIEVED_ACTION`), parses the code, and fills the field.

```kotlin
class SmsBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(ctx: Context, intent: Intent) {
        if (intent.action != SmsRetriever.SMS_RETRIEVED_ACTION) return
        val extras = intent.extras ?: return
        val status = extras.get(SmsRetriever.EXTRA_STATUS) as Status
        if (status.statusCode == CommonStatusCodes.SUCCESS) {
            val message = extras.getString(SmsRetriever.EXTRA_SMS_MESSAGE)
            val otp = Regex("\\d{6}").find(message ?: "")?.value
            // fill OTP field
        }
    }
}
```

### App hash string
An 11-character hash derived from your app's package name and signing certificate. Generate it (Google provides `AppSignatureHelper`) and include it as the SMS suffix. It's safe to expose and ensures only your app receives the message.

### Related: SMS User Consent API
A variant where the OS shows a one-tap consent dialog and then hands your app a single matching message — useful when you don't control the SMS format (no app hash required), at the cost of one user tap.

### Why it matters in interviews
It demonstrates secure, privacy-respecting OTP autofill that complies with Play's restricted-permission policy — the modern, correct way to do OTP autoread on Android.

**📚 Reference:** https://www.linkedin.com/posts/outcomeschool_sms-retriever-api-in-android-activity-7309171578796224513-YRab
