# Designing a Distributed Web Crawler

## 1. Overview and Goals

A web crawler (also called a spider or bot) systematically browses the web, downloading pages, extracting their content and outbound links, and feeding both into downstream systems — typically a search index, an archive, or an analytics pipeline. This design targets a crawler built to continuously discover and refresh web pages and articles at internet scale, starting from a set of seed URLs.

**Functional requirements:**

- Given a set of seed URLs, discover and download reachable pages by following links.
- Extract page content (text, title, metadata) and outbound links from each page.
- Avoid downloading the same page twice (exact duplicates) and avoid indexing near-identical content (near duplicates).
- Respect each site's crawling policy (`robots.txt`, rate limits, crawl-delay).
- Periodically re-crawl pages to keep content fresh, prioritizing pages that change often or matter more.
- Support extensibility to new content types (HTML, PDF, RSS/Atom feeds, sitemaps) without rearchitecting the pipeline.

**Non-functional requirements:**

- **Scalability**: crawl billions of pages; horizontally scale by adding crawler workers.
- **Politeness**: never overwhelm a single origin server with concurrent requests.
- **Robustness**: tolerate malformed HTML, unresponsive servers, redirect loops, and deliberately adversarial content (crawler traps) without crashing or stalling.
- **Freshness vs. efficiency trade-off**: re-crawl important/fast-changing pages more often than static ones, without re-crawling everything constantly.
- **Extensibility**: new parsers, new storage backends, and new filtering rules should plug in without touching the core pipeline.
- **Manageable operational cost**: bandwidth, storage, and compute all scale with the size of the web being crawled, so efficiency (avoiding wasted fetches) matters as much as raw throughput.

These goals conflict in the usual ways: crawling fast conflicts with politeness (a single host can only take so much traffic); freshness conflicts with efficiency (re-crawling everything constantly wastes bandwidth on pages that rarely change); and robustness against traps conflicts with completeness (being too aggressive about cutting off crawl paths risks missing real content). The rest of this document is largely about resolving these tensions with concrete mechanisms.

## 2. Basic Data Model

Two core records flow through the system:

```
URL Record:
  <url, normalized_url, discovered_at, last_crawled_at, priority,
   host, status: {pending, in_progress, crawled, failed, blocked}>

Page Record:
  <url, content_hash, fetched_at, http_status, content_type,
   extracted_text, title, metadata, outbound_links[], content_blob_ref>
```

Keeping these as two separate records (a lightweight scheduling record and a heavier content record) matters: the frontier only ever needs to touch the small URL record to decide what to crawl next, while the bulky page content is written once and read rarely (mostly by downstream indexing), so they belong in different storage tiers with very different access patterns.

## 3. High-Level Architecture

```
Seed URLs
   │
   ▼
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│ URL Frontier │◄────►│  Politeness  │      │ robots.txt  │
│ (priority +  │      │  Enforcer    │◄────►│   Cache     │
│  per-host    │      └──────────────┘      └─────────────┘
│  queues)     │
└──────┬───────┘
       │ dequeue (host, url)
       ▼
┌─────────────┐      ┌──────────────┐
│  DNS Cache/  │      │  HTML/HTTP   │
│  Resolver    │◄────►│  Fetcher     │
└─────────────┘      └──────┬───────┘
                             ▼
                     ┌───────────────┐
                     │ Content Parser │
                     │ (per content-  │
                     │  type plugin)  │
                     └───────┬───────┘
                             ▼
              ┌──────────────┴───────────────┐
              ▼                               ▼
    ┌──────────────────┐           ┌────────────────────┐
    │ Duplicate Content │           │   URL Extractor +   │
    │     Detector      │           │   Normalizer +      │
    │ (exact + near-dup) │           │   Filter + Seen-URL │
    └─────────┬─────────┘           │       Check         │
              ▼                     └──────────┬──────────┘
    ┌──────────────────┐                       │ new URLs
    │   Content Store    │                      ▼
    │ (raw + extracted)  │            back into URL Frontier
    └────────────────────┘
```

A crawl cycle: pull a URL that's both high-priority and safe to fetch right now (politeness-wise) off the frontier, resolve its host, fetch the content, parse it, check whether it's a duplicate, extract new links from it, filter and dedup those links, and push the survivors back onto the frontier — while the fetched content itself is persisted for downstream use.

## 4. URL Frontier — Deep Dive

The frontier is the queue of URLs waiting to be crawled, but a plain FIFO queue fails immediately: whichever host happens to have the most links queued would get hammered with back-to-back requests the moment several of its URLs reach the front together, violating politeness. The standard solution (used by the classic Mercator crawler design) splits the frontier into two queue tiers:

**Front queues — priority.** Several queues (e.g., a fixed number, say `F`), each holding URLs of one priority tier. A prioritizer scores each URL (page rank estimate, historical update frequency, how the URL was discovered — e.g., linked from many pages vs. one obscure page) and assigns it to a front queue. A weighted round-robin picks which front queue to pull from next, so high-priority URLs get pulled more often without starving low-priority ones entirely.

**Back queues — politeness.** A fixed number of back queues (e.g., `B`, chosen to be larger than the expected number of concurrently-fetchable hosts), each holding URLs for *one single host at a time*, plus a routing table mapping `host → back queue`. A separate thread pulls a URL from a front queue (per the priority policy above) and pushes it into whichever back queue currently serves that host — creating a fresh back queue for a host not yet mapped to one. A **heap keyed by "earliest time this back queue may be tapped again"** decides which back queue a fetcher thread pulls from next: pop the queue whose wait time has expired, dequeue its next URL, and re-insert that queue into the heap with a new release time computed as `now + politeness_delay_for_this_host` (commonly a multiple of the last fetch's response time, so a slow/overloaded host automatically gets throttled harder).

**Worked example.** Say `B = 3` back queues currently mapped as `{queue1: example.com, queue2: news.example, queue3: (empty)}`, and `politeness_delay = 2s`. A fetcher pulls `queue1`'s next URL at `t=0`, and `queue1` is reinserted into the heap with release time `t=2s`. Meanwhile `queue2` might already be ready at `t=0.5s` (its own last fetch was earlier), so the *next* fetcher thread pulls from `queue2` instead of blocking on `queue1` — different hosts' cooldowns run independently and in parallel, while no single host is ever hit twice within its own 2-second window. This is what gives high aggregate throughput (many hosts fetched concurrently) without violating politeness for any individual host (§5).

When a back queue empties (a host has no more pending URLs), it's returned to a free pool and un-mapped from that host, ready to be assigned to whichever host next needs one — this bounds memory use to roughly `B` active hosts at a time rather than one queue per host ever seen.

## 5. Politeness and `robots.txt` — Deep Dive

**Fetching and caching `robots.txt`.** Before crawling any URL on a host for the first time, the crawler fetches `https://host/robots.txt`, parses its `Disallow`/`Allow` rules (matched against the crawler's own user-agent string, falling back to the wildcard `*` rules), and caches the parsed result keyed by host with a TTL (commonly 24 hours) — refetched periodically since a site's policy can change. A URL matching a `Disallow` rule for this crawler's user-agent is dropped before ever reaching the frontier.

**`Crawl-delay` and rate limiting.** Some `robots.txt` files specify an explicit `Crawl-delay` (seconds between requests to that host); when present, it overrides the frontier's own default politeness delay (§4) for that host, respecting the site owner's stated preference even if it's more conservative than the crawler would otherwise choose.

**Per-host concurrency cap.** Independent of delay timing, the crawler also caps how many *concurrent* in-flight requests are allowed to one host at a time (commonly 1, sometimes a small constant) — the back-queue design (§4) naturally enforces this, since a host maps to exactly one back queue, and only one fetcher thread pulls from a given back queue's ready URL at a time.

**Identifying the crawler.** Every request carries a descriptive `User-Agent` header (e.g., `MyCrawler/1.0 (+https://mysite.com/crawler-info)`) so site operators can identify the crawler in their logs, look up its purpose, and add specific rules for it in their `robots.txt` if desired — an operational courtesy that also makes debugging complaints from webmasters tractable.

## 6. DNS Resolution and Caching

A DNS lookup on every single fetch would add real, repeated latency and hammer DNS resolvers at crawler scale. The crawler keeps its own DNS cache (host → IP, with the DNS record's own TTL respected, plus a floor and ceiling on how long an entry is trusted regardless of a misconfigured TTL) — separate from any OS-level or library-level DNS cache, since the crawler's request volume to any given host vastly exceeds what a generic client-side cache is tuned for. A negative cache (recently-failed lookups) is kept too, so a host with broken DNS doesn't get retried on every single one of its queued URLs.

## 7. Fetching — Deep Dive

**Timeouts and retries.** Every fetch has both a connect timeout and a total-response timeout; a fetch that exceeds either is treated as a failure, logged, and retried with exponential backoff up to a small retry cap (e.g., 3 attempts) before the URL is marked `failed` and set aside (not retried again for a cooldown period, to avoid wasting a disproportionate share of that host's politeness budget on a URL that's reliably broken).

**Redirects.** HTTP redirects (301/302/307) are followed up to a bounded hop count (e.g., 5); exceeding it is treated as a redirect loop and the chain is abandoned. Each redirect target is itself checked against the seen-URL filter (§9) and `robots.txt` (§5) before being followed — a redirect can just as easily point off-site to a completely different, disallowed host.

**Conditional fetching for freshness.** On re-crawl, the fetcher sends `If-Modified-Since` (using the page's last-fetched timestamp) or `If-None-Match` (using a cached `ETag`) headers; a `304 Not Modified` response lets the crawler skip re-downloading and re-parsing content that the origin server itself confirms hasn't changed — a major bandwidth and processing saving on re-crawls of static pages.

**Content-type and size filtering.** The fetcher checks the response's `Content-Type` and, ideally, a `HEAD` request's `Content-Length` before committing to a full download, skipping content types outside the crawler's supported set (§14) and pages beyond a configured size cap (protecting against a single enormous file monopolizing a fetcher thread and that host's politeness window).

## 8. Duplicate Content Detection — Deep Dive

Two different problems hide under "duplicate content," and they need different techniques:

**Exact duplicates** (the same bytes, reachable via two different URLs — a common occurrence with URL parameters, mirrors, or syndicated copies). Detected cheaply: hash the fetched content (e.g., SHA-256) and check that hash against a store of previously-seen content hashes. A match means this exact content has already been crawled and indexed under some other URL — skip re-storing and re-indexing it, though the new URL can still be recorded as an alias pointing at the existing content record.

**Near-duplicates** (articles republished with a different header/footer/ad block, or lightly edited copies) aren't caught by exact hashing at all, since a single differing byte changes the whole hash. This needs a **similarity-preserving** fingerprint — **SimHash** is the standard technique:

1. Break the page's text into overlapping word shingles (e.g., 3-word sequences) or just weighted terms.
2. Hash each shingle/term to a fixed-width bit vector (e.g., 64 bits) using an ordinary hash function.
3. For each bit position across all these hash vectors, tally how many hashes have a 1 there versus a 0; if 1s outnumber 0s at that position, the final SimHash's bit is set to 1, otherwise 0. This produces one 64-bit fingerprint per page.
4. Compare two pages' SimHash fingerprints by **Hamming distance** (number of differing bits). Near-duplicate pages — even with different surrounding boilerplate — end up with fingerprints differing by only a handful of bits, while genuinely different pages differ by roughly half their bits (around 32 of 64) on average.

**Worked example.** Two articles share 90% of their body text but one has a different navigation menu and ad block. Their word-shingle sets overlap heavily but not completely; because SimHash's bit-majority construction is dominated by the shared, common shingles, the two fingerprints typically end up 3–5 bits apart out of 64 — comfortably under a chosen threshold (e.g., ≤ 8 bits) for "near-duplicate," while two unrelated pages' fingerprints land close to 32 bits apart, nowhere near that threshold.

At web scale, checking every new page's SimHash against every previously-seen one pairwise is infeasible — this is handled with a technique that buckets fingerprints by several different subsets of their bits (multiple hash tables, each keyed by a different slice of the 64 bits), so only fingerprints sharing a bucket with the new page's fingerprint under at least one slice ever need an actual Hamming-distance comparison, cutting the comparison set from "the whole web" down to a small candidate list per page.

## 9. URL Deduplication — the "Seen URL" Problem

Before a newly-extracted link is added to the frontier, the crawler needs to know: **has this exact URL already been crawled or already queued?** At billions of URLs, a naive "check a database/hash-set for every single discovered link" is enormous in both storage and lookup cost, especially since the overwhelming majority of links extracted from any given page point to URLs already seen (site navigation, repeated boilerplate links, etc.).

A **space-efficient approximate membership filter** (a bit array checked via several hash functions, tuned to have no false negatives and a small, bounded false-positive rate) sits in front of the authoritative URL store: check the filter first; if it says "definitely not seen," the URL is certainly new and gets added to the frontier and the filter immediately. If it says "maybe seen," a slower, authoritative lookup against the real URL store confirms or refutes it before deciding. The false-positive cost here is asymmetric and worth naming explicitly: a false positive means a URL that was actually new gets treated as "maybe already seen" and pays for one extra authoritative lookup — never a correctness bug, just a small, bounded amount of wasted lookup work, exactly the same trade-off profile as it has anywhere else it's used. A false negative would be a genuine bug (silently re-crawling forever), which is why the filter's underlying construction must guarantee zero false negatives by design.

Since a single machine's memory can't hold a filter (or the authoritative store) for the whole seen-URL set at internet scale, this layer is **partitioned across many machines by hashing the URL itself** — each worker owns a slice of the URL space and answers "seen or not" only for URLs that hash into its slice, so a "have I seen this?" check is routed to exactly one specific worker rather than broadcast to all of them.

## 10. URL Normalization and Filtering

**Normalization** turns superficially different URLs that point at the same resource into one canonical form before the seen-URL check runs, or the same page gets crawled repeatedly under URLs that only differ cosmetically: lowercase the scheme and host, resolve relative paths (`../`), strip default ports (`:80` on `http://`), remove URL fragments (`#section`), sort or strip tracking query parameters (`utm_source`, session IDs) according to a configurable rule set, and remove trailing slashes consistently.

**Filtering** drops URLs before they're even normalized/deduped: non-HTTP(S) schemes, disallowed file extensions (binary media types outside the crawl's scope), URLs matching a domain blacklist, and — critically — **crawler-trap detection**: pages that generate infinite, ever-changing URLs on the fly (an auto-generated calendar with an infinite "next month" link, or session-ID-bearing URLs that regenerate a "new" URL on every visit). Mitigations: cap the maximum crawl depth from a seed for any single host, cap the maximum number of URLs crawled per host per time window, and detect URL patterns with unbounded, monotonically-changing segments (e.g., a date or numeric ID climbing without bound across many discovered links from the same page template) and throttle or drop that pattern once a suspicious threshold is crossed.

## 11. Distributed Architecture and Partitioning

Crawling is embarrassingly parallel across *different hosts*, but politeness (§5) requires that all URLs for *one* host be handled by whichever single component is enforcing that host's rate limit — splitting one host's crawl across multiple independent workers with no shared state would let them collectively hammer that host, each unaware of what the others just sent.

The standard solution: **partition work by hashing the host** (not the full URL) across crawler worker nodes, so every URL for a given host is always routed to the same worker, and that worker's own local frontier/politeness enforcement (§4) is sufficient — no cross-worker coordination is needed to keep one host's request rate in check. When a worker's assigned host-shares grow (new hosts discovered) or a worker is added/removed, the host-to-worker mapping is recomputed and a small fraction of hosts move to a different worker — the frontier state for a re-assigned host (its queued URLs, its last-fetch timestamps) is handed off to the new owner rather than reset, so politeness state (the "don't fetch again before this time" clock) isn't accidentally lost during a rebalance.

## 12. Storage Layer

Two clearly different access patterns call for two different storage tiers:

- **URL/scheduling metadata** (the URL record from §2): small, frequently updated (status changes constantly as URLs move through pending → in_progress → crawled), and needs fast point lookups by URL and range/priority queries for frontier scheduling. This fits a store optimized for small records with frequent updates and indexed lookups.
- **Raw and extracted page content** (the page record from §2): large (full HTML bodies, extracted text), written once per crawl and rarely if ever updated in place, read mostly by downstream batch consumers (an indexing pipeline) rather than by the crawler itself. This fits cheap, high-throughput blob storage, with the page record's lightweight metadata (hashes, timestamps, links) stored separately from the bulky raw content so metadata-only queries (like the content-hash lookups in §8) don't need to touch the large blobs at all.

**Link graph.** Outbound links extracted from each page are also persisted as edges (`source_url → target_url`) in their own store, separate from both of the above — this becomes the input to any downstream link-analysis (importance scoring, discovering which pages are heavily linked-to, informing the frontier's prioritizer in §4) and is written far more heavily than it's read during normal crawling, favoring an append-optimized storage layout.

## 13. Freshness and Re-crawl Scheduling

Re-crawling every page on a fixed schedule wastes bandwidth on pages that never change and under-serves pages that change constantly (breaking news, frequently-edited wiki pages). Instead, each URL's re-crawl priority is a function of:

- **Observed change frequency**: track, across the last several crawls of a URL, how often its content hash (§8) actually changed; a page that changes on nearly every visit gets a short re-crawl interval, one that's been byte-identical for the last 10 crawls gets a long one, growing multiplicatively (similar in spirit to exponential backoff, but in the opposite direction — backing *off* re-crawling a stable page rather than backing off a failing request).
- **Estimated importance**: pages with many inbound links (from the link graph, §12) or high historical traffic get a floor on how infrequently they can be re-crawled, regardless of how static they've looked so far, since missing a real update on an important page is costlier than on an obscure one.
- **Freshness deadline enforcement**: the frontier's prioritizer (§4) treats a URL whose re-crawl interval has elapsed as newly eligible, feeding it back into a front queue at a priority reflecting both of the factors above.

## 14. Extensibility — Pluggable Content Parsers

The parser stage (§3) is a plugin interface keyed by `Content-Type`: an HTML parser extracts visible text, title, meta tags, and outbound links; a PDF parser extracts text and any embedded links; a sitemap/RSS parser extracts a list of URLs directly rather than page content. Adding support for a new content type means implementing this interface and registering it against its MIME type(s) — nothing else in the pipeline (frontier, dedup, storage) needs to know or care what content type it's handling, since every parser's job is to normalize its input down to the same `Page Record` shape.

## 15. Fault Tolerance and Recovery

- **Checkpointed frontier state**: URL records' state transitions (§2) are persisted durably as they happen, not just held in a worker's memory — a crashed worker's in-flight URLs (marked `in_progress` but never completed) are detected via a staleness timeout and reset back to `pending` so another worker picks them up, rather than being silently lost.
- **Idempotent processing**: since a URL can be re-attempted after a crash or timeout, every stage (fetch, parse, dedup check, store) is designed so repeating it with the same input produces the same end state — re-fetching and re-storing a page that was actually already stored just overwrites with an identical result rather than creating a duplicate record.
- **Dead-letter handling**: a URL that fails repeatedly (past the retry cap, §7) is moved to a separate failed-URL store rather than endlessly retried, so a handful of permanently broken links can't quietly consume an ever-growing share of a host's crawl budget.
- **Backpressure**: if downstream storage or the parsing stage falls behind the fetch rate, the frontier throttles how many new fetches it dispatches rather than letting fetched-but-unprocessed content pile up unboundedly in memory.

## 16. Back-of-the-Envelope Capacity Estimate

Assume a target of crawling 1 billion pages per month:

- **Throughput**: `1,000,000,000 / (30 × 24 × 3600) ≈ 386` pages/second sustained average (real deployments provision well above average to handle bursts and re-crawl backlogs).
- **Average page size**: assuming ~500 KB average raw HTML (many pages are much smaller, some much larger) → `386 × 500 KB ≈ 193 MB/s` sustained inbound bandwidth, roughly 500 TB/month of raw content fetched.
- **Metadata storage**: a URL record at a few hundred bytes each, across several billion discovered URLs (far more URLs are discovered than are ever worth crawling) — on the order of a few TB for scheduling metadata alone.
- **Content storage**: 500 TB/month of raw content, before any compression or extracted-text-only storage, which is typically far smaller than the raw HTML and is what most downstream consumers actually want.

These numbers immediately justify the architectural choices above: single-machine storage is out of the question at this scale (hence partitioning, §11–§12), bandwidth this large must be spread across many fetcher machines (further reinforcing per-host partitioning, §11), and the seen-URL/dedup structures (§8–§9) must stay memory-efficient given the sheer number of URLs involved.

## 17. Monitoring and Operational Concerns

Key operational signals: crawl rate (pages/sec, against the target throughput above); error rate by category (timeouts, DNS failures, HTTP error codes, robots-disallowed) broken out per host, to catch a newly-misbehaving or newly-blocking site quickly; frontier queue depth per priority tier (a growing backlog signals the fetch/parse pipeline is falling behind discovery); freshness lag (average and worst-case time since last successful crawl, weighted by page importance); and per-host politeness-budget utilization (to catch a misconfigured crawl-delay or a host that's being crawled more aggressively than intended).

## 18. Security, Legal, and Ethical Considerations

- **Respect `robots.txt` and `crawl-delay`** as a hard constraint, not a suggestion (§5) — ignoring it risks the crawler's IP ranges being blocked outright, and is the baseline ethical norm for automated crawling.
- **Honor page-level opt-outs**: a `noindex` meta tag or `X-Robots-Tag` header should suppress that page from downstream indexing even if it was fetched (fetching to check the tag is allowed; indexing its content isn't, once the tag says so).
- **Avoid effectively-DoS-like behavior**: the concurrency caps and rate limits (§5, §11) exist as much for legal/reputational safety as for technical politeness — a crawler that overwhelms a small site's server can cause real, attributable harm.
- **Sensitive content handling**: pages behind authentication, or explicitly marked private via standard mechanisms, should never be fetched or stored, regardless of whether a stray link to them was discovered.

## 19. Summary Table

| Goal / Problem | Technique |
|---|---|
| High throughput without overwhelming any one host | Front/back queue URL frontier with per-host politeness delay |
| Respecting site crawling policy | `robots.txt` parsing + cache, `Crawl-delay` |
| Avoiding repeat DNS lookups at scale | Local DNS cache with TTL + negative caching |
| Avoiding re-downloading unchanged pages | Conditional GET (`If-Modified-Since` / `ETag`) |
| Detecting exact duplicate content | Content hashing (e.g., SHA-256) |
| Detecting near-duplicate content | SimHash + Hamming-distance bucketing |
| Avoiding re-crawling already-seen URLs at scale | Approximate membership filter + partitioned authoritative store |
| Avoiding crawler traps | Depth caps, per-host URL caps, URL-pattern anomaly detection |
| Distributing load across workers | Partition by host hash, so one host's traffic stays on one worker |
| Keeping content fresh without wasting bandwidth | Adaptive re-crawl interval from observed change frequency + importance |
| Extending to new content types | Pluggable, content-type-keyed parser interface |
| Surviving worker crashes | Checkpointed frontier state + idempotent processing + staleness-based requeue |

## 20. Interviewer's Guide: Follow-Up Questions

If you're using this design as the basis for a system-design interview, use these to probe past a candidate's first pass — most prepared candidates will name the frontier, politeness, and dedup pieces; the questions below test whether they understand the mechanics and trade-offs underneath.

**On the URL frontier and politeness:**
- Why can't a single priority queue (ordered purely by importance) satisfy both prioritization and politeness at once? Walk through what goes wrong.
- If a host's `Crawl-delay` in `robots.txt` says 30 seconds but the crawler's default politeness delay is 2 seconds, which wins, and why?
- How would you handle a host that becomes slow (not down, just responding in 10+ seconds) — should its politeness delay change, and how would the system know to adjust it?

**On duplicate detection:**
- Why doesn't a simple content hash catch near-duplicate articles? What has to change about the technique, and why does that change work?
- Two completely unrelated pages happen to produce SimHash fingerprints only 2 bits apart. What does the crawler do, and is this a correctness bug or an expected, bounded risk?
- At what stage would you compute the content hash — before or after stripping boilerplate (nav bars, ads)? What breaks if you get this order wrong?

**On URL dedup and scaling:**
- Why is a plain hash set insufficient at billions of URLs, and what's the actual trade-off the approximate filter buys you?
- A false positive in the seen-URL filter says "maybe already seen" for a URL that's actually brand new. What's the worst-case consequence — could this cause the crawler to *miss* real content permanently? (Good candidates should recognize the fallback authoritative check prevents that; the filter can only ever waste work, never silently drop a URL.)
- How would you partition the seen-URL structure across machines, and what has to be true about the partitioning function for lookups to be routed correctly?

**On crawler traps and robustness:**
- Describe a concrete crawler trap and how each of the mitigations named would (or wouldn't) catch it.
- A legitimate site has a genuinely huge but finite number of pages (a large e-commerce catalog) that happens to look, structurally, a lot like a trap pattern. How do you avoid falsely throttling a real site?

**On distributed architecture:**
- Why partition by host rather than by full URL? What would break if you partitioned by URL instead?
- A worker is rebalanced and a host's URLs move to a new owner. What has to be transferred along with the URLs themselves for politeness to keep working correctly after the move?

**On freshness:**
- Design the re-crawl interval function more precisely: what inputs should it take, and what should happen to a page that suddenly starts changing after a long stable period?
- How would you avoid a scenario where a large number of pages all become "due" for re-crawl at the same time, causing a load spike?

**Concepts worth raising even if the candidate doesn't bring them up:**
- **Storage tiering**: treating small, frequently-updated scheduling metadata and large, rarely-updated page content as the same storage problem is a common early mistake — ask why they might want to split these.
- **Idempotency under retries**: what happens if a page is fetched successfully but the process crashes before the "crawled" status is recorded — does it get lost, or double-processed?
- **Legal/ethical scope**: does the design account for `noindex`, authentication walls, and reputational risk of overly-aggressive crawling, or does it only optimize for throughput?
- **Extensibility cost**: how much of the pipeline would need to change to support crawling something structurally different, like a JSON API feed instead of HTML pages?
- **Back-of-envelope sanity check**: ask the candidate to size the seen-URL filter and the frontier's queue memory for a specific target scale (e.g., 10 billion discovered URLs) — a candidate who can't translate the design into rough numbers may not have internalized why certain components (approximate filters, partitioning) are necessary rather than incidental choices.
