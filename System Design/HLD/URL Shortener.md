# Designing a URL Shortener

## 1. Overview and Goals

A URL shortener takes a long URL (`https://www.example.com/some/very/long/path?with=query&params=all&over=it`) and produces a short alias (`https://sho.rt/aZ3xQ`) that, when visited, redirects the browser to the original URL. The service sits on the critical path of two very different workloads at once: an infrequent **write path** (someone submits a long URL and gets a short code back) and an overwhelmingly frequent **read path** (someone clicks a short link and must land on the original destination, fast).

Design goals for a production-grade URL shortener:

- **Uniqueness** — every short code maps to exactly one long URL, and the mapping never silently changes underneath a user who has already shared the link.
- **Availability on the read path** — a broken redirect is a broken link everywhere it was ever shared (social posts, printed materials, emails already sent); the redirect service must not become a single point of failure.
- **Low latency redirects** — a click is a real person waiting; redirection has to feel instantaneous, which pushes hard toward caching and away from anything synchronous on the hot path.
- **Short, unpredictable codes** — the code should be as short as usefully possible, and should not let a stranger enumerate or guess other users' links.
- **Scalability skewed toward reads** — the system is written to occasionally and read constantly, so every architectural choice (cache, database, redirect type) has to be evaluated against a read:write ratio that is routinely 100:1 or higher, not a 1:1 assumption.

These goals conflict with each other, and the conflicts are the substance of the design:

- A short code that's easy to generate without any coordination (a hash of the URL) tends to be either collision-prone or no shorter than the thing it's replacing; a code that's guaranteed unique and compact (a counter) needs some coordination to hand out.
- Caching aggressively to make reads fast fights with wanting real-time analytics and the ability to expire or delete a link — a cached redirect that lives in a browser or CDN can outlive the record it points to.
- Making codes unpredictable (to stop enumeration) fights with making them as short as possible (a purely sequential counter is the shortest possible encoding, but it's also the most guessable).

## 2. Requirements

**Functional requirements:**

- `shorten(long_url, custom_alias?, expiration?, user?) -> short_url` — given a long URL, return a short code. Optionally accept a user-chosen alias and an expiration date.
- `redirect(short_code) -> HTTP 301/302 to long_url` — given a short code, redirect the caller to the original URL.
- `delete(short_code)` / `update expiration` — an owner can deactivate or extend a link.
- (Common extensions, not core) click analytics per link, QR code generation, per-user link management, custom domains.

**Non-functional requirements:**

- **Scale.** A service like Bitly-scale handles hundreds of millions of new links a month and tens of billions of redirects over the same period — the read path dwarfs the write path by one to two orders of magnitude.
- **Read-heavy skew.** A 100:1 (sometimes 200:1+) read:write ratio is the standard assumption; every caching and database decision below is justified by this number, not an arbitrary default.
- **High availability, especially for reads.** A redirect failing is worse than a shorten-request failing — the write path can degrade gracefully (retry later), but an already-published short link failing to redirect is a dead link everywhere it was shared.
- **Low latency.** Redirect responses should be on the order of tens of milliseconds — this is what pushes the design toward an in-memory cache in front of the database rather than a database read on every click.
- **Short codes.** 6–8 characters is the usual practical target (see §4 for why).
- **Not (necessarily) sequential/guessable.** Monotonic ordering or sortability of codes is *not* a goal here — a predictable, enumerable short code is a security and privacy liability (see §9), so unlike many ID-generation problems that want IDs to sort by creation time, this system deliberately avoids that property.

## 3. Capacity Estimation

Assume 500 million new short URLs created per month, with a 100:1 read:write ratio.

**Writes:** 500,000,000 / (30 × 24 × 3600) ≈ **~200 URLs/sec** average write throughput. Peak traffic (launches, viral spikes) should be provisioned for several times this — plan for ~1,000–2,000 writes/sec at peak.

**Reads:** 100 × 200 = **~20,000 redirects/sec** average, and this is the number that actually sizes the cache and the redirect fleet, not the write number.

**Storage:** at 500M new links/month, retained for 5 years, that's 500M × 12 × 5 = 30 billion records. At roughly 500 bytes per record (short code, long URL, metadata, timestamps), that's **~15 TB** of mapping data over 5 years — small enough that storage cost is not the binding constraint; read throughput and cache hit rate are.

**Cache sizing:** traffic to shortened links is heavily skewed — a small fraction of links (often cited as the top ~20%) account for the large majority (~80%) of clicks, a classic power-law/Zipfian access pattern. This is precisely why an LRU cache holding a comparatively small working set (a few GB, sized to the hot 20%) in front of the database can realistically absorb ~95%+ of read traffic, letting the database see only a fraction of the 20,000 reads/sec above.

## 4. Short Code Design

**Alphabet.** Use **Base62**: `[0-9a-zA-Z]`, 62 symbols. It avoids URL-encoding issues that punctuation or `+`/`/` (Base64's extra characters) would introduce, and it's case-sensitive, which is what keeps codes short.

**Length vs. capacity:**

| Code length | Total codes (62^n) |
|---|---|
| 5 | ~916 million |
| 6 | ~56.8 billion |
| 7 | ~3.5 trillion |
| 8 | ~218 trillion |

Against the ~30 billion records estimated in §3, 7 characters gives comfortable headroom (3.5 trillion) for years of growth without a length migration — a fixed-width code is far easier to keep stable than one that grows over time, since a length change would need every existing short URL to remain valid under the old length while new ones use the new one.

**How the code is actually produced — three real strategies, each trading coordination against structure differently:**

1. **Hash the long URL, then Base62-encode a prefix of the hash (e.g., MD5 or SHA-256 truncated to 7 characters).**
   - *Upside:* stateless, deterministic — the same input always gives the same code, and no coordination or counter is needed at all.
   - *Downside:* truncating a cryptographic hash to a short prefix creates real collision risk — with only 7 Base62 characters (~3.5 trillion possible values), the birthday-paradox effect means collisions become likely long before every value has been used, so two different long URLs can end up hashing to the same short prefix. Every write must check the datastore for that code already existing and, on collision, either re-hash with a salt/counter appended or fall back to a different scheme. This turns an ostensibly "no-coordination" scheme into one with an unavoidable read-before-write check on every insert.
   - *Worked example:* take `https://www.example.com/blog/2026/09/27/how-to-design-a-scalable-url-shortener-service` and run it through MD5, giving the 128-bit hex digest `54bc7a9a47d098894c178d75b7decf97`. Rather than truncating the hex string itself (which would need to be sized carefully to line up with Base62's bit width), encode the *entire* digest as one long Base62 string — `2zTupLasgkmbOg4c1JHeoT` (22 characters, since 128 bits needs roughly that many Base62 digits to represent in full) — and take just its first 7 characters as the short code: `2zTupLa`. This sidesteps the earlier bit-counting question entirely: the full digest is converted losslessly to Base62 first, and truncation happens on the Base62 string itself, at exactly the length we want the final code to be. The write path then does `SELECT 1 FROM mapping WHERE short_code = '2zTupLa'` — if no row exists, insert `(2zTupLa → that URL)` and return it to the caller.
     Now suppose a second, unrelated URL — say `https://blog.another-site.net/2026/announcing-our-new-pricing-plans` — is submitted, and (hypothetically, for the sake of illustrating the collision path — its real MD5 actually encodes to a different prefix, `TU71Eay`) its first 7 Base62 characters happen to also land on `2zTupLa`. The uniqueness check above now finds that code already occupied by the first URL, so the service can't simply insert the second mapping — it falls back to a collision-resolution step: for example, appending a fixed salt or a small incrementing counter to the second URL before hashing again (`hash(url + ":1")`) and re-deriving a new code from that result. This retry-with-salt loop is exactly the "unavoidable read-before-write check" cost called out above: cheap the overwhelming majority of the time (one lookup, no collision), but the code has to handle the collision branch correctly, not just assume it away, because with enough URLs sharing a 7-character (~3.5 trillion-value) code space, it *will* eventually happen.

2. **A global counter, Base62-encoded.** A monotonically increasing counter issues a numeric ID (1, 2, 3, …); encode that integer directly in Base62 to get the short code. This guarantees uniqueness by construction — no collision check is ever needed, since two different counter values can never encode to the same string. "Global counter" describes the *logical* guarantee (every value handed out is unique, system-wide) — but *how* that counter is actually implemented is where the single-point-of-failure and distributed-vs-not questions live, and there are several real options with very different answers:

   - **A single database's `AUTO_INCREMENT` column, called on every write.** This is the naive version, and it genuinely is a single point of failure: every short-URL creation, everywhere, funnels through one database instance's one counter. If that instance is down, no new short URLs can be created anywhere, and its write throughput (bounded by that one machine) is the hard ceiling on the whole system's write capacity. It is not distributed at all — it's exactly the "ticket server" pattern, just without the fix described next.
   - **A ticket server made highly available, the way Flickr's design is described doing it:** run two (or more) counter instances, each incrementing by `k` from a different starting offset — e.g., with two servers, one hands out only odd numbers (1, 3, 5, …) and the other only even numbers (2, 4, 6, …). Either one can go down without stopping ID issuance, since the survivor keeps producing valid, non-colliding values on its own. This removes the *availability* single point of failure, but it's still a small, fixed set of coordinating counters that every write has to call over the network — it doesn't scale write throughput the way a fully local scheme does, it just tolerates one node failing.
   - **A dedicated distributed counter primitive — Redis `INCR` (single-threaded, atomic by construction), or an atomic counter built on ZooKeeper/etcd.** These give a single logical counter with real high-availability underneath (replication, leader election), so no single machine failing takes the counter down — but every write still pays a network round trip to whatever node currently holds the authoritative count, and the counter's own throughput ceiling (however well-replicated) is still shared by the entire fleet of writers, unlike a scheme where each node counts independently.
   - **Batch/range allocation per app server** (each app server asks the counter service for a block of, say, 10,000 consecutive IDs at once, then hands those out locally without any further network calls until the block is exhausted). This is the version that's actually closest to being coordination-free on the hot path: coordination happens only once per 10,000 writes (to claim a new block), not once per write — directly reducing the load on whatever counter mechanism sits underneath, whichever of the above it is.
   - **Fully local, no-coordination-at-all counters, only possible by giving up strict global ordering** — e.g., each app server (or shard) maintains its own independent counter with a distinct starting offset and step size (the same odd/even trick above, generalized to more shards), or embeds a machine identifier alongside a local sequence number the way a general-purpose distributed ID generator would. This is fully distributed and has no single point of failure at all, but the resulting codes are no longer *globally* sequential — only sequential per shard — which is actually fine here, since §2 already established that this system doesn't need sortable codes the way some ID-generation problems do.

   In short: a global counter is a spectrum from "one database, one clear SPOF" to "many independent local counters, no SPOF, but no longer one shared sequence" — and batch allocation is the usual practical middle ground, since it keeps a single authoritative counter (simple to reason about) while making the network round trip rare enough that it stops being the write path's bottleneck.
   - *Downside (applies regardless of which implementation above is used):* codes issued this way are sequential in Base62 (…`a`, `b`, `c`… roughly), which makes them **guessable/enumerable** — an attacker can walk the whole code space by incrementing, which is a real privacy and scraping risk for a public-facing system (see §9). Production systems using this approach almost always add a **bit-shuffle or XOR mask** step between the raw counter and the Base62 encoding specifically to break this predictability, without giving up the "no collision, ever" guarantee (the mapping is still a bijection, just not an obviously monotonic one to an outside observer).
   - *Worked example:* counter value `125000000` (the 125-millionth short URL created) encodes in Base62 as `8sud2`. The very next request gets `125000001`, which encodes as `8sud3` — one character different, and trivially guessable by anyone who has seen one valid code. Applying a fixed XOR mask (e.g., XOR the counter with a constant 64-bit key known only to the service) before encoding turns `125000000` and `125000001` into two unrelated-looking integers, which then Base62-encode to two codes with no visible relationship — while the mapping from counter value to code is still one-to-one, so uniqueness is untouched.

3. **Key Generation Service (KGS) — pre-mint a large batch of random, unique 7-character codes ahead of time, store them in an "unused keys" table/queue, and have each app server pull a batch (e.g., a few thousand) into local memory at startup.** Handing out a code from the write path is then a pure in-memory pop — no hash, no counter coordination, no collision check at write time, because uniqueness was already guaranteed when the batch was generated.

   **What KGS actually is, end to end:**

   - **Two logical stores.** An `unused_keys` table (or queue) holding codes nobody has claimed yet, and the `mapping` table itself (§6) which — once a code is written there with a real `long_url` — is the source of truth for "this code is now used." KGS doesn't need a separate `used_keys` table; a code is "used" the instant it's inserted into the mapping table, and it's removed from `unused_keys` the moment a server checks it out (see below), so the two stores never overlap.
   - **How the unused pool is generated without needing to check the database for collisions.** Naively generating random 7-character strings and checking each one against the (eventually billions-of-rows) mapping table would be slow and would get slower as the table fills up — exactly the cost this scheme is trying to avoid. Instead, a background generator job draws from an integer counter (0, 1, 2, …, up to 62⁷−1) and passes each integer through a fixed, keyed pseudorandom permutation (for example, a small Feistel network or a full-period linear congruential generator, keyed with a secret known only to the generator) before Base62-encoding it. A permutation is a bijection over the *whole* 62⁷-value space — every integer maps to exactly one output and no two integers ever map to the same output — so as long as the generator simply advances its input counter and never repeats an input value, the resulting codes are structurally guaranteed distinct from each other, with no per-code existence check against the database needed at all. This is the same idea as the masked-counter approach in item 2, just used here to *pre-generate a pool* rather than to encode codes at request time.
   - **Checking out a batch.** When an app server starts up (or its local pool drops below a low-water mark, e.g. 10% remaining), it asks KGS for the next, say, 10,000 codes. KGS atomically claims that range (e.g., `DELETE ... LIMIT 10000 RETURNING code` against the `unused_keys` table, or advancing its own internal counter by 10,000 if using the permutation approach above) and hands the whole batch to that one server. Because the range is claimed atomically and each server gets a disjoint slice, two servers can never be handed overlapping codes.
   - **Dispensing a code on the write path.** Once a server holds a batch in local memory, satisfying an incoming `shorten()` request is a plain in-process pop from that in-memory list — no network call, no lock, no coordination with any other server, which is exactly what keeps the write path's *steady-state* latency independent of KGS's own availability.

   **Is KGS itself a single point of failure? Is it distributed?** This splits cleanly along the same lines as the counter service in item 2:

   - **KGS being briefly unreachable does not stop writes that are already in progress.** Any app server currently holding an unused batch keeps serving `shorten()` requests purely from memory — KGS is only consulted at startup and at refill time, not per-request, so a short KGS outage is invisible to most traffic (directly analogous to how a ZooKeeper-style worker-ID assignment outage in a general ID generator only blocks *new* workers from starting, not already-running ones).
   - **KGS *is* a single point of failure for any server that runs its local pool all the way to empty while KGS is down** — that server can no longer mint new codes until KGS recovers, though it can still serve reads/redirects normally, since those don't touch KGS at all.
   - **Making KGS itself not a SPOF** is the same menu of options as any small stateful service: run it as a small HA cluster (a leader plus standbys) over a replicated store for the `unused_keys` table/counter, size batches and refill thresholds generously enough that a server's local buffer comfortably outlasts a typical KGS restart, and monitor the unused-pool's remaining size the same way any finite resource pool would be monitored, so a shortage is caught long before any server actually empties its batch.
   - **KGS's own throughput requirement is far lower than the mapping store's** — it's only consulted once per batch (e.g., once per 10,000 writes), not once per write, so even a comparatively simple, non-heavily-scaled KGS deployment can comfortably keep up with the ~200 writes/sec (§3) this system expects, with enormous headroom.
   - *Downside:* if an app server crashes with unused keys still checked out of its local batch, those keys are effectively (harmlessly) wasted unless a reclamation process specifically looks for and recovers them — a small, bounded cost (a batch is a tiny fraction of the 3.5-trillion-code space) in exchange for a write path that touches no shared state at request time. This is the approach real large-scale shorteners (e.g., early Bitly-style architectures) are commonly described as using, precisely because it avoids needing coordination on the hot path while also avoiding the enumerability problem that a raw counter has.
   - *Worked example:* KGS's internal counter is at `84,213,900,551`. A new app server starts up and requests a batch; KGS claims the range `[84,213,900,551, 84,213,910,550]` (10,000 values), advances its own counter past that range so no other server can be handed any value in it, and returns the batch. The server runs each of those 10,000 integers through the shared permutation and Base62-encodes the result, giving it 10,000 ready-to-use codes like `6pfSEoT`, `9kLpQ2z`, `TU71Eay`, … sitting in local memory. The next `shorten()` call on that server just pops one off this list and inserts `(code → long_url)` into the mapping table — no database read, no collision check, no call back to KGS.

**Comparison:**

| Approach | Coordination at write time | Collision risk | Guessable? | Notes |
|---|---|---|---|---|
| Hash + truncate | None, but needs a collision check on every write | Real, grows with truncation length | No (looks random) | Simplest to reason about; the collision check is the real cost |
| Counter + Base62 | Needs a coordinated counter or generator | None — bijective | Yes, unless masked | Best guarantee; SPOF/HA story depends entirely on which counter implementation is chosen (single DB → real SPOF; HA counter or batch allocation → no SPOF, at the cost of more moving parts) |
| KGS (pre-generated pool) | None at request time; batching happens ahead of demand | None — pre-checked unique | No | What most high-scale systems converge on |

This document uses **KGS with random 7-character Base62 codes** as the reference design for the rest of the sections, since it gets the best combination of "no coordination on the hot path," "no collision risk," and "not enumerable" simultaneously.

**Custom aliases.** When a user requests a specific alias (`sho.rt/my-launch`), that write bypasses the KGS pool entirely and goes straight to a uniqueness check against the mapping table (an insert with a uniqueness constraint on the code column, or a conditional put in a key-value store) — reserving the "human-chosen" namespace as logically separate from, but stored in the same table as, machine-generated codes.

## 5. High-Level Architecture

```
                              ┌──────────────┐
            Client ─────────► │ Load Balancer│
                              └──────┬───────┘
                                     │
                 ┌───────────────────┴────────────────────┐
                 │                                         │
                 ▼                                         ▼
          ┌─────────────┐                          ┌───────────────┐
          │ Write Service│                          │ Read/Redirect  │
          │ (shorten)    │                          │ Service        │
          └──────┬───────┘                          └──────┬─────────┘
                 │ pulls a code                             │
                 ▼                                          │ 1. hash(short_code)
          ┌─────────────┐                                   │    to pick a shard
          │ Key Gen Svc  │                                   ▼
          │ (KGS) pool   │                          ┌───────────────┐
          └──────┬───────┘                          │ Cache (Redis/  │
                 │ insert new mapping               │ Memcached, LRU)│
                 │ (write goes straight to the       └──────┬─────────┘
                 │  shard owning that code)                 │ 2. on cache miss,
                 ▼                                          │    query that same
 ┌───────────────────────────────────────────────────┐      │    shard directly
 │         Mapping Store (short_code → long_url)       │◄────┘
 │   NoSQL / distributed KV store, sharded by code      │
 │   (shard = hash(short_code) mod num_shards)           │
 └───────────────────────────────────────────────────┘
                        │
                        ▼ (async, off the hot path)
               ┌───────────────────┐
               │ Analytics pipeline │
               │ (queue → stream →  │
               │  aggregation store)│
               └───────────────────┘
```

The **write path** and **read path** are deliberately separate services (even if co-deployed) because they have almost opposite load profiles and failure tolerances: writes can tolerate a few hundred milliseconds and the occasional retry; reads cannot, and read availability matters far more given how link-sharing works (§2).

**How a request finds the right shard.** There is no separate directory or lookup service standing in front of the mapping store. Both the write service (when it inserts a new `short_code → long_url` row) and the read service (when it looks one up on a cache miss) run the *same* deterministic partitioning function over the short code itself — typically `hash(short_code) mod num_shards`, or, in a managed store like DynamoDB/Cassandra, the partition key's hash is used internally by the database client to route the request to the node that owns it. Because this function only depends on the code's own bytes, any service instance can compute, without asking anyone else, exactly which shard a given code lives on — this is what lets the read path go straight to the right node on a cache miss instead of broadcasting the query to every shard or consulting a separate routing table.

## 6. Data Model

**Primary mapping table (the one lookup that matters for the hot path):**

| Field | Type | Notes |
|---|---|---|
| `short_code` | string (7 chars) | Partition/primary key |
| `long_url` | string (up to ~2048 bytes) | The destination |
| `created_at` | timestamp | |
| `expires_at` | timestamp, nullable | Null = never expires |
| `owner_id` | string, nullable | For anonymous links, null |
| `is_custom_alias` | boolean | Distinguishes user-chosen from generated codes |
| `status` | enum (active / disabled / flagged) | Set to `flagged` by the security scan in §9 without deleting the record |

**Why a key-value / wide-column NoSQL store (DynamoDB, Cassandra) rather than a relational database:** the entire hot-path access pattern is a single-key point lookup (`get(short_code)`), with no joins and no range scans needed on that table. A NoSQL store gives horizontal write/read scalability and predictable single-digit-millisecond point-lookup latency at the billions-of-rows scale estimated in §3, at the cost of the multi-record transactional guarantees a relational database would offer — guarantees this table never actually needs, since each row is independent of every other row. A relational database remains a perfectly reasonable choice at smaller scale, or if the team already operates one and doesn't yet need to shard past a single instance's capacity.

**Sharding.** Partition by `short_code` (or a hash of it) rather than by time or by owner — this spreads both the ~200 writes/sec and the ~20,000 reads/sec estimated in §3 evenly across shards, since Base62 codes (especially KGS-issued random ones) are already uniformly distributed and carry no natural time- or owner-based hot spot.

A separate, much smaller **analytics store** (§8) and **KGS unused-key pool** (§4) are logically distinct tables/services from this mapping table, even though they may live in the same physical database cluster.

## 7. Redirect Mechanics: 301 vs. 302

This is a genuine, consequential trade-off, not a cosmetic HTTP detail:

- **301 (Moved Permanently)** tells the browser and any intermediate caches/CDNs that this redirect is permanent — they are permitted to cache it and, on subsequent clicks, go straight to the long URL **without ever contacting the shortener again**. This makes repeat clicks on a popular link essentially free for the shortener's infrastructure, but it also means the service loses visibility into most of those clicks (undercounted analytics), and it makes changing or revoking a link unreliable — a client that already cached the 301 will keep skipping the shortener even after the link is disabled or its destination changed.
- **302 (Found / temporary redirect)** tells clients not to cache the mapping, so every single click — first or hundredth — actually hits the redirect service. This costs more request volume against the read path, but it is what makes accurate click analytics, link expiration, and link revocation actually work as designed.

**Recommendation:** use **302** as the default, precisely because analytics, expiration, and revocation (§2's stated requirements) all depend on every click reaching the service; the extra request volume this generates is exactly the load §3 and §5's cache layer are already sized to absorb. Reserve 301 for a narrow, deliberate case — e.g., a link the owner has explicitly marked as permanent and where analytics/revocation are known not to matter — rather than as the default.

## 8. Analytics

Click analytics (count, referrer, geography, device, timestamp) must never sit on the redirect's hot path — the redirect response should be returned to the client immediately after the cache/store lookup, with the analytics event **fired asynchronously** (a message dropped onto a queue such as Kafka/Kinesis/SQS) rather than written synchronously before responding. A separate stream-processing/aggregation layer consumes that queue and rolls events up into a reporting store (total clicks, clicks per day, top referrers, top countries) that a dashboard or API can query independently of the redirect path.

This separation matters for the same reason the write and read services are kept separate in §5: analytics writes have a very different durability and latency profile (a lost or delayed click event is a minor, tolerable data-quality issue) than a redirect response (a delayed redirect is a directly felt user-facing failure) — coupling them would make the read path's latency and availability hostage to the analytics pipeline's health.

## 9. Security Considerations

**Enumeration / guessability.** Because a short code is deliberately compact, an attacker who can iterate through the code space can discover other users' (possibly private or unlisted) links. This is the direct motivation for choosing random KGS-issued codes (or a masked counter, §4) over a raw sequential counter — anything that lets a code space be walked in order is a scraping and privacy risk the moment links are meant to be unlisted rather than fully public.

**Malicious destination URLs (phishing/malware).** A short link hides its destination until clicked, which is exactly what makes shorteners an attractive vector for phishing and malware distribution — a link that reads `sho.rt/6pfSEoT` gives a victim no visual clue at all about where it actually leads, unlike a long URL where a suspicious domain is at least visible before clicking.

   - **Write-time scan.** When a long URL is submitted to `shorten()`, before the code is even returned to the caller, the service calls a threat-intelligence/reputation API (e.g., a Google Safe Browsing–style lookup, or a paid threat-feed provider) with that URL and waits for a verdict. This can be done synchronously because writes happen at ~200/sec (§3) and can tolerate an extra tens-of-milliseconds network call — a cost the read path, at ~20,000/sec, absolutely could not afford to pay per request. If the verdict comes back malicious, the service can reject the request outright (`400 Bad Request` with an explanation) or create the mapping but immediately set `status = flagged` (§6) rather than `active`, depending on how strict the product wants to be.
   - **Why a synchronous check on write isn't the whole story.** Reputation databases are updated continuously, and a URL that was clean at creation time can be repurposed for phishing afterward (a common tactic: register a link, let it sit clean for a while to build trust or pass initial review, then swap the destination page's content later) — the write-time check alone cannot catch this, since it only ever sees the URL once, at creation.
   - **Read-time protection without a live call per click.** Because the read path can't afford a remote reputation lookup on every redirect, the actual read-time control is a **denylist held in a fast cache (Redis-resident, O(1) lookup)** — checked on every redirect, but purely against already-known-bad codes, never against a live external service. This denylist is populated two ways: immediately, from the write-time scan's own verdict; and continuously, from an **asynchronous re-scan job** that periodically re-checks previously-approved URLs against the reputation service in the background, plus from **user abuse reports** ("report this link" flows) that flag a code for review. When a code lands on the denylist by any of these paths, the redirect service checks it there before (or instead of) touching the mapping store, and serves an interstitial warning page instead of redirecting.
   - **Worked example:** a user submits `sho.rt/api/shorten` with a URL that currently resolves to a legitimate-looking login page. The write-time scan comes back clean, and the service returns code `9kLpQ2z`. Two weeks later, the destination page is swapped by its owner (who has since been compromised) to a credential-harvesting clone of a bank's login page. A background re-scan job, running hourly over recently-created links, re-checks `9kLpQ2z`'s destination, gets a "phishing" verdict this time, sets `status = flagged` in the mapping table, and pushes `9kLpQ2z` onto the Redis denylist. The very next click on `sho.rt/9kLpQ2z` hits the denylist check first, finds it flagged, and is served a warning interstitial instead of a 302 to the phishing page — without the redirect service ever having made a live reputation-service call itself.

**SSRF (Server-Side Request Forgery).** This risk only exists for any part of the system that itself makes an outbound HTTP request to the *destination* URL — for example, a feature that fetches the target page to generate a link preview (title, thumbnail, OpenGraph image) or to verify the URL is reachable before shortening it. The redirect itself (§7) is not SSRF-prone, because a 302 just tells the *client's* browser to go fetch the URL — the shortener's own servers never make that request. The danger is specifically in any *auxiliary* feature that does.

   - **The attack.** An attacker submits, as the "long URL" to shorten, something like `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (a cloud provider's instance-metadata endpoint, which on many cloud VMs returns temporary credentials with no auth required if the request comes from the instance itself) or `http://localhost:6379/` (an internal Redis instance with no auth, reachable only from inside the network). If the shortener's own backend fetches that URL server-side to build a preview, it is the *shortener's infrastructure* — sitting inside the trusted network, with network-level access to those internal services — that ends up making the request, not the attacker's own machine. The response (credentials, internal data) can then be reflected back to the attacker through whatever preview/response mechanism exists.
   - **Why checking the hostname alone isn't enough.** A naive defense that just string-matches the URL's hostname against a blocklist (`localhost`, `169.254.169.254`, etc.) can be bypassed by **DNS rebinding**: the attacker registers a public domain whose DNS record resolves to a public, harmless IP at the moment the hostname check runs, then changes that same domain's DNS to point to an internal/private IP by the time the actual fetch happens moments later — the hostname passed the check, but the *IP actually connected to* was never validated. A correct defense has to resolve the hostname to an IP and validate the IP itself, immediately before connecting, not just validate the hostname string.
   - **The mitigations, concretely:**
     - Resolve the destination hostname to an IP and reject the fetch if that IP falls in loopback (`127.0.0.0/8`, `::1`), link-local (`169.254.0.0/16`, which covers the cloud metadata endpoint), or private ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`) — checked against the resolved IP, not the hostname string, and re-checked at actual connection time to close the DNS-rebinding gap above.
     - Disable automatic redirect-following in the fetching HTTP client, or re-validate the destination IP after every hop — a URL can pass validation itself but redirect (via an HTTP 3xx) to an internal address on the second hop, which a client that blindly follows redirects would happily connect to.
     - Run the fetcher in a network-isolated environment (its own subnet/VPC with no route to internal services, or an egress firewall allow-listing only outbound HTTP/HTTPS to the public internet) as defense in depth, so that even a validation bug doesn't translate into an actual internal-network connection.
     - Apply a short connection timeout and disallow non-HTTP(S) schemes (`file://`, `gopher://`, `dict://`, etc.), which have historically been used to reach otherwise-unreachable internal services through URL-fetching bugs in other systems.
   - **Worked example:** the preview-generation feature receives `http://metadata.internal-alias.example/latest/meta-data/`, a hostname the attacker controls the DNS for. At validation time, that hostname resolves to `8.8.8.8` (a harmless public IP) and passes the check. Two seconds later, the attacker flips the DNS record to point the same hostname at `169.254.169.254`. If the fetcher re-resolves the hostname at connection time (rather than reusing the IP it validated a moment ago) and re-validates *that* IP immediately before connecting, it catches the rebind and refuses the connection — this is precisely why "validate once, then trust the hostname for the rest of the request" is not a safe implementation, and why the IP check has to happen right at connection time.

**Rate limiting.** Rate-limit URL creation per user/IP (a token-bucket or sliding-window limiter, typically enforced at the API gateway/load balancer) — most abuse of a shortener is a flood of automated creation requests (spam/phishing link farms), not organic write load, so this is a security control as much as a capacity one, and it protects the KGS/mapping store from being used as a bulk phishing-link factory rather than protecting against genuine legitimate traffic spikes.

## 10. Expiration and Cleanup

Links may carry an optional `expires_at`. Two complementary mechanisms handle expiry without adding cost to the hot path:

- **Lazy expiry check on read:** when a redirect request hits the mapping store (cache miss) and finds `expires_at` in the past, return a "link expired" response instead of redirecting, and evict/skip caching that entry.
- **Background reaper:** a periodic batch job scans for and removes (or archives) expired rows, so the mapping table doesn't grow unbounded with dead entries that would otherwise only ever get cleaned up opportunistically on a future read that may never come for an unpopular link.

This follows the same read/write separation theme as the rest of this design: expiry *correctness* is enforced cheaply and immediately on the read path (a single timestamp comparison), while expiry *cleanup* (reclaiming storage) is pushed to an asynchronous background process that never blocks a user-facing request.

## 11. Failure and Availability Considerations

**Cache layer down.** The redirect service falls back to the mapping store directly; latency degrades (more requests reach the database) but correctness is unaffected — this is why the cache should be treated as a pure performance optimization, never as a system of record, and why the mapping store must always be provisioned to survive a full cache outage, even if only at degraded latency.

**KGS pool exhausted on an app server.** The server blocks briefly and requests a new batch from the KGS rather than failing the write outright — a short, self-limiting latency spike on the infrequent write path, not a correctness issue, and one that's easy to avoid by sizing batch pulls and replenishment thresholds against expected write throughput (§3).

**A datacenter/region goes down.** Because the mapping store is sharded and (ideally) replicated across regions, and because the read path's correctness never depends on any single region being reachable, losing one region degrades capacity, not correctness — a decentralized, no-shared-state hot path tolerates partial outages far better than any design that puts a single coordinator on the request path.

**Database write conflicts.** Because every short code is unique by construction (KGS pre-allocation, §4) before it is ever handed to a write, the insert into the mapping table can be a simple, conflict-free write — there is no last-writer-wins race to resolve, unlike a design where uniqueness was only checked, not guaranteed, at write time.

## 12. Summary Table

| Goal / Problem | Technique |
|---|---|
| Short, compact codes | Base62 alphabet, 7 characters (~3.5 trillion codes) |
| No collisions, no write-time coordination | Key Generation Service pre-minting unique random codes into a pool |
| Not guessable/enumerable | Random KGS codes (or a masked counter) instead of a raw sequential counter |
| Fast redirects at ~20K+ reads/sec | LRU cache (Redis/Memcached) in front of the mapping store, sized to the hot ~20% of links |
| Read-heavy horizontal scale | NoSQL key-value store sharded by short code |
| Accurate analytics, working expiry/revocation | 302 (temporary) redirects, not 301 |
| Analytics without hurting redirect latency | Asynchronous event queue + separate aggregation store |
| Phishing/malware links | Write-time reputation scan + Redis-resident denylist checked on read |
| SSRF via preview-fetching | Block loopback/link-local/private IP ranges in any server-side fetch |
| Abuse / link-farming | Per-user/IP rate limiting on the write path |
| Storage growth from expired links | Lazy expiry check on read + async background reaper |
| Regional/cache/pool outages | Cache is a pure optimization (store is source of truth); KGS exhaustion self-heals; multi-region replication keeps reads correct during a regional outage |

## References

1. ByteByteGo, "Design A URL Shortener": https://bytebytego.com/courses/system-design-interview/design-a-url-shortener
2. AlgoMaster, "Design URL Shortener": https://algomaster.io/learn/system-design-interviews/design-url-shortener
3. GeeksforGeeks, "URL Shortener System Design": https://www.geeksforgeeks.org/system-design/system-design-url-shortening-service/
4. Karan Pratap Singh, "URL Shortener | System Design": https://www.karanpratapsingh.com/courses/system-design/url-shortener
5. EnjoyAlgorithms, "URL Shortening Service like TinyURL": https://www.enjoyalgorithms.com/blog/design-a-url-shortening-service-like-tiny-url/
6. System Design Handbook, "TinyURL System Design": https://www.systemdesignhandbook.com/guides/tinyurl-system-design/
