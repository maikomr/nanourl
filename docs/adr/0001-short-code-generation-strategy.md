# ADR-0001: Short Code Generation Strategy

## Status

Accepted

## Context

We are building a URL shortener. The system is read-heavy and write-light:
redirects (reads) will vastly outnumber URL creations (writes), likely by
several orders of magnitude. This asymmetry matters because it tells us where
to spend complexity budget — optimizing the write path for raw performance is
not the priority; correctness and simplicity are.

Current scale target: a single-server MVP, low-to-moderate traffic, not yet
distributed across multiple write nodes. We need a way to generate a short
code (6–7 characters, base62: `[a-zA-Z0-9]`) for each submitted long URL,
such that:

- Two different long URLs never end up sharing the same short code.
- The mechanism is simple enough to run on one server without introducing
  distributed coordination we don't yet need.

## Decision Drivers

- Avoid collisions reliably.
- Avoid introducing multi-server coordination complexity before it's needed
  (YAGNI).
- Write-path performance is not a primary concern given the read-heavy ratio.
- Prefer database-native guarantees over application-level logic where
  possible, since they're harder to get subtly wrong.

## Decision

Generate a random 7-character base62 string for each new short URL at write
time. Enforce uniqueness with a **unique index/constraint on the short-code
column** at the database level, rather than checking for existence before
inserting.

Write flow:

1. Generate a random 7-character base62 code.
2. Attempt to insert the (code, long URL) pair.
3. If the insert fails due to a uniqueness violation, generate a new code and
   retry.

This is a probabilistic collision-avoidance strategy, not a
collision-by-construction guarantee: with 7 base62 characters the keyspace is
~3.5 trillion, so collision probability stays negligible until the active
key set reaches billions of entries — well beyond MVP scale — but it is not
mathematically impossible the way a counter-based scheme would be.

Uniqueness enforcement happens as part of the same write operation (a single
insert attempt), not as a separate check-then-act step. This avoids a
race condition where two concurrent requests could each check "code doesn't
exist," both get a negative result, and both insert the same code before
either commit completes. Collapsing check-and-insert into one atomic
operation removes that race entirely, rather than trying to make the two
steps individually "safe."

## Alternatives Considered

**Counter-based ID + base62 encoding** — Rejected for now. Gives a true
collision-by-construction guarantee (no probability involved), but requires
a single global source of truth for "the next number." The moment writes
happen from more than one server, that counter becomes either a centralized
bottleneck/single point of failure, or requires pre-allocating ID ranges per
server — a distributed-systems coordination problem we don't need to solve
at current scale. Also produces sequential, guessable codes, though this was
judged a non-issue for this use case.

**Hash the URL (e.g., MD5/SHA-256), truncate to 7 characters** — Rejected.
Deterministic mapping (same input URL always yields the same code) can be
desirable for deduplication, but truncating a cryptographic hash reintroduces
collisions — you still need the same collision handling as random
generation, without gaining anything over it. It also removes the option of
letting the same URL be shortened multiple times into distinct codes, which
may be a product requirement later (e.g. per-link analytics).

**Pre-generated key pool** — Rejected for now, explicitly deferred. A
background process generates and stores unique codes ahead of time; writes
pop a code off the pool rather than generating on demand. This removes
collision-checking from the write's critical path and would be the natural
evolution of this design once we're running multiple write servers and the
active keyspace has grown large enough that random-generation collision
retries become non-trivial. Not adopted now because it adds pool-management
and multi-server-pool-coordination complexity that the current single-server
MVP doesn't need.

## Consequences

**Positive:**

- Simple to implement and reason about; no distributed coordination.
- Correctness is enforced by the database itself (unique constraint), not by
  application logic that could be subtly wrong under concurrency.
- Retries only happen on the rare occasion of an actual collision — the
  common path costs a single write operation.
- Codes are unpredictable/non-sequential, which is a mild positive (no
  information leakage about volume or ordering).

**Trade-offs accepted:**

- Not a hard collision guarantee — relies on the keyspace staying sparse
  relative to its size. This is acceptable at current scale but is a
  ceiling, not a permanent property of the system.
- Occasional wasted writes on collision retries (negligible at current
  scale, but non-zero).
- This strategy does not by itself solve multi-server write coordination;
  if we later shard or scale writes horizontally, this decision will need
  to be revisited (see below).

## When to Revisit

Revisit this decision if any of the following occur:

- We move to multiple servers handling writes concurrently (this decision
  assumes single-writer-node simplicity is still acceptable).
- The active short-code count approaches a scale (hundreds of millions to
  billions of active codes) where collision-retry rates become noticeable.
- A product requirement emerges that needs deterministic code generation
  (e.g., idempotent shortening of the same URL) or sequential/orderable
  codes.

At that point, the pre-generated key pool alternative (above) is the
intended next step, not a redesign from scratch.
