# MVP Roadmap

## Purpose

Build the smallest production-usable version of the music migration tool that can validate three things:

1. People can successfully transfer Spotify playlists to YouTube Music.
2. The product provides a meaningfully more trustworthy migration experience through transparent matching and reporting.
3. A small subset of users will pay for larger migrations or additional convenience.

This roadmap is intentionally narrow. The MVP is a validation instrument, not the final product.

---

## MVP Success Criteria

The MVP is considered successful when a stranger can:

1. Open the website.
2. Understand the value proposition.
3. Connect Spotify.
4. Select a playlist/library within the allowed limit.
5. Authorize the destination account through the supported official flow.
6. Start a transfer.
7. See useful progress.
8. Receive a clear result showing matched, uncertain, and unmatched tracks.
9. Successfully find the resulting playlist in YouTube Music.
10. Optionally pay for a larger transfer.

The product must be usable without developer assistance.

---

# Phase 0 — Repository and Technical Baseline

### Goal

Understand and stabilize the existing migration engine before introducing the web product.

### Tasks

- [ ] Read the existing repository and identify the current Spotify ingestion flow.
- [ ] Identify the existing YouTube Music integration.
- [ ] Identify the current matching implementation.
- [ ] Identify persistence/resume behavior.
- [ ] Identify all secrets and authentication mechanisms.
- [ ] Remove or isolate prototype-only credential handling from the public-service architecture.
- [ ] Establish the minimum test/build commands.
- [ ] Create `.env.example`.
- [ ] Confirm Git ignores all secrets and local credentials.
- [ ] Record important findings in `docs/ARCHITECTURE.md` and `docs/DECISIONS.md`.

### Exit Criteria

- The current codebase can be run and understood.
- There is a clear separation between source-provider logic, matching logic, and destination-provider logic.
- No secret or browser session credential is committed.
- The implementation path for official destination authorization is documented.

---

# Phase 1 — Core Migration Engine

### Goal

Make the migration engine reliable independently of the web UI.

### Tasks

- [ ] Define a normalized `Track` representation.
- [ ] Define a normalized `Playlist` representation.
- [ ] Implement provider-independent matching interfaces.
- [ ] Preserve the existing matching logic where it is useful.
- [ ] Add deterministic normalization for:
  - [ ] title
  - [ ] artist
  - [ ] album
  - [ ] duration
  - [ ] common version/remix/live suffixes
- [ ] Implement candidate scoring.
- [ ] Define confidence levels:
  - [ ] high confidence
  - [ ] uncertain
  - [ ] unmatched
- [ ] Add global caching for reusable track → destination matches where permitted.
- [ ] Add bounded retries.
- [ ] Make transfer jobs resumable/idempotent.
- [ ] Prevent duplicate playlist items when retrying.
- [ ] Produce a machine-readable transfer result.

### Required Result Model

```text
TransferResult
├── total
├── matched
├── uncertain
├── unmatched
├── failed
└── items[]
       ├── source_track
       ├── destination_track
       ├── status
       ├── confidence
       └── reason
```

### Exit Criteria

A test playlist can be processed repeatedly without corrupting the destination playlist, and the engine accurately reports what happened.

---

# Phase 2 — Official Authentication and Provider Integration

### Goal

Create the minimum secure hosted architecture.

### Tasks

- [ ] Implement Spotify OAuth.
- [ ] Implement the supported Google identity/authentication flow required by the product.
- [ ] Implement the required YouTube authorization/scopes.
- [ ] Separate site identity from destination-service authorization.
- [ ] Store tokens securely.
- [ ] Never expose provider secrets to the browser.
- [ ] Never log OAuth tokens or authorization headers.
- [ ] Handle token expiration/refresh.
- [ ] Handle user denial and revoked access.
- [ ] Document required OAuth redirect URIs.
- [ ] Document provider quotas and limits.
- [ ] Implement quota-aware request handling.

### Important Constraint

Do not replace the existing browser-header prototype by simply storing users' raw YouTube Music request headers on the server.

The hosted MVP must use the approved authentication architecture defined in `docs/DECISIONS.md`.

### Exit Criteria

A test user can authorize both services through the intended flows and complete a transfer without manual developer intervention.

---

# Phase 3 — Minimal Web Product

### Goal

Expose the migration engine through the simplest usable interface.

### Pages

#### Landing Page

Use the locked landing-page copy.

Primary CTA:

**Transfer My Music**

#### Connect

- [ ] Spotify connection
- [ ] Destination authorization
- [ ] Clear explanation of requested permissions

#### Playlist Selection

- [ ] Show user's playlists.
- [ ] Show track counts.
- [ ] Allow multiple selection.
- [ ] Enforce free-tier limits.
- [ ] Handle empty/error states.

#### Transfer

- [ ] Show playlist name.
- [ ] Show progress.
- [ ] Show tracks processed.
- [ ] Show current status.
- [ ] Allow safe cancellation where practical.

#### Result

Show:

```text
2,418 analyzed
2,356 matched
43 uncertain
19 unavailable
```

- [ ] Show successful destination playlist.
- [ ] Link/open destination where supported.
- [ ] Allow viewing unmatched/uncertain tracks.
- [ ] Allow retrying eligible failures.

### Exit Criteria

A non-technical user can complete the whole journey without instructions from the developer.

---

# Phase 4 — Free Tier and Monetization Experiment

### Goal

Test willingness to pay without building a full billing system.

### Free Offer

Start conservatively.

Example:

```text
Small migrations
Up to ~100 tracks per transfer
```

The exact limit should be configurable rather than hardcoded.

### Paid Offer

Initial hypothesis:

```text
Large Library Transfer
One-time payment

Higher track limit
More playlists
Priority processing
Detailed results
Retry failed transfers
```

Target test range:

```text
₹199–₹499 one time
```

Do not optimize pricing before observing real behavior.

### Tasks

- [ ] Add a payment provider.
- [ ] Create a one-time payment product.
- [ ] Verify payment server-side.
- [ ] Never trust client-side payment status.
- [ ] Grant entitlements only after verified payment.
- [ ] Handle failed/cancelled payments.
- [ ] Provide receipt/status information.
- [ ] Include optional support/dāna contribution.

### Exit Criteria

A real user can complete a small free transfer and a larger paid transfer from beginning to end.

---

# Phase 5 — Reliability and Security Hardening

### Goal

Make the MVP safe enough for real strangers.

### Tasks

- [ ] Validate all user-controlled input.
- [ ] Add rate limiting.
- [ ] Add job concurrency limits.
- [ ] Add provider quota protection.
- [ ] Add request timeouts.
- [ ] Add bounded retries with backoff.
- [ ] Prevent duplicate jobs.
- [ ] Prevent duplicate destination playlist items.
- [ ] Handle partial failures.
- [ ] Ensure secrets are excluded from logs.
- [ ] Ensure temporary user data is deleted according to policy.
- [ ] Add basic abuse protection.
- [ ] Add privacy-policy link.
- [ ] Add terms/basic service disclaimer.
- [ ] Verify production error handling does not expose internals.

### Exit Criteria

Common failures produce controlled user-facing behavior rather than crashes, leaked secrets, or corrupted transfers.

---

# Phase 6 — Observability

### Goal

Know whether the product works without inspecting user accounts manually.

### Track

```text
landing_views
spotify_connections
destination_authorizations
transfer_started
transfer_completed
transfer_failed
tracks_processed
tracks_matched
tracks_uncertain
tracks_unmatched
payment_started
payment_completed
```

### Derived Metrics

```text
activation_rate
transfer_completion_rate
match_rate
uncertain_rate
payment_conversion
revenue_per_transfer
provider_api_usage
cost_per_transfer
```

Never log raw playlists or unnecessary personal library contents.

### Exit Criteria

You can answer:

- How many people started?
- How many completed?
- What percentage of tracks matched?
- Where do transfers fail?
- How much does a transfer cost to operate?
- Did anyone pay?

---

# Phase 7 — Real-User Validation

### Goal

Validate the business before expanding the product.

### Initial Target

Get approximately:

```text
100 real visitors
20+ completed transfers
2+ paying or supporting users
```

These are working thresholds, not laws.

### Observe

Look for:

- users failing authentication
- users misunderstanding the product
- poor matching
- incorrect versions
- users abandoning large transfers
- support/payment behavior
- repeated requests for a missing capability

### Do Not Build Yet

Do not add features simply because one person asks for them.

Only promote a request into the roadmap when evidence shows it is common or materially affects conversion/retention/revenue.

---

# MVP Completion Gate

The MVP is ready for public testing only when all are true:

- [ ] Official authentication path works.
- [ ] Core Spotify → YouTube Music transfer works end-to-end.
- [ ] Matching produces confidence/status information.
- [ ] Large failures are reported rather than hidden.
- [ ] Free limits work.
- [ ] Payment flow works for a one-time purchase.
- [ ] Secrets are protected.
- [ ] Rate limiting and quota protection exist.
- [ ] Basic monitoring exists.
- [ ] Privacy/data-retention behavior is documented.
- [ ] Landing page is live.
- [ ] A stranger can complete the process without developer help.

---

# Explicitly Out of MVP

Do not build these unless MVP evidence demands them:

- [ ] Mobile app
- [ ] Browser extension
- [ ] Desktop application
- [ ] Apple Music integration
- [ ] Tidal integration
- [ ] Deezer integration
- [ ] Amazon Music integration
- [ ] Continuous synchronization
- [ ] Social features
- [ ] User profiles
- [ ] Recommendation engine
- [ ] AI chatbot
- [ ] Custom ML model
- [ ] Complex admin dashboard
- [ ] Microservices
- [ ] Kubernetes
- [ ] Enterprise accounts
- [ ] Affiliate program
- [ ] Elaborate referral system
- [ ] Full subscription platform

---

# Decision Rule After MVP

After meaningful real-world usage:

## If users transfer but do not pay

Investigate whether:
- the paid offer is unnecessary,
- free limits are too generous,
- the value proposition is wrong,
- or the transfer quality is not differentiated enough.

Do not immediately add more features.

## If users pay

Improve:
- transfer reliability
- matching quality
- large-library handling
- payment conversion
- acquisition

## If users complain mainly about wrong matches

Prioritize the matching/review system.

Do not respond by adding unrelated product features.

## If Google/platform constraints prevent viable scale

Evaluate alternatives in this order:

1. quota optimization and caching
2. quota increase/application
3. reduce expensive operations
4. local helper architecture
5. additional providers/architectures

Do not build the local helper preemptively.

## If demand is weak

Stop.

The MVP has done its job.

---

# Working Principle

The roadmap follows one rule:

> **Earn the right to add complexity.**

Every new feature should answer one of these questions:

1. Does it help a user complete a migration?
2. Does it materially improve migration accuracy or trust?
3. Does it materially improve acquisition?
4. Does it materially improve sustainable revenue?
5. Does it address a security/reliability requirement?

If the answer is no, it probably does not belong in the MVP.
