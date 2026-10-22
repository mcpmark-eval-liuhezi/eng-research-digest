# Retry with Exponential Backoff — How Real Open-Source Projects Do It

**Purpose:** Short, citable research note for the incoming backend cohort (three engineers joining next month). Every cohort so far has hand-rolled retry logic badly. This digest looks at how mature open-source projects implement retry with exponential backoff, so new hires cite production patterns instead of inventing them.

**Method:** GitHub code search for the exact phrases `"exponential backoff"` and `"exponential backoff" jitter`; the top results were read directly and are the primary sources below.

---

## 1. temporalio/temporal (Go) — a full, production-grade backoff package

**Files:** `common/backoff/retry.go`, `common/backoff/retrypolicy.go`, `common/backoff/jitter.go`

Temporal ships a dedicated `backoff` package rather than scattering retry loops through call sites. Key takeaways for our codebase:

- **Policy objects, not inline sleeps.** `ExponentialRetryPolicy` is a first-class struct with `initialInterval`, `backoffCoefficient`, `maximumInterval`, `expirationInterval`, and `maximumAttempts`. A separate `Retrier` (`NextBackOff(err)`) owns the state (attempt count, start time) and drives the loop. Callers never hand-write `sleep(2^n)`.
- **The formula is explicit and capped.** `ComputeNextDelay` computes `initialInterval * pow(backoffCoefficient, currentAttempt - 1)`, then caps it with `min(nextInterval, maximumInterval)`, so a runaway multiplier can never produce an unbounded (or overflowing — note the deliberate `MaxInt64` overflow guard and the comment about arm64 vs amd64 wrap behavior) wait.
- **Jitter is mandatory and separate.** `jitter.go` provides `FullJitter` (uniform random from 0 to input) and a coefficient-based `Jitter` helper. The exponential policy applies jitter as `nextInterval*0.8 + rand(0.2*nextInterval)` specifically "to avoid global synchronization" — i.e., hundreds of clients hammering a server in lockstep.
- **Retries stop for good reasons.** `done` is returned when `maximumAttempts` is hit, when the overall `expirationInterval` has elapsed, or when the interval overflows. Expiration-based stopping is the part teams usually forget — it bounds total time spent in a retry storm.
- **Context and error classification are respected.** `ThrottleRetryContext` never retries context cancellation/timeout errors, never retries non-`isRetryable` errors, and uses a *separate, gentler* throttle policy (initial 1s, max 10s) when the server responds `ResourceExhausted` (rate-limit style), taking the *larger* of the two delays.
- **Server hints override the formula.** `MakeBackoffAlgorithm` and the Retry-After-style `requestedDelay` path return a fixed server-requested delay when one is supplied, otherwise fall back to exponential backoff. The clock is injectable (`clock.TimeSource`) so tests don't actually sleep.

## 2. r-lib/httr (R) — Retry-After wins over backoff, terminate conditions are explicit

**Files:** `NEWS.md` (changelog entries for `RETRY()`), the `RETRY()` API

httr's `RETRY()` is the HTTP-client case study. From its changelog and API:

- **Honor the server's clock first.** From 1.3.0: on HTTP 429 with a `retry-after` header, that value is used instead of exponential backoff with jitter, unless it is smaller than `pause_min`. This is the single most-cited rule for HTTP clients: respect the rate-limit signal before you apply your own backoff math.
- **Jitter by default.** `RETRY()`'s default behaviour is exponential backoff *with jitter* (`pause_min`/`pause_cap` bound it, with sub-second `pause_min` explicitly flagged as "use with caution").
- **Termination is configurable, not guessed.** `terminate_on` lets callers declare which status codes stop the retry loop, and since 1.4.0 any successful request terminates by default (`terminate_on_success = FALSE` restores the old behaviour). Retrying a 4xx that can never succeed is the classic hand-rolled bug.
- **Retry on transport errors, too.** `RETRY()` also retries when an error occurs *during* the request (connection dropped), not just on bad status codes.

## 3. ndilieto/uacme (C) — even a minimal client replaced a fixed sleep

**Files:** `NEWS` (changelog)

uacme is an ACMEv2 (Let's Encrypt) certificate client written in plain C — about as minimal as a network client gets:

- **They shipped the fixed-delay bug and then fixed it.** Changelog for 1.7.2 (2022): "exponential backoff for status polling instead of constant 5s delay, to reduce load on server". The original code polled ACME authorization status on a fixed 5-second loop; they moved to exponential backoff specifically because constant polling hammers the CA.
- **Later, they added server-driven delays.** 1.8.0 added "Use Retry-After header when server sends it" — the same server-signal-first principle as httr, adopted for the same reason.

---

## The five rules your retry code should pass

1. **Cap the delay.** Always pair exponential growth with a `maximumInterval`; without it, the 2^n term eventually produces an absurd wait. Temporal even guards integer overflow explicitly.
2. **Add jitter.** Without it, every client that failed at the same moment retries in lockstep and creates a thundering herd. Temporal's comment says it outright: jitter exists "to avoid global synchronization". httr enables it by default.
3. **Bound total time.** Use an `expirationInterval`/deadline (Temporal) or an attempts cap — ideally both — so a retry storm ends.
4. **Never retry the un-retryable.** Classify errors: context cancellation (never), 4xx-type failures (never), rate-limit/`ResourceExhausted` (yes, and back off more), transport errors (yes). Both Temporal and httr make this an explicit, pluggable decision.
5. **Trust server hints over your formula.** HTTP 429 + `Retry-After`, or any explicit server-requested delay, overrides your backoff math. Both httr and uacme implement this and Temporal wires it into its algorithm factory.

**Bonus for reviews:** make the clock injectable so retry code is testable without sleeping (Temporal's `clock.TimeSource`).

---

## Suggested review checklist for new code

- [ ] Is the delay capped (`maximumInterval`)?
- [ ] Is jitter applied?
- [ ] Is there an overall deadline or attempts cap?
- [ ] Are non-retryable errors excluded (explicitly listed)?
- [ ] Is `Retry-After` honored when present?
- [ ] Is the clock injectable for tests?

*Last updated: October 2026.*