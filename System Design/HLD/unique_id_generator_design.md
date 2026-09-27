# Designing a Distributed Unique ID Generator

## 1. Overview and Goals

A unique ID generator produces identifiers for records (orders, users, posts, messages) in a system where a single auto-incrementing counter on one database no longer works — either because there are multiple database shards that would each restart counting from 1, or because a single counter is a throughput and availability bottleneck at scale.

Design goals for a production-grade distributed ID generator:

- **Uniqueness** — no two IDs collide, ever, across every node that can mint one, for the lifetime of the system.
- **Availability** — the generator must keep issuing IDs even when some nodes or the network are unhealthy; it cannot become a single point of failure the way one database auto-increment column is.
- **Low latency, high throughput** — minting an ID is on the hot path of nearly every write; it must be fast (ideally sub-millisecond, in-process) and must scale horizontally as request volume grows.
- **Roughly sortable / monotonically increasing (usually)** — many systems want IDs that increase over time, because a numerically or lexicographically increasing ID sorts naturally by creation time, keeps database indexes append-friendly (§9), and lets pagination/range-scans work without a separate timestamp column.
- **Compact representation** — a 64-bit integer is dramatically cheaper to store, index, and compare than a 128-bit or string ID, and this compounds across billions of rows and every index that includes the ID.

These goals conflict, and the conflicts are the entire substance of this design:

- Uniqueness *without* coordination (every node generates independently, no locking, no shared counter) is what makes the system available and fast — but the less coordination there is, the harder it is to also guarantee ordering across nodes.
- Compact, sortable, and coordination-free are in genuine tension: a fully random 128-bit UUID needs no coordination at all but sorts (and compresses) poorly; a single coordinated counter sorts perfectly but isn't coordination-free. Every approach below is a specific point on this trade-off surface, not a strictly better or worse option in isolation.

## 2. Requirements

**Functional requirements:**

- `generate_id() -> ID` — the only operation. No `get`, no `delete`; an ID generator has no read path of its own once an ID is issued (it does not remember which IDs it has handed out beyond what's needed to avoid a duplicate).
- IDs should ideally be numerical, or at least comparable/sortable, so they can be used as primary keys and index efficiently.

**Non-functional requirements:**

- **Scale.** Systems like Twitter (Snowflake's original motivating case) needed tens of thousands of new IDs per second, sustained, from many machines simultaneously.
- **High availability.** The ID service sits directly in the write path of every other service that depends on it — if it goes down, so does everything that needs a new ID, so it must be designed with no single point of failure.
- **IDs fit in 64 bits** (a common, though not universal, target) — this is what keeps the ID cheap to store and index compared to a 128-bit UUID, and is achievable if a design gives up some uniqueness "budget" (random bits) in exchange for structure (timestamp + machine ID + sequence, see §5).
- **IDs increase over time** (not necessarily by exactly 1 with every single call, and not necessarily *strictly* — "roughly time-ordered" is usually enough, see §9's k-sortability discussion).

## 3. Approaches Overview

There are four broad families, each trading coordination for a different weakness:

1. **Multi-master / database auto-increment with offset** (§4) — reuse the database's own counter mechanism, avoiding collisions across shards purely by arithmetic (each shard's counter starts at a different offset and increments by the shard count). Simple, but doesn't scale horizontally and the IDs aren't chronologically ordered across shards.
2. **UUID** (§5) — every node generates independently, no coordination, no shared state at all. Trivially available and infinitely scalable, but 128 bits, not (numerically) sortable in the most common version, and not guaranteed monotonic.
3. **Ticket server / centralized counter service** (§6) — one dedicated service (or database) hands out ranges of IDs. Simple and strictly ordered, but the ticket server itself becomes the single point of failure/bottleneck unless it's made highly available in its own right.
4. **Snowflake-style: timestamp + worker ID + sequence, encoded into a single integer, generated locally** (§7) — each node embeds the current time, its own pre-assigned machine ID, and a local per-millisecond counter into one 64-bit number. No coordination is needed *at generation time* (only at machine-ID-assignment time, once, per node) — this is what makes it the usual answer for "design a distributed unique ID generator" precisely because it sits at the sweet spot of the trade-off surface in §1: compact, roughly sortable, and coordination-free on the hot path.

## 4. Multi-Master Replication (Database Auto-Increment with Offset)

**Mechanism.** Instead of one database with one `AUTO_INCREMENT` column starting at 1 and stepping by 1, configure `k` database instances so that instance `i` starts its counter at `i` and increments by `k` (not by 1) on every insert. With `k=3`: DB1 produces 1, 4, 7, 10…; DB2 produces 2, 5, 8, 11…; DB3 produces 3, 6, 9, 12…

**Why this avoids collisions.** Every ID produced by any of the `k` databases is congruent to a distinct residue mod `k` — DB1's IDs are all `≡ 1 (mod 3)`, DB2's are all `≡ 2 (mod 3)`, DB3's are all `≡ 0 (mod 3)`. No two databases can ever land on the same integer, by pure number theory, with zero communication between them at generation time.

**What this buys you, concretely.** Each database can be written to independently and in parallel — there's no shared counter to contend on, so this trivially scales writes across `k` machines instead of one. It also reuses infrastructure a team already has (an RDBMS with auto-increment) rather than building a new service.

**What it costs you — and why each cost matters:**

- **Not horizontally scalable in the way that matters most.** Adding a new database instance later means recomputing the offset scheme (a new instance needs a residue class not already in use, and existing rows' residues are now fixed forever) — this is an operational, coordinated change, not something you can do by just adding a box. Contrast with Snowflake (§7), where adding a node only requires handing it an unused machine ID.
- **IDs don't increase across databases, only within one database.** DB1's sequence (1, 4, 7…) and DB2's sequence (2, 5, 8…) interleave in real time in a way that has nothing to do with which write actually happened first — `7` (from DB1) might be inserted chronologically *after* `8` (from DB2) despite being numerically smaller. Any application logic that assumes "bigger ID = created later" breaks the moment rows from different shards are compared.
- **Doesn't work at all once you need more IDs per second than a single database's write throughput** for *any one* of the `k` databases specifically, since each one is still a single machine underneath.

This approach is a reasonable answer when the write volume is modest and the team wants to stay entirely within existing RDBMS tooling — but it is a stopgap, not the design most systems converge on once ID throughput genuinely needs to scale past what a handful of database instances can do; it's included here because interviewers commonly expect it named as the "naive but real" starting point.

**Brief recap, in short:** each shard just runs a normal local auto-increment (`auto_increment_increment`/`auto_increment_offset` in MySQL terms), but with step = `k` and a distinct starting offset per shard — e.g. with `k=4`: shard 0 → 1,5,9,13…; shard 1 → 2,6,10,14…; shard 2 → 3,7,11,15…; shard 3 → 4,8,12,16… Different residues mod `k` can never collide, so no cross-shard communication is needed. Adding a 5th shard later means changing the modulus for everyone (not just plugging in a new offset), and comparing IDs across shards doesn't tell you which was created first — only within one shard is the sequence chronological.

## 5. UUID (Universally Unique Identifier)

A UUID is a 128-bit number, conventionally rendered as 32 hex digits in five dash-separated groups (`f47ac10b-58cc-4372-a567-0e02b2c3d479`).

**Why it works with zero coordination.** The collision probability follows from simple birthday-paradox math: with `M` possible hash values, a 50% chance that *some* pair among your generated values collides shows up once you've generated roughly `1.17 × √M` of them — far fewer than `M` itself, but still an enormous number once `M` is large. With 128 bits (`M ≈ 3.4 × 10^38`), that threshold sits around `2.1 × 10^19` — and RFC 4122 states it more concretely: you'd need to generate roughly 2.71 quintillion version-4 UUIDs before hitting a 50% chance of any collision. That's astronomically beyond any real system's actual ID volume, so in practice the risk is treated as zero. No node needs to know what any other node has generated, ever — this is the maximum point on the "no coordination" end of the trade-off spectrum from §1.

**What it costs:**

- **128 bits, not 64.** Twice the storage and index cost of an integer ID, which compounds across every foreign key and every index that includes it.
- **Not numerically sortable (for the most common version).** A version-4 (fully random) UUID has no relationship whatsoever between generation time and value — inserting them as a primary key in a B-tree index causes writes to land at random positions across the whole index rather than appending at the end, which is dramatically worse for index locality and page cache behavior than an append-mostly monotonic key (§9 covers this in more depth for the Snowflake case).
- **Not human-typeable/shareable as comfortably as a short integer**, though this is a minor, mostly cosmetic concern.

**The different UUID versions exist specifically because "not sortable" was recognized as a real cost, and later versions were designed to fix it:**

| Version | Contents | Sortable by time? | Notes |
|---|---|---|---|
| v1 (time-based) | 60-bit timestamp (100ns ticks since 1582) + a 48-bit node ID (originally the generating machine's MAC address) + a clock sequence | Roughly, but the timestamp bits are arranged non-contiguously (interleaved with version bits) so raw lexicographic/numeric sort does **not** match time order | Leaks the generating machine's MAC address into every ID — a real, historically-cited privacy/security leak (this is literally how the Melissa virus author was identified in 1999, via the MAC address embedded in a v1 UUID left in a document's metadata) |
| v3 / v5 | Hash (MD5 for v3, SHA-1 for v5) of a namespace + a name | Not time-based at all — deterministic given the same input | Useful when you want the *same* input to always produce the *same* ID (e.g., deriving a stable ID from a URL), not for general-purpose generation |
| v4 (random) | 122 random bits (6 bits fixed for version/variant) | No | The most common "just give me a UUID" default; maximum entropy, zero structure, zero coordination |
| v6 | A re-ordering of v1's fields so the timestamp bits *are* contiguous and in the right order | Yes, byte/numeric order now matches time order | A backwards-compatible fix for v1's sort problem while keeping the MAC-address field (inheriting v1's privacy leak unless the node ID is randomized instead, which the spec allows) |
| v7 (newer, increasingly the default recommendation) | 48-bit Unix millisecond timestamp (big-endian, so it sorts correctly as raw bytes) + ~74 random bits | Yes | Purpose-built to close the gap with Snowflake-style IDs: sortable like a timestamp-prefixed ID, but still 128 bits generated with zero coordination and no embedded machine identifier — the practical modern answer when the "not sortable" and "not compact" costs of classic UUIDs matter but a dedicated ID-generation service (§7) is more infrastructure than the problem warrants |

The version table is itself a compressed history of this whole design space: each version is a response to a specific, named cost of an earlier one, and v7's existence is direct evidence that "sortable and coordination-free" is achievable — it just costs more bits than Snowflake's 64 to get there without any assigned machine ID.

## 6. Ticket Server (Centralized Counter)

**Mechanism.** One dedicated, centralized database (Flickr's well-known approach uses MySQL for this) has a single table whose sole job is producing the next number:

```sql
CREATE TABLE Tickets64 (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  stub CHAR(1) NOT NULL DEFAULT '',
  PRIMARY KEY (id),
  UNIQUE KEY stub (stub)
);

REPLACE INTO Tickets64 (stub) VALUES ('a');
SELECT LAST_INSERT_ID();
```

Every service that needs a new ID calls this "ticket server" instead of maintaining its own counter. The `REPLACE INTO` + `LAST_INSERT_ID()` pattern atomically produces the next integer and hands it back — simple, strictly increasing, globally unique by construction (there's exactly one counter).

**What this buys you:** dead-simple correctness (it's just one auto-increment column, the most well-understood primitive in relational databases) and strictly monotonic, gap-permitting-but-never-repeating IDs.

**What it costs, and how real deployments address each cost:**

- **Single point of failure, if there's only one ticket server.** Flickr's actual mitigation is exactly the multi-master offset trick from §4, but applied to just *two* servers instead of many application shards: one ticket server increments by 2 starting at 1 (odd IDs), the other increments by 2 starting at 2 (even IDs). If either is down, the other keeps serving IDs without collision — this is a small, two-node special case of §4's general technique, used here specifically to eliminate the ticket server's own single-point-of-failure risk rather than to shard application data.
- **A network round trip on every single ID generation.** Every caller, for every ID, pays a request to a remote service — this is real, added latency compared to Snowflake's fully local generation (§7), and the ticket server's own throughput ceiling (bounded by that one database's write rate, even split two ways) becomes the whole system's ceiling.
- **Doesn't encode a timestamp or any other information** — an ID here is just a number; anything else about *when* or *where* it was created has to live in a separate column.

The ticket server is what a team reaches for when correctness and simplicity matter more than raw throughput or per-ID latency, and when the team would rather operate one more small, well-understood database than a bespoke stateless ID-generation fleet. It is the direct conceptual predecessor to Snowflake: Snowflake can be understood as "what if we moved the counter from one shared, remote database into each node's own memory, and glued a timestamp onto it so nodes don't need to talk to each other or agree on a shared sequence at all."

## 7. Snowflake-Style ID Generation (Timestamp + Worker ID + Sequence)

This is the design most interviews are actually asking for, and the one the rest of this document goes deep on. Originated at Twitter (as "Snowflake") to solve exactly the ticket-server bottleneck above; broadly copied since, with minor bit-layout variations, by Instagram, Sony (Sonyflake), Baidu (UidGenerator), and many others.

### 7.1 The Core Idea

Every ID-generating node builds its own IDs **entirely locally**, with no network call and no shared state consulted at generation time, by packing three pieces of information into a single 64-bit integer:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1  ...(64 bits total)
+-+-------------------------------------------------------------+----------+------------+
|0|                    timestamp (41 bits)                       | worker id| sequence   |
+-+-------------------------------------------------------------+----------+------------+
```

A common concrete split (Twitter's original):

| Field | Bits | What it holds |
|---|---|---|
| Sign bit | 1 | Always 0, so the value is always a non-negative signed 64-bit integer (keeps it safely representable in languages/DBs where a 64-bit *unsigned* type is awkward, e.g. Java's `long`) |
| Timestamp | 41 | Milliseconds since a custom **epoch** (not Unix epoch 1970 — a custom, more recent epoch, e.g. the service's own launch date), so 41 bits reach further into the future before overflow than they would counting from 1970 |
| Worker/machine ID | 10 | Uniquely identifies the generating machine/process (§7.3) — 10 bits allows up to 1,024 concurrently active workers |
| Sequence | 12 | A per-worker, per-millisecond counter, reset to 0 at the start of every new millisecond — 12 bits allows up to 4,096 IDs per worker per millisecond |

**Why this specific split is a deliberate, tunable trade-off, not an arbitrary default.** The three fields compete for the same fixed 63 usable bits (64 minus the sign bit): more timestamp bits push the year-2^41-milliseconds overflow date further out but leave fewer bits for workers and per-ms throughput; more worker-ID bits support a larger fleet but reduce either the timestamp's reach or the per-worker throughput; more sequence bits raise the per-worker-per-millisecond ceiling at the cost of the other two. Twitter's 41/10/12 split supports roughly 69 years from the chosen epoch, up to 1,024 workers, and up to 4,096,000 IDs/second/worker (4096 × 1000ms) — sized for their actual fleet size and throughput needs, not a universal constant; Sonyflake, for comparison, uses a coarser 10ms tick with more worker-ID bits, trading finer-grained ordering for supporting more machines.

### 7.2 Why No Coordination Is Needed at Generation Time

Two IDs can only collide if they're identical in **all three** fields simultaneously. Fixing the worker ID as globally unique per node (§7.3 covers *how* that's guaranteed) means two different nodes can never produce a colliding ID no matter what they do internally — their worker-ID field alone already differs. Within a single node, the timestamp+sequence pair is trivially made collision-free because that node is the only writer to its own in-memory counter: it just needs to never hand out the same `(timestamp, sequence)` pair twice, which is a single-threaded (or mutex-protected) local invariant, not a distributed one.

**Why "local" vs. "distributed" is the important distinction here.** A distributed invariant would mean several independent machines have to somehow agree with each other to keep something true — that needs real coordination (network round trips, consensus, cross-machine locks), which is exactly the expensive machinery this design avoids. But `last_timestamp` and `sequence` live only in one worker process's memory; no other machine ever reads or writes them. The only way this invariant could break is *that same process* racing against itself — e.g., two threads inside one worker both calling `generate_id()` concurrently, both reading `sequence` before either has incremented it, and both getting back the same value. That's an ordinary concurrency bug, solved the same way any shared in-memory counter is protected in a multithreaded program (a mutex, an atomic increment, or funneling all calls through one dedicated thread) — it has nothing to do with the network or other machines, which is precisely why it's cheap compared to the worker-ID coordination in §7.3.

This is the crux of the whole design: **uniqueness is decomposed into "different machines can't collide" (solved once, structurally, by disjoint worker IDs) and "one machine can't collide with itself" (solved trivially, since it's the sole writer to its own state)** — no consensus round, no shared counter, no network call, ever required just to mint an ID.

### 7.3 Worker ID Assignment — the One Place Coordination Actually Happens

Uniqueness of the whole scheme rests entirely on no two live workers ever using the same worker ID at the same time — so *this* is the part that genuinely needs coordination, even though ID generation itself doesn't. Several real strategies exist:

- **Static configuration.** Assign worker IDs by hand (or via deployment config/environment variable) at provisioning time, out of a fixed pool. Simple, but manual, error-prone at scale, and a real risk if two machines are ever misconfigured with the same ID by human mistake — nothing in the runtime system itself would catch that.
- **ZooKeeper (or etcd/Consul) sequential ephemeral znodes — Twitter's actual original approach.** Each worker, on startup, creates a sequential ephemeral node under a well-known path (e.g., `/snowflake-workers/worker-`); ZooKeeper atomically appends an incrementing sequence number to the path (`worker-0000000001`, `worker-0000000002`, …), which the worker reads back and uses as its worker ID. **Ephemeral** means the znode is automatically deleted the moment that worker's session ends (crash, graceful shutdown, or a lost heartbeat) — so a worker ID is only ever "in use" while the process that claimed it is actually alive and connected, and a restarted worker gets a *fresh* sequential ID rather than risking silent reuse of a stale one. This pushes the coordination problem entirely onto ZooKeeper's own consensus protocol (ZAB, structurally similar in spirit to Paxos/Raft) — a well-tested, purpose-built system for exactly this kind of "exactly-once, globally agreed" allocation — rather than reinventing it inside the ID generator.
- **Database row claim.** A single table with `(worker_id, hostname, last_heartbeat)` rows; a worker claims a free row (or the row matching its own identity) via a transactional `UPDATE ... WHERE worker_id IS NULL/expired`, and periodically renews a heartbeat/lease on it. Functionally similar to the ZooKeeper approach but reuses existing RDBMS infrastructure instead of adding a new coordination system — the same "which infra do we already operate" trade-off named in §4 for the multi-master approach.
- **Cloud-metadata-derived (e.g., last octet of a private IP, or a Kubernetes pod ordinal in a StatefulSet).** Works well when the deployment platform already guarantees the needed uniqueness (a StatefulSet's pod ordinals are unique and stable by the platform's own contract) — but ties the ID scheme's correctness to an assumption about the deployment platform that has to be re-verified if the platform ever changes (e.g., migrating off StatefulSets to a deployment style that doesn't guarantee stable ordinals silently breaks this).

**The common thread:** worker-ID assignment is the *one* moment in the entire design where something has to look like classic distributed-coordination machinery (leader election, consensus, leases) — because uniqueness here really is a shared, global invariant across the fleet, unlike ID generation itself. Getting this one piece wrong (e.g., two workers racing to claim the same static ID during a bad deploy) silently reintroduces the exact collision the rest of the design goes to great lengths to avoid without coordination.

### 7.4 The Sequence Counter and Clock Handling — Worked Through Carefully

**Normal case.** A worker keeps two pieces of local, in-memory state: `last_timestamp` (the millisecond value used for the most recently generated ID) and `sequence` (the counter within that millisecond). On each `generate_id()` call:

1. Read the current wall-clock time, `now`.
2. If `now > last_timestamp`: a new millisecond has started — reset `sequence = 0`, set `last_timestamp = now`.
3. If `now == last_timestamp`: still the same millisecond — increment `sequence`. If `sequence` would overflow its 12 bits (i.e., 4,096 IDs already handed out this millisecond), **do not** wrap around silently (that would manufacture a duplicate `(timestamp, sequence)` pair) — instead, busy-wait (or sleep) until the clock actually ticks over to the next millisecond, then proceed as in step 2.
4. Pack `(0, timestamp_bits, worker_id, sequence)` into the 64-bit result.

This whole sequence has to run under a single lock (or be handled by a single-threaded actor/goroutine) per worker — it's the one piece of truly shared mutable state within a process, and the reason "a worker" in this design usually means one process/thread owning this state, not literally one physical machine (a machine can run several independent workers, each with its own worker ID, if it needs more than 4,096 IDs/ms).

**The dangerous case: clock moves backward.** `now < last_timestamp` can genuinely happen. NTP (Network Time Protocol) periodically corrects a machine's clock by comparing it against a reference time server and nudging it toward the true time; if the local clock had drifted ahead, that correction can step it *backward*, not just slow it down. Real-world NTP accuracy is typically single-digit-to-tens-of-milliseconds over the internet — good enough for almost everything, but not a guarantee that a clock only ever moves forward.

**NTP running does not, by itself, prevent this — and it's worth being precise about why.** NTP correction happens in two different modes, and only one of them is safe against backward jumps:
- **Slewing.** When the local clock is only slightly off, NTP daemons (`ntpd`, `chronyd`) correct it gradually — speeding up or slowing down the clock's own tick rate for a while until it converges on the true time. This never moves the clock backward; it just makes seconds pass very slightly faster or slower temporarily.
- **Stepping.** When the offset is large (after a VM pause/resume, a hypervisor live-migration, a container being unfrozen after a long pause, a long-lost connection to the time source, or simply first boot), NTP instead sets the clock directly to the correct value in one jump. If the local clock had drifted ahead, that step is a genuine, instantaneous backward jump — precisely the case this guard exists for.

So "we run NTP" does not imply "our clock never goes backward" — it implies corrections happen, some of which are smooth and some of which are hard jumps, depending on how far off the clock had gotten and how the NTP daemon is configured. **The actual guarantee against a duplicate ID doesn't come from NTP at all — it comes from the generator's own check.** Every call to `generate_id()` is forced through the same guarded comparison (`now` vs. `last_timestamp`) inside the same lock that protects `sequence` (§7.4 below); because *every* code path that could emit an ID passes through that one guarded comparison, there is no way for the generator to hand out a timestamp value it has already used, regardless of what the underlying clock does. NTP being well-configured just makes the backward-jump case rarer in practice; it is the generator's own defensive logic — not any property of NTP — that makes the no-duplicate claim actually hold. If the generator naively used `now` as-is here, it could produce a `(timestamp, sequence)` pair identical to one already issued moments ago — a real collision, not a theoretical one. Production Snowflake implementations explicitly guard against this: if `now < last_timestamp` is detected, the worker refuses to generate an ID at all (raising an error, or blocking) until the clock catches back up past `last_timestamp` — accepting a short availability gap on that one worker in exchange for the correctness guarantee never being silently violated. Some implementations instead maintain a small, bounded tolerance (accept a backward jump of a few milliseconds by blocking briefly) but always refuse to ever emit a timestamp smaller than one already used.

**Why this specific failure mode is easy to under-appreciate.** Everything else in this design achieves uniqueness *structurally* (disjoint worker IDs, a single in-process writer) — clock monotonicity is the one place uniqueness instead depends on an *assumption about the environment* (that a worker's clock, once past a value, never sees that value again) holding true, and the guard above exists specifically because that assumption is not automatically true of real wall clocks. Any scheme that leans on wall-clock time for a correctness-adjacent decision (not just for display or logging) needs an explicit, named mitigation like this one, because real clocks aren't perfectly monotonic or synchronized across machines.

### 7.5 Concrete Worked Example

Worker ID `42`. Custom epoch `2024-01-01T00:00:00Z`. At real time `2024-06-15T10:00:00.123Z`, `timestamp = 14,354,400,123` ms since epoch (illustrative).

- Call 1 at this millisecond: `sequence=0` → ID packs `(timestamp=14354400123, worker=42, seq=0)`.
- Call 2, same millisecond: `sequence=1` → same timestamp, same worker, `seq=1` — differs from call 1 only in the sequence field, so no collision.
- Suppose 4,096 calls have now happened in this same millisecond (`sequence` about to overflow its 12 bits): call 4,097 detects the would-be overflow, spins until the clock ticks to `.124`, resets `sequence=0`, and proceeds with the new timestamp.
- Clock then jumps backward to `.120Z` due to an NTP correction: the next call detects `now (.120) < last_timestamp (.124)`, refuses to generate, and blocks (or errors) until real time passes `.124Z` again — at which point it resumes normally.

Every ID produced by worker 42, across all of this, is distinct: same-millisecond IDs differ by `sequence`, different-millisecond IDs differ by `timestamp`, and the backward-clock guard prevents the one scenario that could have made two IDs identical in both fields at once.

## 8. Comparison of Approaches

| Approach | Coordination needed at generation time | Size | Sortable | Availability risk | Throughput ceiling |
|---|---|---|---|---|---|
| Multi-master DB w/ offset (§4) | None per-write, but adding a node needs a global scheme change | Small int (DB-native) | Only within one shard | Each DB is its own SPOF unless separately made HA | Bounded by one DB's write rate × k |
| UUID v4 (§5) | None, ever | 128 bits | No | None — fully decentralized | Effectively unbounded (local CPU only) |
| UUID v7 (§5) | None, ever | 128 bits | Yes (millisecond-level) | None — fully decentralized | Effectively unbounded |
| Ticket server (§6) | Every call hits the central service | Small int | Strictly, globally | The ticket server itself, mitigated via odd/even pairing | Bounded by ticket server's write rate |
| Snowflake-style (§7) | Only once, at worker-ID assignment | 64 bits | Yes (millisecond-level, per-worker) | None for generation; worker-ID assignment service (e.g., ZooKeeper) needs its own HA story | Per-worker: sequence bits × 1000/sec; scales linearly by adding workers |

The table makes the shape of the whole design space legible at a glance: every row after the first buys *something* (size, sortability, or coordination-freedom) by giving up a different thing, and Snowflake is the row that gets a "yes" or "small/bounded" in the most columns simultaneously — which is exactly why it's the default answer, not because the other rows are wrong.

## 9. Monotonicity and k-Sortability — What "Sortable" Actually Buys You, and What It Doesn't

**Why sortable IDs matter beyond aesthetics.** A B-tree index (the default index structure in virtually every relational database) organizes keys in sorted order across a tree of fixed-size pages. Inserting a new key that sorts to the *end* of the tree is cheap — it just appends into the current rightmost page, splitting that page into two only occasionally, once it fills up. Inserting a key that sorts somewhere in the *middle* is far more expensive on average: it has to locate and rewrite whatever page that key belongs in, which may already be full, forcing a page split there too — and because the insertion points are now scattered randomly across the whole tree instead of concentrated at one edge, far more of the tree's pages end up cold (not recently touched, likely evicted from cache), so more of these inserts miss the page cache and hit disk. A monotonically-increasing ID (Snowflake, UUIDv6/v7) keeps every insert an append into that same hot rightmost page; a fully random ID (UUIDv4) scatters inserts across the whole index unpredictably — the very randomness that made the ID collision-free with no coordination is exactly what hurts index locality here.

**What Snowflake actually guarantees, precisely — and the gap between "sortable" and "monotonic."** Two Snowflake IDs from the *same* worker are strictly increasing, because that worker's own `(timestamp, sequence)` state only ever moves forward (§7.4). But two IDs from *different* workers generated in the same millisecond are only *roughly* ordered by real time — their relative order is decided by the (essentially arbitrary) worker-ID field, not by which one was actually requested microseconds earlier. This is usually called **k-sortability**: IDs are sorted to within a small window (here, one millisecond, across the whole worker fleet), not perfectly totally ordered. For the overwhelming majority of real use cases (pagination, "recent activity" feeds, index locality) this is entirely sufficient — genuine, strict global ordering isn't actually needed, only "close enough that things mostly look chronological and indexes stay append-friendly."

**Where this actually bites, and why it's a deliberate scope boundary, not a bug.** An application that needs a hard guarantee like "ID A was *definitely* issued before ID B whenever A < B, across the whole fleet, with zero exceptions" cannot get that from Snowflake as described — that would require exactly the kind of cross-node coordination (a single shared, agreed-upon sequence) the whole design exists to avoid. Systems that genuinely need this (e.g., a strictly ordered event log used for replay/audit) either accept a coordinated sequencer as a deliberate, scoped-down exception for that one log, or use a scheme like Google Spanner's `TrueTime` — instead of trusting a single clock reading, TrueTime exposes an explicit *uncertainty interval* `[earliest, latest]` around "now," and the system deliberately waits out that interval before committing an event, so that ordering is guaranteed correct rather than merely probable. That's a fundamentally heavier mechanism than anything described here: if an application truly needs hard, total ordering, the right answer is a different, purpose-built system for that one need, not a bigger sequence field bolted onto Snowflake.

## 10. Multi-Datacenter and Failure Considerations

**Worker-ID exhaustion across regions.** If workers are deployed across multiple datacenters/regions, the 10-bit (or however many) worker-ID space is shared globally, not per-region — a fleet of 2,000 workers spread across 3 regions still needs 2,000 *distinct* IDs drawn from one shared 1,024-slot space if using Twitter's original split, which doesn't fit. Real deployments either widen the worker-ID field (shrinking the timestamp or sequence budget, per the §7.1 trade-off) or partition the worker-ID space by region up front (e.g., top 2 bits of the worker-ID field denote the region, the rest denote the worker within that region) — deliberately sacrificing some of the "up to 1,024 workers" headroom for built-in region-awareness, the same kind of explicit, sized trade-off as everything else in §7.1.

**A datacenter goes fully offline.** Because there's no shared state between workers at generation time (§7.2), a datacenter outage doesn't threaten uniqueness at all — the surviving datacenters' workers keep minting IDs exactly as before, since their correctness never depended on the failed datacenter's workers being reachable. This is a meaningfully *stronger* availability story than the ticket-server design (§6), where a lost datacenter housing the ticket server is a real, direct outage for ID generation everywhere, not just locally. The only thing lost is whatever throughput that datacenter's workers were contributing — a capacity reduction, not a correctness or full-availability failure.

**The ZooKeeper (or equivalent) worker-ID-assignment path being briefly unavailable.** Since assignment happens once at worker startup (§7.3), an outage in the coordination system only blocks *new* workers from starting up — it does not stop already-running workers from continuing to generate IDs, since they've already cached their assigned worker ID in memory and never need to re-consult ZooKeeper again during steady-state operation. This is a meaningful, deliberate design property: the coordination dependency is front-loaded to a rare, one-time event (worker startup) rather than sitting on the per-request hot path.

**Clock skew across datacenters.** §7.4's backward-clock guard is a *local*, per-worker safeguard — it does not, and cannot, guarantee that a worker in DC-East with a slightly fast clock and a worker in DC-West with a slightly slow clock produce IDs whose *relative* order matches the true order of the real-world events they represent. NTP typically keeps clocks within single-digit-to-tens-of-milliseconds of each other over the internet, which is fine for "roughly ordered," but sub-millisecond cross-datacenter ordering guarantees are not something wall-clock-based schemes can promise. A design that genuinely needs that has to reach for something heavier (bounded-uncertainty clocks like Spanner's TrueTime, §9, or a coordinated sequencer), not a bigger sequence field.

## 11. System Architecture Summary

```
Client / application service
        │
        ▼
  Load balancer (any healthy worker can serve any request — workers are symmetric)
        │
        ├─► Worker 1 (worker_id=1, in-memory clock+sequence state)
        ├─► Worker 2 (worker_id=2, ...)
        └─► Worker N (worker_id=N, ...)
              │
              └── One-time, at startup only: claim a unique worker_id
                  from ZooKeeper / etcd / a claims table (§7.3)
```

Key characteristics: **no single point of failure once workers are running** (any worker can be lost without affecting any other worker's correctness), **no per-request coordination** (the entire hot path is local CPU + memory), and **horizontal scalability by adding workers**, bounded only by the size of the worker-ID field.

## 12. Other Points Worth Noting

- **Time-based sharding as a side benefit.** Because a Snowflake ID's high bits are literally a timestamp, a database can extract creation time directly from the primary key with no separate column or index — genuinely useful for time-range queries or partition-pruning schemes that key off creation date. This works cleanly here because the "partition key" (time) and the "load-balancing key" (worker ID) are two separate bit-fields within the same ID, not one field forced to serve both purposes the way a purely hash-based key would.
- **Exposing sequential-ish IDs publicly is an information-leak risk, not just an aesthetic one.** A Snowflake ID (or any monotonic scheme) reveals an approximate creation timestamp, and comparing two IDs reveals their approximate relative creation order and, indirectly, relative counts (e.g., "how many orders were placed between these two IDs" becomes roughly inferable) — something a fully random UUID never leaks. Public-facing IDs where this matters (an order ID a competitor could scrape to infer sales volume, e.g.) sometimes deliberately use a random or obfuscated *external* ID that maps internally to a real, sortable Snowflake-style ID, keeping the storage/indexing benefits internal while not leaking them externally.
- **Sequence-overflow busy-waiting (§7.4) is a real, measurable latency spike, not just a theoretical edge case,** for any worker sustaining sequence-bit-exhausting throughput — capacity planning for a Snowflake deployment has to size the sequence-bit width against genuine expected peak QPS per worker, not just average QPS, or that worker will visibly stall at peak load waiting for the millisecond to tick over.
- **The custom epoch choice is a one-way door.** Once IDs have been issued against a chosen epoch, that epoch can never be changed without breaking every previously-issued ID's meaning (and, if the epoch were ever moved *forward*, potentially producing collisions with already-issued timestamps) — teams pick a fixed launch-adjacent epoch specifically to maximize the useful lifetime of the 41 (or however many) timestamp bits before the field overflows, and never revisit it.
- **Testing uniqueness claims.** A Snowflake-style generator's correctness claims are worth verifying under deliberate fault injection rather than trusting the design on paper — force a backward clock jump (via a test hook or `faketime`-style tooling) and confirm the generator actually blocks/errors rather than silently emitting a duplicate, and run many workers concurrently under a shared worker-ID-assignment service to confirm two workers genuinely never race to the same worker ID during a coordinated restart.

## 13. Summary Table

| Goal / Problem | Technique |
|---|---|
| Uniqueness across machines with no coordination | Disjoint worker IDs baked into every ID (Snowflake) or high-entropy random bits (UUID) |
| High availability of the generator itself | No shared state consulted per-request; any worker's failure is isolated (Snowflake) |
| Compact IDs | 64-bit packed integer instead of a 128-bit UUID |
| Roughly time-ordered / index-friendly IDs | Leading timestamp bits (Snowflake, UUIDv6/v7) instead of fully random bits (UUIDv4) |
| Handling per-worker burst throughput | Per-millisecond sequence counter, sized to expected peak QPS |
| Handling clock drift / backward jumps | Explicit backward-clock detection and block/error, never silent wraparound |
| Assigning worker IDs safely | ZooKeeper/etcd sequential ephemeral nodes, or a leased claims table |
| Scaling across datacenters | Partition the worker-ID space by region, or widen the worker-ID field |
| Preventing information leakage from sequential IDs | A separate, random/obfuscated external ID mapped to the internal sortable ID |
| Legacy/simple deployments without a new service | Multi-master DB auto-increment with a fixed offset/step (§4), or a ticket server (§6) |

## 14. Interviewer's Guide: Follow-Up Questions and What to Probe

**On the core trade-off:**
- Why can't an ID scheme be simultaneously fully coordination-free, 64 bits, and strictly globally ordered? What has to give, and which of the three does each approach in §8 sacrifice?
- Why is a UUID 128 bits instead of 64? What would you lose by trying to make it 64 bits?

**On Snowflake specifically (push here — many candidates can name the three fields but haven't reasoned about the failure modes):**
- Walk through exactly what happens if two workers are accidentally assigned the same worker ID during a bad deploy. What's the actual observable failure, and how would you detect it happened after the fact?
- The system clock on a worker jumps backward by 50ms due to an NTP correction. What does a correct implementation do? What does a naive one do wrong, concretely?
- A single worker needs to sustain 10,000 IDs/second. Does the 12-bit sequence field (4,096/ms = 4,096,000/sec) actually bound this, or could you still see stalls? Under what condition?
- If you needed 5,000 concurrent workers instead of 1,024, what would you change, and what do you give up to get it?

**On availability and scaling:**
- Compare the failure blast radius of a ticket-server outage (§6) versus a Snowflake worker outage (§7). Why is the difference structural, not incidental?
- Your worker-ID-assignment service (ZooKeeper) goes down for 10 minutes. What actually breaks during those 10 minutes, and what keeps working?

**On sortability:**
- Are Snowflake IDs totally ordered, or just "roughly" ordered? Give a concrete scenario where two IDs' numeric order doesn't match the true order of the real-world events they represent.
- Why does a monotonically increasing primary key help database index performance, concretely — what's actually happening at the B-tree level with a random key versus a monotonic one?

**Concepts worth raising even if the candidate doesn't bring them up:**
- **Information leakage from sortable IDs** — a public-facing sequential-ish ID reveals approximate volume/timing; ask how they'd avoid that for a customer-facing order ID.
- **Epoch choice as a one-way door** — ask what happens to already-issued IDs if the epoch were ever changed.
- **Multi-region worker-ID space partitioning** — a single flat worker-ID space doesn't obviously extend to multiple datacenters; ask how they'd extend it.
- **The distinction between "coordination needed at generation time" and "coordination needed once, at setup time"** — this is the single idea that explains why Snowflake is preferred over a ticket server despite both ultimately relying on *some* coordination somewhere.

## 15. A Critical Second Look: Gaps Worth Naming Out Loud

- **"Roughly time-ordered" is a real, load-bearing scope limitation, not a footnote.** Any application logic that silently assumes ID order is a perfect proxy for creation-time order across the whole fleet (not just within one worker) will occasionally be wrong, in ways that are easy to miss in testing (single-worker dev environments never exercise the cross-worker interleaving that production, multi-worker traffic does) and only show up under real concurrent multi-worker load.
- **Sequence-overflow blocking is a genuine, if usually small, latency cost that capacity planning has to account for explicitly** — it's easy to describe the sequence field as "4,096 IDs per millisecond, plenty," without stress-testing whether real peak traffic (not average traffic) actually exceeds that on any single worker, especially right after a deploy reduces the worker count temporarily.
- **The backward-clock guard trades a small availability gap for correctness — and that trade should be a stated decision, not a surprise.** A worker that refuses to generate IDs during a backward clock jump is, for that brief window, unavailable — this is the right trade (a duplicate ID is a much worse failure than a brief stall), but it means "the ID generator has 100% uptime" is not quite true as stated; it's "100% uptime except during a rare, self-detected, self-limiting clock anomaly," and that caveat is worth surfacing explicitly rather than letting a reader assume unconditional availability.
- **Worker-ID assignment reintroduces exactly the kind of coordination system (ZooKeeper/etcd, leader election, sessions, leases) that the rest of the design is built to avoid needing on the hot path** — it's a smaller, rarer dependency than the ticket server's per-request one, but it is still a real dependency with its own availability story, and a design review should name it as such rather than describing the overall system as having "no dependencies."
- **None of this addresses what happens to already-issued IDs if a bit-layout change is ever needed later** (e.g., discovering the worker-ID field is too small after all, per the multi-region case in §10) — widening any field is not a live, in-place migration. Two options exist, and neither is free:

  **Option 1 — coordinated cutover within one ID space, using an explicit version tag.** This only works if a version/format marker was reserved as its own dedicated bit-field from the very start — say, bits 62–63, carved out specifically so they're never reinterpreted as anything else. A decoder always reads that tag *first*, then branches to the correct field-width rules for the rest of the bits: version `00` → old widths (e.g., 41/10/12), version `01` → new widths (e.g., 39/13/10, freeing up 3 extra worker-ID bits). Every service that ever decodes an ID has to ship this dual-decode logic at the same time — that's the "coordinated" part. Critically, this option is only available if those version bits were reserved *before* any ID was ever issued; you cannot retrofit a version tag into a scheme that already uses every bit for something else, because there's no unused bit left to signal the format from, and a value's bits alone can't safely disambiguate two different layouts (a number that decodes to a plausible timestamp under the new widths can just as easily decode to a plausible worker-ID+sequence under the old ones — the bits carry no built-in label).

  **Option 2 — don't reinterpret; start a new, independent scheme for new records.** Every already-issued ID keeps its original meaning under the original rules, permanently, and is never re-decoded under a different layout. New records instead get IDs from a separate scheme entirely — e.g., a new product line's orders are issued 128-bit UUIDv7s in a new column, or a new table (`orders_v2`) draws from its own, independent worker-ID pool unrelated to the original 1,024-slot space. "Which scheme produced this ID" is then answered by *context* (which table, column, or service the row came from), not by inspecting the number's bits at all. The cost is permanent: the codebase now supports two ID formats side by side indefinitely, and any code that might encounter either has to know which one it's looking at from context, since the ID itself won't tell it.

  Any real deployment should treat the bit layout as something that might need to change, and should decide *up front* whether it's worth sacrificing a couple of bits as a permanent version tag (buying option 1 later) — because if that reservation isn't made at design time, option 2 is the only path left once the field genuinely runs out of room.

## References

1. Twitter Snowflake (original announcement): https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake
2. Instagram's ID generation (Sharding & IDs at Instagram): https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c
3. Sonyflake: https://github.com/sony/sonyflake
4. Baidu UidGenerator: https://github.com/baidu/uid-generator
5. RFC 4122, A Universally Unique IDentifier (UUID) URN Namespace: https://www.rfc-editor.org/rfc/rfc4122
6. RFC 9562, UUID Version 6/7/8 (updates RFC 4122): https://www.rfc-editor.org/rfc/rfc9562
7. Flickr's Ticket Servers (blog post, "ticket servers: distributed unique primary keys on the cheap"): https://code.flickr.net/2010/02/08/ticket-servers-distributed-unique-primary-keys-on-the-cheap/
8. Apache ZooKeeper (sequential znodes / ephemeral nodes): https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html#Sequence+Nodes+--+Unique+Naming
9. Google Spanner TrueTime (bounded clock uncertainty): https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf
