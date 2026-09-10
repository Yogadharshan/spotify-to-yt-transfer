# CLAUDE.md — Spotify → YouTube Music Transfer

## Mission

This is a small cash-engine experiment, not a large platform.

Turn the existing Spotify → YouTube Music prototype into a trustworthy public utility with a small production surface area and a plausible path to first revenue.

`SPEC.md` is the product contract. Before product or architectural changes, read it. If it conflicts with a task, surface the conflict rather than inventing a compromise.

---

## 1. Engineering Operating Model

Act as a senior engineer working with an AI implementation partner.

For any non-trivial task:

1. Inspect the relevant code, docs, tests, dependencies, and current behavior.
2. Identify the smallest change that satisfies the requirement.
3. Check `SPEC.md` and relevant decisions.
4. Implement incrementally.
5. Run targeted tests/checks.
6. Inspect the diff.
7. Run broader verification when appropriate.
8. Report what is verified, what is unverified, and any remaining risk.

Never move directly from prompt → large rewrite.

Never claim:
- “works” without verification,
- “tests pass” without running them,
- “the API supports this” without checking current evidence,
- “production-ready” while an essential security/integration dependency is unresolved.

Separate observed facts, verified behavior, assumptions, and proposed design.

When uncertain, inspect the repository, installed dependencies, or current official documentation. Do not invent APIs, files, framework behavior, environment variables, or authentication mechanisms.

---

## 2. Scope

The MVP exists to answer:

1. Can strangers reliably transfer Spotify playlists to YouTube Music?
2. Is transparent/high-fidelity matching meaningfully better?
3. Will some users pay for larger migrations or convenience?

Before adding anything, ask whether it materially improves:

- migration completion,
- matching correctness/trust,
- security/reliability,
- validation of demand/revenue.

If not, do not build it.

Out of MVP:

- mobile/native apps
- multi-provider migration
- social/recommendation features
- full AI assistant
- custom ML models
- subscriptions
- elaborate admin/dashboard systems
- microservices/Kubernetes
- speculative abstractions

Earn the right to add complexity.

---

## 3. Existing Repository First

Before changing behavior, inspect:

- repository structure and docs
- tests and run/build commands
- Spotify integration
- destination integration
- matching logic
- persistence/resume behavior
- authentication assumptions
- current CLI/local entrypoints

Preserve working behavior before replacing it.

When refactoring, separate existing behavior into cleaner boundaries first. Do not redesign unrelated parts at the same time.

---

## 4. Target Architecture

Prefer a simple architecture:

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
Postgres (only necessary metadata)
```

A worker/queue is acceptable only when asynchronous jobs actually require it.

No microservices for MVP.

Provider-specific code stays inside provider adapters. The application layer should use provider-neutral domain objects.

Prefer existing code → small modification → local abstraction → new infrastructure.

---

## 5. Domain and Matching

Use normalized provider-neutral models.

```python
Track:
    title
    artists[]
    album
    duration_ms
    explicit
```

```python
match(source_track, destination_candidates) -> MatchResult
```

`MatchResult` should preserve the destination track, confidence, and reason/evidence.

The matcher must not depend directly on Spotify/YT Music SDK classes.

Matching is the core technical value.

Priorities:

1. Minimize false positives.
2. Prefer strong evidence.
3. Make thresholds explicit/configurable.
4. Keep candidate search bounded.
5. Preserve useful evidence for debugging.

Account for:
- remixes
- live/acoustic versions
- covers
- duplicate titles/artists
- regional/non-English music
- duplicate album names
- explicit/clean versions
- unavailable tracks
- duplicate source tracks

Low-confidence results must not silently become “successful.”

---

## 6. Authentication, Security, and Privacy

Use standard Spotify OAuth with secure state/session handling.

For the destination:

- use the approved supported authorization architecture documented in `docs/DECISIONS.md`;
- do not expose the prototype's `raw_headers.txt` browser-header workflow as a hosted feature;
- never ask users to paste cookies or browser request headers;
- never store raw browser headers on the server;
- never log tokens, cookies, authorization codes, or credentials;
- never put credentials in URLs or source code.

Before implementing authentication, verify current official provider documentation for:

- scopes
- token lifecycle
- redirect URIs
- storage requirements
- trust boundary
- quota/policy constraints

If an essential authorization path is unresolved, stop that implementation path and document the blocker.

Secrets belong only in environment variables or deployment secret storage.

Minimize retained data. Store only necessary job/session metadata. Do not sell or repurpose user music-library data. Avoid indefinite retention and provide cleanup/deletion for retained job data.

---

## 7. Jobs and API

Use explicit job states:

```text
pending
running
completed
partial
failed
cancelled
expired
```

Jobs must have ownership checks so users cannot access another user's job.

Jobs should be resumable/idempotent where practical.

Retries must be bounded and must not create duplicate destination playlists/tracks.

Validate external data at provider boundaries.

Never trust client-provided job state or payment entitlement.

---

## 8. Monetization

The MVP model is:

```text
free useful migration
        ↓
optional support
        +
paid convenience/scale
```

Initial paid hypothesis: one-time purchase around ₹199–₹499, configurable.

Do not build subscriptions in MVP.

Free users get the real matching quality. Paid users buy scale/convenience, not artificially better correctness.

Keep payment logic behind a small interface. Verify payment server-side before granting entitlements.

---

## 9. Cost Controls and AI

This must remain a bounded-cost service.

Configure limits for:

- tracks/job
- concurrent jobs
- provider requests
- retries
- job duration
- stale jobs
- optional AI calls

Do not add an LLM to the normal transfer path unless explicitly required by `SPEC.md`.

If AI later helps ambiguous matching:

```text
max AI calls / transfer
max AI cost / transfer
fallback when budget is exhausted
```

AI is a bounded assist, not an excuse for uncontrolled cost or complexity.

---

## 10. Web UX

The product is a utility, not a dashboard.

Core flow:

```text
Arrive
→ Connect Spotify
→ Select playlists
→ Connect destination
→ Transfer
→ Result
```

Every async operation must have explicit states. No infinite spinners.

Errors should explain:
- what happened,
- what the user can do,
- whether retrying is safe.

Support desktop and mobile web with basic keyboard accessibility and visible focus states.

Use a small reusable component vocabulary; do not turn UI into a design-system project.

---

## 11. Verification

Tests should validate behavior, not implementation details.

Before declaring meaningful work complete:

1. run formatting/lint/type checks applicable to the repo;
2. run targeted tests;
3. run relevant integration tests;
4. use a controlled real-provider test when provider behavior changed;
5. inspect the diff;
6. check auth/ownership/security paths;
7. check retry/idempotency behavior;
8. check limits and payment verification when relevant;
9. inspect logs for leaked secrets.

When fixing a bug:

```text
reproduce
→ regression test
→ fix root cause
→ rerun verification
```

Never weaken/delete tests merely to make the suite green.

Do not make real provider calls in ordinary unit tests.

---

## 12. External Research

For changing platform behavior, use current official documentation, especially for:

- Spotify API
- Google OAuth
- YouTube Data API
- YouTube policies/quota
- payment provider behavior
- deployment platform behavior

Prefer primary sources.

Record durable external constraints and major architectural choices in `docs/DECISIONS.md`.

Do not build around undocumented behavior unless explicitly marked as experimental.

---

## 13. Dependencies and Changes

Before adding a dependency, answer:

1. What concrete problem does it solve?
2. Can existing dependencies solve it?
3. Is the maintenance/security/cost tradeoff justified?
4. Does it shorten the path to MVP validation?

For every meaningful change:

```text
identify problem
→ identify smallest affected surface
→ reuse existing code
→ implement smallest correct change
→ test
→ inspect diff
→ update durable docs only if the decision changed
```

Do not bundle unrelated cleanup or refactors into feature work.

---

## 14. Documentation and Decisions

Keep concerns separated:

- `SPEC.md` — product requirements and acceptance criteria
- `docs/DECISIONS.md` — durable architectural/product decisions
- `docs/ROADMAP.md` — current MVP priorities
- code — ordinary implementation details

For a significant decision, record:

```text
# Decision: <title>

Date:
Status:
Context:
Decision:
Why:
Consequences:
```

Do not create a decision record for trivial implementation choices.

---

## 15. Launch and Stop Conditions

Do not call the MVP public-ready until:

- real transfer has been tested;
- failure and duplicate-retry paths work;
- authentication/security has been reviewed;
- no secrets are exposed;
- privacy/terms/contact pages exist;
- environment/deployment/rollback steps are documented;
- basic monitoring/health checks work.

After launch, evaluate:

```text
traffic
→ activation
→ transfer success
→ match quality
→ payment
```

If evidence is weak, prefer parking the project over adding features.

The purpose of the MVP is to determine whether this small utility deserves more time.
