# ADR: Redirect (Read Path) Strategy

## Status

Accepted

## Context

The system is read-heavy and write-light: redirects will be hit far more
often than short URLs are created. This is the path where request volume and
latency actually matter, in contrast to the write path (see the short code
generation ADR), where correctness was the priority and raw performance was
not.

A short code's mapping to a long URL is effectively immutable once written —
it almost never changes after creation. Real-world link-sharing traffic also
follows a power-law distribution: a small number of links receive a large
share of total clicks (e.g. links shared widely on social media), while the
majority of links receive very little traffic. Both properties shape this
decision.

The product is intended to serve businesses as well as individuals, and
click analytics (accurate click counts per short link) is treated as a hard
requirement for the business use case, not an optional nice-to-have.

## Decision Drivers

- Accurate click analytics must not be silently lost.
- Read-path latency matters, given this is the high-traffic path.
- Avoid caching strategy that requires manually deciding which links are
  "important" — let actual traffic patterns decide.
- Avoid adding unnecessary latency to the user-facing response.

## Decision

**Redirect status code:** Use **302 (Found)**, not 301 (Moved Permanently),
for all redirects.

301 tells browsers the redirect is permanent, allowing the browser to cache
the mapping locally and skip contacting our server on subsequent visits to
the same short link. This would silently undercount clicks — the redirect
would still work correctly for the user, but our server would never see
repeat visits, breaking click analytics without any visible error. 302
ensures every click is a server hit, which analytics depends on. This
sacrifices a browser-level caching optimization in exchange for accurate
data, which is judged the right trade given the business use case.

**Lookup path:** Use a **cache-aside (lazy-loading) pattern** with an
in-memory cache (e.g. Redis) in front of the primary database.

1. On a redirect request, check the cache for the short code.
2. **Cache hit:** return the long URL immediately.
3. **Cache miss:** query the primary database for the long URL, return the
   response to the user immediately, then populate the cache
   asynchronously (not blocking the response). The cache write doesn't
   affect what's being returned to the user, so it doesn't need to sit on
   the response's critical path.

**Eviction policy:** Configure the cache with an LRU (or Redis's
approximated-LRU) eviction policy, explicitly set. This is a required
configuration step, not a default behavior — Redis defaults to
`noeviction`, which would reject new writes once memory fills rather than
evicting old entries. LRU naturally matches the power-law traffic shape:
frequently-hit ("hot") links stay in cache because they're continually
re-accessed, while rarely-hit ("cold") links age out on their own, without
needing to manually classify links by importance.

**Concurrent cache misses:** If multiple requests for the same uncached
short code arrive close together, each may independently query the database
and write the same value into the cache. This is treated as an accepted,
benign race — unlike the write-path race (see the short code generation
ADR), there is no conflicting data here: every concurrent request retrieves
and writes the *same* long URL, so the only cost is a small amount of
duplicated read work, not a correctness risk.

## Alternatives Considered

**301 (Moved Permanently) redirect** — Rejected. Technically "correct" in
the sense that the mapping is in fact permanent, but it hands control of
future requests to the browser's local cache, silently starving click
analytics. Rejected specifically because of the business analytics
requirement.

**No cache / always query the primary database** — Rejected. Given the
read-heavy traffic profile, routing every redirect through the primary
database is unnecessary load on the system's most frequently hit path, for
data that essentially never changes once written.

**Synchronous cache population (populate Redis before responding to the
user)** — Rejected. Would add the latency of a cache write to every
cache-miss request, even though the cache write has no bearing on the
correctness of the response already retrieved from the database.

## Consequences

**Positive:**

- Click analytics remain accurate; every redirect is observable server-side.
- Read latency stays low for the (traffic-weighted) majority of requests,
  since hot links stay resident in cache.
- Cache sizing and contents adapt automatically to real traffic patterns,
  with no manual curation needed.
- Cache misses don't add cache-write latency to the user-facing response.

**Trade-offs accepted:**

- 302 forfeits a browser-level caching optimization that 301 would have
  provided; every click involves a round trip to our server. Judged
  acceptable — the added latency is negligible, and the analytics
  requirement outweighs it.
- Cold links (long tail) will always incur a cache-miss database read on
  first (and infrequent subsequent) access; this is inherent to
  cache-aside and considered acceptable given how rarely those links are
  hit.
- Occasional duplicated cache writes on concurrent cache misses for the
  same short code; harmless, but not eliminated.

## When to Revisit

Revisit this decision if any of the following occur:

- Analytics requirements change such that browser-level caching (301)
  becomes acceptable, or click tracking moves to a mechanism that doesn't
  depend on hitting our server on every request (e.g. client-side beacons).
- Cache hit rate data shows the LRU policy isn't actually matching real
  traffic shape well (e.g. sizing needs adjustment, or a different eviction
  policy performs better in practice).
- Read traffic grows to a scale where a single cache instance/tier is no
  longer sufficient, prompting a move to a distributed or multi-tier
  caching setup.
