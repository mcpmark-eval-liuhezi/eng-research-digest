# Sources — Retry/Backoff Research

This note lists exactly what was searched and what was cited, so the digest is auditable like code.

## Search phrases used

Both searches were run against GitHub code search (exact-phrase, quoted):

1. `"exponential backoff"` — ~2.3M results. Top hits: `ndilieto/uacme` (file `NEWS`), `landing-ai/ade-python` (file `api.md`).
2. `"exponential backoff" jitter` — ~334K results. Top hits: `informatikr/hedis` (file `TODO.md`), `r-lib/httr` (file `NEWS.md`).

Supplementary searches used to find implementation code and a Discussions-enabled project:

- `"exponential backoff" timeout retry path:*.py`
- `"exponential backoff" jitter timeout` (top hits: `r-lib/httr` `NEWS.md`, `celestiaorg/op-alt-da` `API.md`)
- `"exponential backoff" reconnect retry max` (top hits: `albertogeniola/MerossIot` `README.md`, `Bewinxed/river.ts` `SKILL.md`)
- `"exponential backoff" retry language:javascript` (top hits: `ateferi/student_polling_system` `retry.js`, `Kong/unirest-nodejs` `index.js`)
- `"exponential backoff" language:java` (top hits: `booksbyus/zguide` `fragments/Java/heartbeats.java`, `OpenTSDB/opentsdb` `src/uid/UniqueId.java`)
- `"exponential backoff" org:temporalio` — top hits: `temporalio/temporal` `common/backoff/retry.go`, `common/rpc/grpc.go`
- `"exponential backoff" org:supabase` — top hits: `supabase/supabase` `.agents/skills/dev-toolbar-review/SKILL.md`, `examples/prompts/use-realtime.md`

## Repositories cited in the digest

The digest covers three repositories, all found via the searches above:

| Repository | Language | Files read | What it contributes to the digest |
|---|---|---|---|---|
| [temporalio/temporal](https://github.com/temporalio/temporal) | Go | `common/backoff/retry.go`, `common/backoff/retrypolicy.go`, `common/backoff/jitter.go` | Policy-object design, capped formula, mandatory jitter, expiration/attempt bounds, error classification, throttle policy, injectable clock, server-requested delay override |
| [r-lib/httr](https://github.com/r-lib/httr) | R | `NEWS.md`, `RETRY()` API/changelog | Retry-After precedence over backoff, jitter on by default, `terminate_on`/`terminate_on_success` termination control, retrying transport errors |
| [ndilieto/uacme](https://github.com/ndilieto/uacme) | C | `NEWS` (changelog) | Real-world migration from a fixed 5s poll loop to exponential backoff to protect the ACME CA; later addition of `Retry-After` handling |

Other top search hits noted but not cited in depth (no Discussions, or source read but not needed for the digest): `informatikr/hedis` (Haskell Redis client — its `TODO.md` explicitly lists "Exponential backoff with jitter to avoid connection storms" as a roadmap item), `landing-ai/ade-python`, `celestiaorg/op-alt-da`, `albertogeniola/MerossIot`, `Bewinxed/river.ts`, `ateferi/student_polling_system`, `Kong/unirest-nodejs`, `booksbyus/zguide`, `OpenTSDB/opentsdb`, `supabase/supabase`.

## Discussion read

**Project:** `temporalio/temporal` — the only repository found in the search results with GitHub Discussions enabled (the first several candidates checked — `ndilieto/uacme`, `landing-ai/ade-python`, `informatikr/hedis`, `r-lib/httr`, `celestiaorg/op-alt-da`, `albertogeniola/MerossIot`, `Bewinxed/river.ts`, `booksbyus/zguide`, `Kong/unirest-nodejs` — had no Discussions at all).

**Discussion title:** *"[Feature Request] Provide details of previous Activity Task failure"* — https://github.com/temporalio/temporal/discussions/12165

**Why it is relevant:** it asks for the retry context to be observable — users want to know, when an Activity Task is being retried, what caused the previous attempt to fail (a returned error vs. a `startToCloseTimeout` vs. a `heartbeatTimeout`). It connects directly to the digest's point that retries must be classifiable and that the failure reason should be surfaced, not swallowed.

**Comments on that thread (2):**
1. A maintainer pointed to the API surface that would need to change first (`temporal/api/.../request_response.proto`) and that it must be discussed with the server team.
2. A follow-up noted the request had come up before and linked the server-side feature request `temporalio/temporal#3517`.

Takeaway for our team: when you retry an operation, preserve and expose *why* the previous attempt failed — and expect the same retry-observability ask to surface repeatedly from users.

## Notes

- Primary sources were read directly from the repositories, not from blog posts or tutorials.
- The repositories listed in the table above are the cited sources for `backoff-research.md`.
