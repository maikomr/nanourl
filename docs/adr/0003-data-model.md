# ADR: Data Model

## Status

Accepted

## Context

The data model is not being designed from scratch, it needs to support commitments already made in two prior decisions:

- The write-path decision (short code generation) relies on a database-level uniqueness constraint on the short code, enforced atomically on insert.
- The read-path decision (redirect strategy) depends on the core mapping being read-heavy, essentially write-once, and cache-friendly, and commits to accurate click analytics as a hard requirement for business customers.

These two constraints pull in different directions if modeled naively: the core mapping wants to be rarely written and stable, while analytics wants to be written on effectively every request (every redirect is a potential click to record). The data model needs to reconcile that tension rather than let one undermine the other.

## Decision Drivers

- Preserve the read-heavy, low-contention properties the read-path ADR depends on.
- Support the write-path ADR's uniqueness-constraint mechanism.
- Support accurate, business-usable click analytics without turning every redirect into a write against the core table.
- Keep the initial model simple; defer fields that aren't needed yet rather than guessing at future requirements.

## Decision

Use **two separate tables**, split by access pattern rather than by subject matter alone:

### `urls` (core mapping)

| Field        | Notes                                                                                                                                                                                       |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `short_code` | Unique, indexed. Required for the write-path's insert-and-let-the-DB-reject-duplicates mechanism to function, without a `UNIQUE` constraint there is nothing for that mechanism to enforce. |
| `long_url`   | The destination URL.                                                                                                                                                                        |
| `created_at` | Audit trail; also the natural home for future expiry logic if added later.                                                                                                                  |

This table is written once per short URL and read on essentially every redirect. No analytics data lives here.

### `click_events` (analytics, event log)

| Field | Notes |
| ----- | ----- |

| `event_id` | Unique identifier per click event. |
| `short_code` | Which link was clicked; links back to `urls`. |
| `timestamp` | When the click occurred. |
| `referrer` | Referring page/domain, where available. |
| `user_agent` | Raw user-agent string, stored as-is. Device type, browser, and OS can be derived from this later without needing to re-collect data, so parsing is deliberately deferred rather than done at write time. |

One row is written per click, decoupled entirely from the `urls` table. Writing a click event happens **after** the redirect response has already been sent to the user (asynchronously), the same reasoning as the cache-population decision in the read-path ADR: the user isn't waiting on analytics, so it doesn't belong on the response's critical path.

An event-log shape (one row per click) was chosen over a single running counter per short code, because a bare count can't answer the questions a business customer actually asks of analytics, _when_ clicks happened, _where_ they came from, trends over time. A counter column would also reintroduce the exact "write on every read" problem being avoided, just on a different field.

## Alternatives Considered

**A single table with a `click_count` column on `urls`**, Rejected. This would turn every redirect into a write against the same table the read path depends on being cache-friendly and low-contention, directly undermining the read-path ADR. The failure mode isn't hypothetical, it's the same table being asked to serve two incompatible access patterns at once.

**Running counter table, one row per `short_code` with an incrementing count**, Rejected for now. Cheaper to store and query than an event log, and doesn't touch the `urls` table, but doesn't give businesses the richer picture (timing, source, device) that analytics is expected to provide. An event log can still answer "what's the total count" via aggregation, so nothing is lost by choosing the richer shape.

**Storing parsed `device_type`/`browser`/`os` fields directly instead of raw `user_agent`**, Deferred, not rejected outright. Raw `user_agent` is strictly more flexible: it can be parsed into those fields at any point later without needing to have collected them in advance. Parsing was left out of the initial model to keep the first version simple.

**Additional analytics fields considered and explicitly deferred:**
`ip_address`/derived geolocation, UTM parameters, `is_bot` flagging. Not implemented now, but flagged as real, known gaps rather than overlooked ones, in particular, `is_bot` matters because unfiltered bot/crawler traffic (link-preview fetchers, security scanners, etc.) can quietly inflate click counts and undermine the trustworthiness of the analytics being sold to business customers. `ip_address` also carries a privacy/ compliance consideration (e.g. GDPR treats raw IP as personal data) that will need a deliberate decision, not a default, whenever it's picked back up.

## Consequences

**Positive:**

- The core `urls` table keeps the properties the read-path ADR relies on: rarely written, cache-friendly, low contention.
- Click analytics can be as rich as the business use case needs, without design pressure to keep the core table minimal.
- The two tables can be scaled, indexed, and even stored differently in the future without affecting each other.
- Deferred fields are captured here as known gaps, not silently forgotten.

**Trade-offs accepted:**

- Two tables to maintain instead of one; a click's full context requires a join or a second lookup rather than living on the same row as the URL.
- The initial analytics schema is intentionally incomplete (no bot filtering, no geolocation, no campaign attribution), click counts should not yet be treated as fully trustworthy for business reporting until at least bot filtering is addressed.

## When to Revisit

Revisit this decision if any of the following occur:

- Bot/crawler traffic is found to be materially inflating click counts, making `is_bot` filtering necessary before analytics can be trusted.
- A business requirement emerges for geolocation or campaign-source reporting, requiring `ip_address`/geolocation or UTM fields to be added, at which point the IP/privacy handling decision needs to be made deliberately.
- Short URL expiry becomes a real feature, requiring an `expires_at` (or similar) field on `urls`.
- Query patterns on `click_events` at scale (e.g. aggregating totals per link) reveal a need for a supplementary rollup/counter table alongside the event log, rather than computing aggregates from raw events every time.
