# Spotify → YouTube Music Transfer — Cash Engine V0.2

## 0. Mission

Build the smallest trustworthy web product that transfers a user's Spotify playlists to YouTube Music.

Primary objective: validate whether the existing transfer engine can become a small, low-maintenance cash engine.

Success is not "launch a startup." Success is:

> A stranger can arrive, transfer a playlist, understand the result, and optionally pay for additional convenience.

Keep the scope narrow. Optimize for first revenue and learning, not completeness.

---

## 1. Product Constitution

These are hard product constraints, not suggestions.

1. Free core migration must be genuinely useful.
2. Never intentionally degrade free matching to force payment.
3. Be honest about match confidence and failed tracks.
4. Minimize collection and retention of user data.
5. Never sell personal playlist data.
6. Avoid dark patterns, fake scarcity, forced subscriptions, or hostage-style paywalls.
7. Paid features should charge for scale, convenience, priority, or additional capabilities.
8. The user should be able to leave the product without friction.
9. Do not build the business around unsupported credential extraction or authentication workarounds.
10. Prefer sustainable value creation over attention extraction.

Economic model:

```text
free useful core
      ↓
optional support + paid convenience
      ↓
sustainable operation
```

The product is a utility, not an engagement trap.

---

## 2. Target User

A person who wants to move from Spotify to YouTube Music and does not want to recreate playlists manually.

Main job-to-be-done:

> "Move my music library with as little manual work as possible, and tell me what did not transfer."

Primary user characteristics:

- understands playlists and basic web authentication
- values speed and reliability more than customization
- may only use the service once
- does not want to create another account unless necessary

---

## 3. V0 User Flow

### 3.1 Landing

Show:

- clear value proposition: move Spotify playlists to YouTube Music
- primary CTA: Transfer my playlists
- short, factual privacy statement
- simple explanation of how transfer works
- no account creation unless technically required

The page should communicate the product in under 10 seconds.

### 3.2 Spotify connection

- authenticate with Spotify using supported OAuth
- validate OAuth state
- fetch user playlists
- display playlist names and track counts
- allow selection

Liked Songs can remain a future-capable domain concept, but must not expand V0 unless already reliable.

### 3.3 Playlist selection

Display:

- playlist name
- track count
- selection control

Actions:

- Transfer selected
- Transfer all within free limits

Clearly show free-tier limits before the transfer starts.

### 3.4 Destination authentication

Use a legitimate, supportable authorization mechanism.

Hard rule:

> The hosted product must not ask users to paste cookies, browser headers, request headers, or raw credentials into the website.

Do not expose the repository's current `raw_headers.txt` workaround as hosted functionality.

If the destination operation cannot be implemented through a supportable authorization mechanism, document the blocker instead of silently shipping an insecure workaround.

### 3.5 Transfer job

Show real progress:

- playlist name
- tracks processed / total
- matched count
- unmatched count
- current state
- estimated remaining work only when reliable

Transfers should be resumable/idempotent where practical.

### 3.6 Result

Show:

- source track count
- matched tracks
- unmatched tracks
- match percentage
- destination playlist link when available
- useful failure explanation
- review/downloadable unmatched list if practical

Never imply 100% accuracy unless independently verified.

---

## 4. UX State Contract

Every primary operation must explicitly model these states:

```text
idle
loading
authenticating
queued
running
partial
completed
cancelled
failed
expired
rate_limited
```

Every non-success state must answer:

1. What happened?
2. What can the user do now?
3. Will retrying create duplicates?

Never leave users staring at an ambiguous spinner.

### Error language

Errors should be written for users first and developers second.

Bad:

> 401 from provider adapter.

Better:

> YouTube Music authorization expired. Reconnect and try again. Your completed tracks will not be added twice.

Technical detail may be available behind an expandable diagnostic section where appropriate.

---

## 5. Core Architecture

Use a boring monolith first.

Preferred shape:

```text
Web UI
  ↓
Application API
  ↓
Transfer Service
  ├── Spotify Adapter
  ├── Matching Engine
  └── YouTube Music Adapter
  ↓
Postgres (only necessary job/session metadata)
```

A worker/queue may be added only if transfer jobs require asynchronous processing in deployment. Do not introduce distributed infrastructure prematurely.

### Provider-neutral track model

Normalize source and destination tracks into a common representation:

```python
Track:
    title: str
    artists: list[str]
    album: str | None
    duration_ms: int | None
    explicit: bool | None
```

Matching interface:

```python
match(source_track, destination_candidates) -> MatchResult
```

Minimum result:

```python
MatchResult:
    destination_track
    confidence
    reason
```

The matching engine must not depend directly on Spotify-specific or YouTube-specific classes.

---

## 6. Matching Engine

Matching quality is the core technical value of the product.

Start from the repository's existing matching behavior rather than rewriting it from scratch.

Initial strategy:

1. Search albums/artists where useful.
2. Inspect a bounded candidate set.
3. Prefer exact/strong title matches.
4. Consider artist similarity.
5. Use fallback song + artist search.
6. Use fuzzy matching only as a fallback.

Candidate scoring may eventually consider:

- title similarity
- artist similarity
- album similarity
- duration similarity
- version/remix/live/acoustic signals
- explicit/version signals when available

Hard rule:

> Prefer a false negative over a confidently wrong match.

Thresholds must be explicit/configurable.

Preserve enough evidence to debug why a match was chosen.

### Required test categories

- exact studio tracks
- remixes
- live versions
- acoustic versions
- covers
- songs with identical titles
- artists with identical/similar names
- regional/non-English music
- albums with duplicate names
- explicit vs clean versions
- unavailable/deleted destination tracks
- duplicate source tracks

### Optional AI assistance inside the product

Do not introduce an LLM into the core matching path by default.

If later experiments use AI for ambiguous matches, it must be:

- an explicit fallback only
- confidence-bounded
- cost-bounded
- observable
- deterministic enough to audit
- never allowed to silently override strong provider evidence

An AI call should not become a hidden dependency for ordinary transfers.

---

## 7. Free Tier

The free tier must be useful enough to drive adoption.

Initial suggested limits, configurable rather than scattered through code:

- up to 3 playlists OR 500 tracks per transfer batch
- standard processing priority
- standard result report

Do not artificially reduce match quality for free users.

A user should know the limit before spending time connecting services.

---

## 8. Paid Tier

Initial paid experiment: one-time purchase, approximately ₹199.

Paid capability may include:

- larger/unlimited playlist volume within reasonable infrastructure limits
- larger libraries
- priority processing
- retries for failed jobs
- detailed unmatched-track report
- repeat transfers

Exact price and limits must be configuration-driven.

Do not build subscriptions for V0.

Do not build a complex billing platform before demand exists.

Payment provider integration should be isolated behind a tiny interface so it can be replaced.

Paid entitlement must be verified server-side. Never trust a client-side `is_paid` flag.

---

## 9. Optional Support / Dāna Layer

After a successful free transfer, optionally show:

> This tool is free. If it saved you time, support development.

Suggested one-time amounts:

- ₹49
- ₹99
- ₹299

This is secondary to the paid power tier.

Do not repeatedly nag users after they decline.

---

## 10. Data Handling

Collect the minimum information needed to execute the transfer.

Store only necessary metadata, such as:

```text
job_id
created_at
source
destination
playlist_name
track_count
matched_count
failed_count
status
```

Do not persist complete playlist contents indefinitely.

Use a documented retention policy for temporary tokens, job state, and result data.

Never log:

- OAuth tokens
- cookies
- browser headers
- authorization codes
- provider credentials
- unnecessary complete private playlist contents

Secrets must come from environment variables or deployment secret storage.

Provide a simple data-deletion path for retained user/job data.

---

## 11. Security Baseline

Required before public launch:

- OAuth state validation
- secure session handling
- CSRF protection where applicable
- no tokens in URLs
- no credentials in frontend logs
- no secrets committed to Git
- rate limiting on transfer endpoints
- input validation at provider boundaries
- job ownership enforcement
- safe error messages
- structured logs with sensitive fields redacted
- HTTPS
- secure cookie configuration
- server-side payment entitlement checks
- provider scopes limited to what is actually needed
- dependency/security scanning in CI where practical

Security decisions that are uncertain must be documented instead of guessed.

---

## 12. Reliability / Cost Guardrails

The cash engine must not silently become an expensive automation service.

Set configurable ceilings for:

- maximum tracks per job
- maximum concurrent jobs
- maximum provider requests per job where practical
- maximum retries
- job timeout
- stale-job cleanup
- optional AI fallback calls

Every external-provider failure must have bounded retry behavior.

Use exponential backoff where appropriate.

Prevent duplicate destination writes on ordinary retries.

Before enabling any AI service, define:

```text
cost per transfer ceiling
maximum AI calls per transfer
fallback behavior when budget is exhausted
```

The default transfer path should work without AI.

---

## 13. Observability

Track only the metrics needed for validation:

- landing visits
- Spotify connections
- transfer starts
- transfer completions
- match counts
- unmatched counts
- support clicks
- payment attempts
- successful payments
- revenue
- provider/API failure rate
- transfer duration

Useful ratios:

```text
successful_transfers / visitors
paying_users / successful_transfers
revenue / successful_transfer
average_match_rate
```

Do not add a heavyweight analytics stack for V0.

Analytics must not capture private playlist contents or authentication material.

---

## 14. SEO / Acquisition

Target exact migration intent:

- Spotify to YouTube Music
- transfer Spotify playlist to YouTube Music
- move Spotify playlists
- transfer Spotify liked songs
- Spotify playlist converter

The GitHub repo should clearly link to the hosted utility.

V0 may include one landing page plus a few static explanatory pages.

No content-management system.

---

## 15. Trust / Legal Surface

Before public launch, provide simple, readable pages for:

- Privacy
- Terms
- Contact/support
- What data is accessed and why
- How transfer works
- Limitations / unsupported cases

Do not copy legal language blindly. Use service-appropriate text and have it reviewed when necessary.

The product must not claim official affiliation with Spotify, Google, or YouTube unless such affiliation exists.

Use accurate wording such as "unofficial" where applicable and legally appropriate.

---

## 16. UX Constraints

Treat the website as a utility, not a SaaS dashboard.

Primary flow:

```text
Arrive
→ Connect Spotify
→ Select playlists
→ Connect destination
→ Transfer
→ Result
```

Every screen should answer:

- What do I do next?
- What is happening now?
- What succeeded?
- What failed?

The product must work on desktop and mobile web.

Accessibility baseline:

- keyboard navigable primary flow
- visible focus states
- sufficient text contrast
- semantic controls
- clear labels for forms and buttons
- status updates understandable to screen readers where applicable

Do not add a large design system. Use a small consistent component vocabulary.

---

## 17. AI-Assisted Product Engineering

This repository is intentionally built with AI coding agents such as Claude Code. AI assistance is allowed, but generated code is not trusted by default.

The AI is an implementation partner, not the decision-maker.

### AI work loop

For every non-trivial change:

```text
inspect
→ understand
→ propose smallest change
→ implement
→ test
→ inspect diff
→ verify behavior
→ record meaningful decision
```

Claude must not:

- invent APIs or library behavior without checking the repository/docs
- silently replace working code with a preferred architecture
- add dependencies without justification
- claim tests passed without actually running them
- claim an integration works without exercising it or clearly marking it unverified
- create placeholder security mechanisms and present them as production-ready
- hide uncertainty

### AI-generated code quality gate

Every generated change must satisfy:

1. It has an identifiable user/product reason.
2. It fits the architecture in this spec.
3. It has appropriate tests or a concrete reason tests are not practical.
4. It passes formatter/linter/type checks applicable to the repo.
5. Its diff is reviewed for accidental scope expansion.
6. Security-sensitive code receives explicit review.

### Context discipline

Before editing, inspect the smallest relevant set of files.

Do not dump the entire codebase into context unless necessary.

Prefer repository search, targeted file reads, and existing tests/docs.

When a task is ambiguous, infer from `SPEC.md`, existing behavior, and tests before introducing a new convention.

### Decision memory

Record durable architectural/product decisions in `docs/DECISIONS.md` using a short ADR-like format:

```text
# Decision: <title>

Date:
Status: accepted | superseded
Context:
Decision:
Why:
Consequences:
```

Do not create an ADR for trivial implementation choices.

---

## 18. Technical Scope Rules

Do NOT build in V0:

- mobile app
- native desktop app
- social features
- recommendations
- full AI assistant
- multi-provider migration
- subscriptions
- complex user profiles
- elaborate design system
- microservices
- Kubernetes
- custom ML model
- large admin systems
- speculative abstractions

Build only what is necessary to validate usage and first revenue.

---

## 19. Deployment

The system must be deployable with a simple production setup.

Requirements:

- one frontend/application service
- one database if needed
- simple worker only if necessary
- environment-based configuration
- health endpoint
- production error handling
- HTTPS
- secure secret management
- database migrations if a database is used
- documented environment variables
- basic rollback procedure

Keep operational costs small and visible.

---

## 20. Testing Requirements

### Unit tests

- track normalization
- string normalization
- candidate scoring
- confidence thresholds
- provider response parsing
- free-tier limits
- entitlement checks

### Integration tests

- Spotify playlist retrieval using mocked responses
- candidate search using mocked destination responses
- transfer job lifecycle
- retry/idempotency behavior
- ownership checks
- payment verification using provider mocks

### End-to-end validation

Use a small real test account/playlist before public release.

Record:

- source track count
- matches
- false positives
- false negatives
- transfer duration
- provider errors

Automated tests must never use real production credentials.

---

## 21. Definition of Done

V0 is done when all are true:

1. A new user can understand the product without explanation.
2. Spotify authentication works through supported OAuth.
3. Destination authentication is supportable for the required hosted flow.
4. A user can select a playlist and start a transfer.
5. Progress is visible and honest.
6. Completed transfers are not duplicated by ordinary retries.
7. Match results distinguish matched and unmatched tracks.
8. User job ownership is enforced.
9. Secrets are not present in source control or logs.
10. Free-tier limits work server-side.
11. Paid entitlement, if enabled, is verified server-side.
12. Basic privacy/terms/contact surfaces exist.
13. Tests and checks pass.
14. A real transfer has been manually validated.
15. Deployment and rollback steps are documented.

---

## 22. Validation Targets

Suggested first real-world targets:

```text
100 strangers
→ 20+ successful transfers
→ 2+ people voluntarily paying/supporting
```

These are decision thresholds, not promises.

Interpretation:

- low traffic → acquisition problem
- high traffic, low transfer starts → UX/trust problem
- high starts, low completion → provider/matching problem
- high completion, poor match rate → matching problem
- high completion, zero payment → weak monetization/value perception
- successful transfers + payment → evidence to continue

---

## 23. Kill Condition

Do not rescue the project through sunk-cost reasoning.

After the first real-user validation cycle, evaluate:

```text
traffic
→ activation
→ transfer success
→ match quality
→ payment
```

If evidence remains weak after a reasonable acquisition attempt, park the project and preserve the code.

Do not solve lack of demand by adding more features.

---

## 24. Implementation Order

1. Inspect existing repository and preserve useful logic.
2. Establish provider-neutral domain models.
3. Extract/refactor transfer engine from CLI/local assumptions.
4. Build robust tests around matching.
5. Resolve supported destination authentication before public transfers.
6. Add minimal API/job lifecycle.
7. Build minimal web UI.
8. Add free limits.
9. Add support/payment CTA.
10. Add lightweight telemetry.
11. Add trust/legal surfaces.
12. Deploy privately.
13. Test real accounts/playlists.
14. Fix highest-impact failure modes only.
15. Public launch.

---

## 25. Engineering Decision Rule

When deciding between implementations, prefer the one that:

- is simpler
- is easier to delete/replace
- has fewer external dependencies
- is easier to test
- makes provider boundaries explicit
- preserves user privacy
- gets to real-user validation sooner

Do not optimize for theoretical scale before real demand exists.
