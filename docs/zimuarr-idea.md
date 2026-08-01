# Zimuarr — Project Handoff

## Overview

**Zimuarr** is a self-hosted subtitle translation control plane designed to work seamlessly with Bazarr.

The name comes from the Mandarin Chinese word **字幕 — zìmù**, meaning "subtitles" or "captions," combined with the familiar `arr` naming convention.

Zimuarr is an independent product with its own domain model, database, API, translation engine, safety controls, and user interface. It is the successor to, and a total replacement of, two earlier systems: Lingarr (a translation service) and Perevoditarr (a declarative orchestration bridge between Bazarr and Lingarr). Zimuarr must not depend on, reference, or inherit runtime concepts from either. It collapses the previous three-party topology (Bazarr ↔ orchestrator ↔ translator) into a two-party one: Bazarr and Zimuarr.

Bazarr remains responsible for subtitle and media integration. Zimuarr is responsible for deciding which translations should exist, safely producing them, and keeping them tracked and upgradeable over time.

Zimuarr is licensed under **AGPL-3.0**. Because the AGPL's network clause applies to a self-hosted web application, the UI must link to the corresponding source (a footer or About item is sufficient) from the first release onward.

### Why Zimuarr exists — the sharpened differentiator

The predecessor stack has verified, shipping failure modes that Zimuarr's core invariants are designed to eliminate. These are not hypothetical; they were confirmed by source inspection:

* **Silent partial success.** When a translated position is missing from a provider batch result, Lingarr logs a warning and substitutes the *original source line*, then proceeds as if the translation succeeded. Bazarr, on its side, only replaces lines present and non-empty in the response and keeps the source text for everything else — and then records the operation as a success in history. The combined result is a subtitle file that is part source-language, part target-language, recorded everywhere as complete.
* **Duplicate requests collapse to empty output.** Lingarr deduplicates active content-translation requests by returning an *empty array*. Through Bazarr's handling, an empty array means "no positions translated" → all source lines kept → success logged. Bazarr's own automatic retry on timeout can trigger this: attempt 1 times out while still in progress, attempt 2 is deduplicated to an empty array, and Bazarr writes an untranslated file as a successful translation.

Zimuarr's answer is enforced completeness invariants, idempotency that returns cached completed output (never an empty dedup response), full provenance, budgets, and admission control. The headline is not "more AI features" — it is: **a translation is either provably complete or it did not happen.**

## Engineering Baseline

Repository quality controls should be established at the beginning of the project, not added after the codebase has grown.

Zimuarr should use **prek** for local Git hooks and repeat the same critical checks in CI. The repository should reject malformed files, accidental secrets, invalid commit messages, unsafe type-checker configuration, and code that does not pass the backend or frontend quality gates.

The project should use Conventional Commits.

### `prek.toml`

```toml
# prek.toml — pre-commit hooks configuration
# Docs: https://prek.j178.dev/configuration/

# Builtin hooks — fast, offline, Rust-native
[[repos]]
repo = "builtin"
hooks = [
  { id = "trailing-whitespace" },
  { id = "end-of-file-fixer" },
  { id = "mixed-line-ending", args = ["--fix=lf"] },
  { id = "check-merge-conflict" },
  { id = "check-case-conflict" },
  { id = "check-added-large-files", args = ["--maxkb=512"] },
  { id = "detect-private-key" },
  { id = "check-json" },
  { id = "check-toml" },
  { id = "check-yaml" },
  { id = "no-commit-to-branch", args = ["--branch", "main"] },
]

# Conventional Commits — enforce Conventional Commit message format
[[repos]]
repo = "https://github.com/compilerla/conventional-pre-commit"
rev = "v4.4.0"
hooks = [
  { id = "conventional-pre-commit", stages = ["commit-msg"] },
]

# Secret and credential leak protection
#
# Bazarr credentials, AI-provider credentials, fixtures, logs, generated
# diagnostic bundles, and local environment files must never contain
# committed secrets.
[[repos]]
repo = "https://github.com/gitleaks/gitleaks"
rev = "v8.30.1"
hooks = [
  { id = "gitleaks" },
]

# Project-specific monorepo quality gates
[[repos]]
repo = "local"
hooks = [
  # Backend: uv, Ruff, and basedpyright
  #
  # NOTE: use `uv run --project backend`, NOT `--directory backend`.
  # prek passes repo-relative filenames (backend/src/...); `--directory`
  # changes the working directory to backend/, which breaks those paths.
  # `--project` keeps the cwd at the repo root while using backend's
  # environment.
  {
    id = "ruff-format",
    name = "ruff format",
    entry = "uv run --project backend ruff format",
    language = "system",
    types = ["python"],
    files = "^backend/",
  },
  {
    id = "ruff-check",
    name = "ruff check",
    entry = "uv run --project backend ruff check --fix",
    language = "system",
    types = ["python"],
    files = "^backend/",
  },
  {
    id = "basedpyright",
    name = "basedpyright",
    entry = "uv run --project backend basedpyright",
    language = "system",
    types = ["python"],
    files = "^backend/",
    pass_filenames = false,
  },

  # Typing-policy gate
  #
  # Global basedpyright diagnostic overrides, warning suppression, baselines,
  # and weakened project-wide settings are forbidden.
  {
    id = "basedpyright-config-gate",
    name = "basedpyright config gate",
    entry = "uv run --project backend python backend/tools/check_basedpyright_config.py",
    language = "system",
    files = "^backend/(pyproject\\.toml|tools/|pyrightconfig\\.json|\\.basedpyright/|src/|tests/)",
    pass_filenames = false,
  },

  # Frontend: Bun, Biome, and svelte-check
  #
  # Biome is the deliberate formatter and linter choice for the frontend.
  # svelte-check covers TypeScript and Svelte component type checking.
  {
    id = "biome-check",
    name = "biome check",
    entry = "bun run --cwd frontend lint",
    language = "system",
    files = "^frontend/",
    pass_filenames = false,
  },
  {
    id = "svelte-check",
    name = "svelte check",
    entry = "bun run --cwd frontend check",
    language = "system",
    files = "^frontend/",
    pass_filenames = false,
  },
]
```

CI should execute equivalent checks independently. Local hooks improve feedback speed, but they must not be the only enforcement boundary.

The complete CI quality gate should eventually include:

* Prek validation
* Secret scanning
* Ruff formatting verification
* Ruff linting
* basedpyright
* The basedpyright configuration gate
* Backend tests
* Database migration checks
* Biome
* svelte-check
* Frontend tests
* Production frontend build
* **Bazarr contract tests** — a test suite exercised against a pinned Bazarr version (see "Bazarr Wire Contract"), so contract drift is detected in CI rather than in users' libraries
* Container and deployment validation

## Strict Typing Policy

The backend should use basedpyright in `recommended` mode, with warnings treated as failures through project policy and CI enforcement.

Project-wide diagnostic suppression is forbidden.

The backend `pyproject.toml` should contain:

```toml
[tool.basedpyright]
# HARD RULE — NO GLOBAL DIAGNOSTIC OVERRIDES IN THIS TABLE. EVER.
# Never set failOnWarnings, never disable a report* rule here, never add a
# baseline file, and never widen this table beyond the allowlisted keys below.
# CI and the prek hooks enforce this via tools/check_basedpyright_config.py.
# Warnings are failures: fix the code. If a third-party API is genuinely
# untyped at a single call site, suppress that one line with
# `# pyright: ignore[ruleName]` plus a short justification — nothing broader.
pythonVersion = "3.14"
typeCheckingMode = "recommended"
include = ["src", "tests", "tools"]
```

The configuration gate should verify that:

* Only approved keys exist in `[tool.basedpyright]`.
* No diagnostic category is disabled globally.
* No baseline file is configured.
* No broad ignore path is introduced.
* Warnings are not weakened or hidden.
* Any future configuration expansion is an explicit reviewed decision.

A third-party typing problem may be suppressed only at the narrowest affected line, with the specific basedpyright rule and a short justification.

## Modular Codebase Structure

Zimuarr should be organized as a modular monorepo.

The codebase must prefer cohesive packages and directories over large flat modules. Domain areas should have clear ownership and dependency boundaries.

The project should avoid:

* God files
* God services
* God controllers
* Large generic utility modules
* Business rules embedded in route handlers
* Database models doubling as every other application representation
* Circular dependencies between feature areas
* Frontend components that fetch data, implement business policy, manage global state, and render complex UI in one file
* Catch-all directories that accumulate unrelated code

Files should generally have one clear responsibility. When a module begins coordinating unrelated concerns, it should be divided into smaller cohesive modules.

Suggested repository shape:

```text
zimuarr/
├── backend/
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── migrations/
│   ├── tools/
│   ├── tests/
│   └── src/
│       └── zimuarr/
│           ├── app/
│           ├── config/
│           ├── domain/
│           ├── persistence/
│           ├── integrations/
│           ├── providers/
│           ├── reconciliation/
│           ├── translation/
│           ├── safety/
│           ├── auth/
│           ├── events/
│           ├── workers/
│           ├── diagnostics/
│           └── api/
├── frontend/
│   ├── package.json
│   ├── bun.lock
│   ├── biome.json
│   ├── uno.config.ts
│   ├── svelte.config.js
│   ├── vite.config.ts
│   ├── tests/
│   └── src/
│       ├── lib/
│       │   ├── api/
│       │   ├── components/
│       │   ├── features/
│       │   ├── state/
│       │   ├── types/
│       │   └── utilities/
│       └── routes/
├── deploy/
├── docs/
├── tools/
├── prek.toml
└── README.md
```

This is an organizational direction rather than a requirement that every directory exist immediately.

Packages should be introduced when they represent a real boundary, not merely to create artificial nesting.

### Perevoditarr as a structural quarry

Perevoditarr (same author, same AGPL-3.0 license) already implements much of the reconciliation architecture: its module decomposition (`mirror`, `intents`, `policy`, `rails`, `dispatch`, `doctor`, `watch`), its authentication with first-administrator setup, its SSE infrastructure, its instance-registration version gating, and its basedpyright configuration-gate tool. These map almost one-to-one onto Zimuarr's needs and should be ported structurally rather than rewritten from principle.

What must **not** survive the port:

* The Lingarr HTTP client and any Lingarr-shaped abstraction.
* Any dispatch, health-check, circuit-breaker, or version-gating logic that assumes a *third* external system exists. Perevoditarr reasoned about two external systems with their own queues and opaque states; in Zimuarr, translation execution is an internal, fully observable pipeline with no blind spot where "the translator has it now."
* Historical terminology anywhere in the domain (see "Product Independence").

Ported code should be audited for three-party-topology assumptions, not merely renamed.

### Backend boundaries

The backend should broadly separate:

* Domain rules
* Application orchestration
* Persistence
* External integrations
* API transport
* Background execution
* Configuration and application assembly

Route handlers should validate transport-level input, invoke application services, and return typed responses. They should not contain reconciliation logic, provider orchestration, budgeting calculations, or SQL queries.

Domain logic should remain independent of Litestar and HTTP wherever practical.

Persistence implementations should not define domain policy.

External-provider payloads should be converted into Zimuarr-native types at the integration boundary.

### Frontend boundaries

The frontend should be organized primarily by product feature, with shared UI primitives kept separate.

A feature may contain its own:

* Components
* API queries
* View models
* Runes-based state
* Forms
* Tables
* Filters
* Feature-specific utilities

Large page components should be decomposed into meaningful feature components rather than growing into complete application controllers.

The frontend must not duplicate backend reconciliation, safety, or lifecycle rules. It may present decisions and collect configuration, but the backend remains authoritative.

## Core Purpose

Zimuarr should make subtitle translation declarative.

Instead of treating translation as a one-time button press or temporary job, users define translation policies such as:

> Every episode matching this policy should have a Swedish subtitle translated from the preferred English source.

Zimuarr continuously compares that desired state with the subtitles currently reported by Bazarr.

It can then determine whether a translation is:

* Missing
* Current
* Stale
* Manually modified
* Unmanaged
* Blocked by a safety policy
* Quarantined
* No longer required

This reconciliation model allows translated subtitles to remain tracked after creation and to be safely upgraded when their source subtitle, model, prompt, glossary, validation rules, or policy changes.

## Product Independence

Zimuarr must be its own application.

It must not require another subtitle translation application to be installed or running.

It should not contain historical application names in:

* Domain terminology
* Database models
* User-facing screens
* Configuration concepts
* Provider abstractions
* Internal events
* Service names
* Documentation for normal operation

Bazarr currently exposes an external translation contract under terminology chosen by Bazarr. Zimuarr implements the required wire-compatible callback, but that compatibility belongs only at the outer integration boundary.

Internally, it should be represented using Zimuarr-native concepts such as translation sessions, subtitle translation requests, translation attempts, and managed artifacts.

## Bazarr Wire Contract (verified against Bazarr source)

The Bazarr integration is the project's highest-risk dependency: it is an **undocumented, unversioned internal contract** of Bazarr that can change in any release with zero notice. The facts below were verified by source inspection and are normative for the integration boundary. They must be encoded as contract tests run in CI against a pinned Bazarr image, with a supported Bazarr version range enforced at instance registration and surfaced in diagnostics.

### Dispatch (Zimuarr → Bazarr)

* Translation is triggered via `PATCH /api/subtitles` with `action=translate`, requiring: `path` (the subtitle file path *as Bazarr sees it* — path mappings matter), `language` (target, alpha2), `type` (`episode`/`movie`), `id` (Sonarr **episode** ID or Radarr ID), and optional `forced`/`hi`.
* The call enqueues a job in Bazarr's internal queue and returns immediately. The dispatch response confirms nothing about the translation outcome; correctness is established only by later observation.
* Bazarr's translator is a **single global setting** (`translator_type`, plus the external translator's URL and token). Onboarding must verify that Bazarr is configured to point at Zimuarr, and diagnostics should include a self-test translation round trip. Zimuarr cannot coexist with Bazarr's built-in Google/Gemini translators on the same instance; this must be documented.

### Callback (Bazarr → Zimuarr)

* Bazarr POSTs to `{url}/api/translate/content` with header `X-Api-Key` (one static token per Bazarr instance) and payload: `arrMediaId`, `title`, `sourceLanguage`, `targetLanguage`, `mediaType` (`Episode`/`Movie`), and `lines: [{position, line}]`.
* **The request is synchronous and blocking**, with a 1800-second (30-minute) client timeout. There is no async job handoff: the complete translated payload must be returned in the same HTTP response. Consequences:
  * The translation session is not a token exchange; it is a long-lived HTTP request the callback handler holds open while workers execute.
  * Granian and any reverse proxy must be configured for long-lived requests.
  * Chunking, provider calls, and provider retries must fit a per-session deadline of roughly 28 minutes (the Bazarr timeout minus safety margin). The safety engine must model this deadline explicitly.
* **Correlation is weak.** The payload carries no session token, no file path, and — for episodes — `arrMediaId` is the Sonarr **series** ID, not the episode ID. Two episodes of the same series translating to the same language concurrently are metadata-indistinguishable. The only robust correlator is a **content fingerprint of the source lines**: at dispatch time, Zimuarr fetches the source subtitle via Bazarr's `GET /api/subtitles/contents?subtitlePath=...`, computes the expected fingerprint, and matches incoming callbacks against open sessions by fingerprint (plus instance identity and language pair).
  * Subtlety: the callback carries pysubs2 `plaintext` (markup stripped), while the contents endpoint returns raw SRT cue text. Zimuarr's normalization must replicate pysubs2's stripping exactly or fingerprints will not match. This must be proven in the Phase 0 spike before the session model is finalized.
  * Each connected Bazarr instance must be issued a **distinct** API key so the calling instance is identified by credential, not payload.
* **Bazarr silently accepts partial translations.** It replaces only positions present and non-empty in the response, keeps original source text for the rest, and logs the result as a success. Zimuarr's completeness validation is therefore the *only* defense against mixed-language output — not defense-in-depth.
* **Response status semantics** (Bazarr's client behavior): `429` and `>=500` are retried (3 attempts, exponential backoff with jitter); `401` fails as an auth error; any other non-200 fails the Bazarr job immediately without retry. Zimuarr should therefore return:
  * `200` with the complete translated line set — only when every invariant passes.
  * `422` (or another non-retryable 4xx) for validation failures and quarantined output — fail fast, no retry storm.
  * `429` / `5xx` only when a retry might genuinely succeed.
* **Timeouts are a cost trap and an opportunity.** If Bazarr gives up at 30 minutes, it retries while Zimuarr may still be translating and paying. The policy is: keep computing, cache the completed result keyed by session/fingerprint, and serve the cached output instantly on Bazarr's automatic retry. Idempotency must return **cached completed output**, never an empty deduplication response (the predecessor's empty-array dedup, combined with Bazarr's partial-acceptance, produced untranslated files logged as successes — this exact sequence must exist as an adversarial regression test).

### Data-shape facts

* Language codes are alpha2 with a small conversion map applied by Bazarr (`zh→zh-CN`, `zt→zh-TW`, `pb→pt-BR`); Bazarr internally mixes alpha2 and alpha3. Zimuarr needs an explicit language-identity normalization layer at the integration boundary.
* The destination subtitle is always written as `.srt`.
* Because Bazarr sends stripped plaintext and writes translated text back as the full cue text, source markup (italics, positioning) is destroyed by Bazarr regardless of what Zimuarr does. Markup validation on the callback path therefore governs what the *model introduces*, not preservation of source markup; the upstream limitation must be documented.

### Observation (Zimuarr ← Bazarr)

* Bazarr exposes no outbound event stream and no delta/cursor API; its webhooks are inbound. The observed-state mirror is **polling-based, full stop**.
* Mirror synchronization must be designed for large libraries: incremental (per-series), paginated, and rate-limited so Zimuarr does not become a load problem for the Bazarr instance it observes.

## Translation Domain

A translation should not be represented as a single terminal job.

### Desired Subtitle

A persistent declaration that a translated subtitle should exist for a particular media item, language, and subtitle variant.

A desired subtitle remains meaningful after successful translation. It can later become stale, blocked, drifted, or satisfied by a newer artifact.

### Observed Subtitle

A subtitle currently reported by Bazarr.

An observation may contain:

* Bazarr instance
* Media identity
* Language
* Forced status
* Hearing-impaired status
* File path or Bazarr identifier
* Content fingerprint
* Observation timestamp
* Available file metadata

Observed-state data is replaceable and may be rebuilt from Bazarr.

### Managed Artifact

A translated subtitle produced or deliberately adopted by Zimuarr.

It should be linked to:

* Its desired subtitle
* Its source subtitle revision
* Its successful translation attempt
* Its content fingerprint
* Its creation and verification timestamps
* Its current drift status

### Translation Attempt

One execution using a specific combination of:

* Source content revision
* Source and target languages
* Provider
* Model
* Prompt revision
* Glossary revision
* Validation-policy revision
* Chunking strategy
* Provider parameters

An attempt may succeed, fail, be retried, or produce a quarantined candidate.

Prompt revisions, glossary revisions, and validation-policy revisions should be **content-addressed** (identified by a hash of their content). This makes provenance comparisons and staleness detection reduce to hash inequality, and makes revision tracking nearly free.

### Translation Session

A short-lived authorization linking a Zimuarr reconciliation decision with the callback subsequently received from Bazarr.

A session should contain enough information to:

* Authenticate and correlate the callback — by per-instance credential plus **source-content fingerprint** (see "Bazarr Wire Contract"; metadata alone cannot disambiguate concurrent episodes of the same series)
* Confirm the expected source
* Reserve a budget
* Prevent duplicate execution
* Enforce a **deadline derived from Bazarr's callback timeout** (chunking and provider retries must fit within it)
* Return cached completed output for retries — including Bazarr's automatic retry after its own timeout
* Record the complete execution lifecycle, including the abandoned-then-completed case (Bazarr timed out; Zimuarr finished, cached, and served the result on retry)

### Reconciliation Plan

An explained set of actions produced by comparing desired, observed, and managed state.

### Safety Decision

The result of evaluating dispatch windows, budgets, limits, breaker state, concurrency, session deadlines, and other admission rules.

### Quarantine Case

A translation candidate that was retained for inspection but rejected from normal activation.

## Provider Capability Model

Zimuarr should support many AI providers without turning each into a bespoke fork of the translation engine.

Providers must be described by a **capability model** — explicit flags and parameters such as:

* Structured-output support: strict schema, JSON mode, or free text only
* Maximum request size / context budget
* Token accounting granularity (exact usage reporting vs estimation)
* Streaming support
* Rate-limit characteristics
* Pricing model reference

The validation strategy, chunking strategy, and output-parsing path are **selected by capability flags**, not implemented per provider. Adding a provider should be a matter of writing a thin adapter plus a capability declaration — a configuration-level change, not an engine change.

The internal provider contract should be richer than the minimal response Bazarr requires, and structured provider responses should be used wherever the capability model says they are supported.

### Explicit non-goal: raw request templating

User-customizable raw provider request bodies (as offered by predecessor systems) are a **non-goal**. Raw templating is fundamentally at odds with the validation invariants: a user-templated request can break the schema contract the completeness guarantees depend on. Zimuarr offers typed prompt configuration, typed provider parameters, and glossaries — never raw request-body templates. This is a deliberate, documented product decision, recorded here so it does not creep in later.

## Structured Translation and Validation

Before returning a translation to Bazarr, Zimuarr must enforce hard invariants such as:

* Every source cue is represented exactly once.
* No positions are missing.
* No positions are duplicated.
* No unknown positions are introduced.
* Output ordering remains valid.
* Non-empty source lines do not silently become empty.
* Markup and control sequences satisfy the configured policy (governing model-introduced markup; see the wire-contract note on source markup).
* Translation expansion remains within reasonable limits.
* Repetition checks pass.
* Untranslated-text checks pass.
* Provider output satisfies the expected schema.
* Every chunk succeeds before the complete subtitle is accepted.

A partial translation must never be treated as successful. Because Bazarr silently accepts partial responses and records them as successes, this invariant is the only line of defense — a failed validation must surface to Bazarr as a non-retryable error status, never as a reduced line set.

Provider retries and repeated Bazarr callbacks must be idempotent. A completed translation must be reused rather than translated and billed again, and idempotent responses must always carry the full cached output.

## Safety-First Behaviour

Zimuarr must be safe by default.

New Bazarr connections and translation policies should begin in observation-only or dry-run mode. Active translation requires deliberate activation.

The safety system should support:

* Dispatch windows
* Global concurrency limits
* Per-Bazarr-instance concurrency limits
* Per-provider concurrency limits
* Hourly and daily volume caps
* Character limits
* Subtitle-line limits
* Per-attempt budget ceilings
* Daily budget ceilings
* Monthly budget ceilings
* Cost reservation before dispatch
* Cost settlement after provider completion
* Per-session deadlines derived from the Bazarr callback timeout
* Provider circuit breakers
* Bazarr circuit breakers
* Structured-output failure breakers
* Bounded retries with backoff
* Global pause controls
* Provider-level pause controls
* Policy-level pause controls
* Quarantine for invalid or suspicious output
* Manual-edit protection
* Dry-run plan previews

Cost accounting requires **pricing provenance**: provider token prices change over time, so pricing tables are versioned configuration, and every settled cost records which pricing revision it was computed against. Historical cost records must remain explainable after price changes.

Existing unmanaged subtitles should not be overwritten automatically.

Manually modified subtitles should not be replaced without an explicit policy or approval.

Unknown or unauthorized translation callbacks should be rejected by default.

## Security and Authentication

Zimuarr's own security surface is a first-class Phase 1 concern, not an afterthought:

* The web UI and API require authentication, with a first-run administrator setup flow (the predecessor's implementation is a proven starting point to port).
* The Bazarr callback endpoint authenticates via per-instance API keys and rejects unknown callers by default. It must be safely exposable when Bazarr and Zimuarr are on different networks.
* Machine access to Zimuarr's own API uses API keys.
* Reverse-proxy deployment guidance is documented.
* Secrets (Bazarr tokens, provider keys) are encrypted at rest where practical and never logged; diagnostic bundles are redacted.

## Reconciliation Principles

Zimuarr should behave as a declarative reconciliation engine rather than a traditional queue processor.

Each reconciliation cycle should:

1. Observe current Bazarr state.
2. Update the local external-state mirror.
3. Resolve effective translation policies.
4. Calculate desired subtitles.
5. Compare desired, observed, and managed state.
6. Classify missing subtitles and drift.
7. Produce an explainable plan.
8. Evaluate safety admission.
9. Dispatch authorized work.
10. Verify the resulting subtitle.

The planner used for dry-run previews and the planner used for real dispatch should share the same decision logic.

Live events may refresh the interface or cause an earlier observation, but they should not be treated as authoritative proof of success. Durable observations and Zimuarr's own records determine correctness.

## Application Architecture

Zimuarr should be developed as a monorepo containing:

* A Python backend
* A SvelteKit single-page application
* Shared development tooling
* Deployment assets
* Architecture and operational documentation

## Backend Stack

* Python 3.14
* Litestar
* Granian
* msgspec
* SQLAlchemy 2 async
* asyncpg
* PostgreSQL
* Advanced Alchemy where it provides clear value
* Alembic
* httpx
* structlog
* uv
* Ruff
* basedpyright
* pytest
* Hypothesis where property testing is beneficial
* prek

The backend should be async-first and fully typed.

PostgreSQL is a deliberate choice: durable worker claims, dispatch-session leases, and reconciliation locks need it. It diverges from the SQLite-by-default norm of the `arr` ecosystem, so installation friction is expected and packaging must make single-command deployment trivial (deployment specifics are handled outside this document).

The backend must not use:

* Pydantic for native request and response models
* FastAPI dependency patterns
* Uvicorn or Gunicorn
* Synchronous HTTP clients
* Blocking database access in async handlers
* SQLAlchemy's legacy query API
* Broad type suppressions
* Production schema creation through `create_all`

Major backend areas should include:

* Bazarr integration
* Reconciliation domain
* Translation engine
* Provider adapters and the capability model
* Safety and budget engine
* Authentication and security
* Persistence
* HTTP API
* Bazarr translation callback
* Background worker runtime
* Server-sent event stream
* Audit and domain events
* Diagnostics ("doctor")

The API and worker should be distinct runtime roles from the same backend package. A combined mode may be offered for straightforward single-instance installations.

Correctness must not depend on only one Granian process being present. Worker claims, dispatch sessions, and reconciliation leases should be durable where duplicate execution would be unsafe. Granian (and any fronting proxy) must be configured for long-lived requests to accommodate the synchronous Bazarr callback.

## Frontend Stack

* Bun
* Svelte 5
* SvelteKit 2
* TypeScript
* UnoCSS with `presetWind4`
* shadcn-svelte
* Bits UI
* Biome
* svelte-check
* Bun's test runner for pure TypeScript modules; **Vitest + Testing Library for Svelte component tests** (decided up front so the test story does not fragment)
* prek

shadcn-svelte and UnoCSS `presetWind4` do not combine out of the box. The integration follows the `unocss-preset-shadcn` path per the frontend guidelines: do **not** run `shadcn-svelte init`; create `components.json` and the `cn()` utility manually; keep an empty `tailwind.config.js` solely to satisfy the shadcn CLI; configure `uno.config.ts` with `presetWind4`, `unocss-preset-animations`, and `presetShadcn`, with the content pipeline widened to scan `.ts`/`.js`. This is spelled out here so an implementer does not fight the CLI.

During development, the SvelteKit development server should proxy `/api` to the Litestar backend. This includes the `/api/v1/events` server-sent event stream.

The frontend should be implemented using modern Svelte 5 patterns:

* Runes
* `$app/state`
* Callback props
* Snippets
* Attachments where appropriate
* Typed generated route data
* Strict TypeScript

It should not introduce legacy Svelte 4 patterns into new code.

Biome is the selected frontend formatter and linter. ESLint and Prettier should not be added unless a future documented requirement cannot be met through Biome and the exception is approved explicitly.

The frontend should consume Zimuarr's API as its source of truth and should not reproduce backend reconciliation or state-machine logic.

## API Contracts

Zimuarr should expose its own versioned API under `/api/v1`.

Public API models should be explicit and typed.

The frontend should use a generated or otherwise mechanically synchronized TypeScript API client based on the backend's OpenAPI schema.

The server-sent event stream should use versioned event envelopes and durable event identifiers where recovery after reconnection is required.

Provider-specific and Bazarr-specific payloads should not leak directly into the public frontend API.

## Initial Product Areas

The first interface should include:

* First-run administrator setup and authentication
* Bazarr onboarding, including verification that Bazarr's global translator is pointed at Zimuarr
* Bazarr connection diagnostics, including a self-test translation round trip and supported-version checks
* Dashboard
* System health
* Media and subtitle inventory
* Translation policies
* Source-language preferences
* Dry-run reconciliation plans
* Translation activity and history
* Provider and model configuration
* Budget and usage reporting
* Circuit-breaker state
* Quarantine review
* Managed-artifact provenance
* Global pause controls
* Scoped pause controls
* An About view linking to the source (AGPL-3.0 network clause)

## Suggested Delivery Sequence

The sequencing principle: **prove the riskiest bet first.** The entire product rests on an undocumented Bazarr contract; that is validated before reconciliation infrastructure is built, not after.

### Phase 0 — Contract Spike (de-risk before building)

* Stand up the `/api/translate/content` callback against a real Bazarr instance
* Prove fingerprint-based session correlation end to end, including the pysubs2 plaintext-normalization subtlety
* Verify concurrent-episode disambiguation (same series, same target language) under the two-party topology — the predecessor's correlation ran under different dispatch-timing assumptions and does not fully de-risk this
* Exercise the timeout/retry/cached-response path, including the adversarial "Bazarr retry after timeout" sequence
* Capture the verified behavior as the initial CI contract-test suite against a pinned Bazarr image

### Phase 1 — Repository and Observation

* Monorepo foundation
* Prek configuration
* Conventional Commit enforcement
* Secret scanning
* Backend and frontend quality gates
* Strict basedpyright configuration gate
* Modular package boundaries (porting perevoditarr module structure where applicable, audited for three-party assumptions)
* Authentication and first-run setup
* Bazarr connection with version gating
* Read-only media and subtitle mirror (incremental, paginated, rate-limited polling)
* Domain events
* SSE
* Basic dashboard
* No translation dispatch

### Phase 2 — Declarative Planning

* Translation policies
* Desired subtitle calculation
* Source selection
* Drift classification
* Explainable dry-run plans
* Policy provenance
* Safety-decision previews

### Phase 3 — Closed-Loop Translation

* Translation sessions with fingerprint correlation and deadline enforcement
* Bazarr dispatch
* Bazarr-compatible translation callback (building on the Phase 0 spike)
* One AI provider, implemented through the capability model from day one
* Structured-output validation and completeness invariants
* Idempotency with cached-output replay
* Result observation
* Managed-artifact provenance

### Phase 4 — Safety and Operations

* Dispatch windows
* Volume caps
* Budget reservations and settlement with pricing provenance
* Circuit breakers
* Concurrency limits and session deadlines
* Quarantine
* Emergency pause controls
* Durable worker claims
* Operational diagnostics

### Phase 5 — Upgrades and Lifecycle

* Source-change detection
* Prompt revisions (content-addressed)
* Glossary revisions (content-addressed)
* Provider and model revisions
* Validation-policy revisions (content-addressed)
* Managed upgrades
* Manual-edit detection
* Adoption of existing subtitles
* Rollback and replacement workflows

## Product and Engineering Principles

1. **Bazarr integration without Bazarr modification.**
2. **Zimuarr has no runtime dependency on historical translation applications.**
3. **Desired state is durable; jobs and attempts are temporary.**
4. **Every managed subtitle has provenance.**
5. **No partial translation is successful — and because Bazarr silently accepts partial output, Zimuarr's validation is the only defense.**
6. **Safety decisions happen before dispatch.**
7. **Retries must not create duplicate work or duplicate cost; idempotent responses always return the full cached output, never an empty result.**
8. **Live telemetry never replaces durable verification.**
9. **Unmanaged and manually edited subtitles are protected by default.**
10. **Plans and decisions must be explainable.**
11. **The Bazarr contract is verified by tests, version-gated at registration, and monitored for drift.**
12. **Providers are described by capabilities; the engine adapts by capability flags, not per-provider forks.**
13. **Raw provider request templating is a permanent non-goal.**
14. **Strict typing is enforced rather than weakened.**
15. **Global diagnostic suppression is forbidden.**
16. **Secrets must never be committed or logged.**
17. **The codebase should remain modular and navigable.**
18. **No single file, service, controller, or component should become the centre of the entire application.**
19. **Business rules belong in explicit domain and application modules, not transport handlers.**
20. **The frontend presents backend decisions; it does not independently recreate them.**
21. **Local quality hooks and CI enforce the same engineering expectations.**

## Summary

Zimuarr is a declarative, safety-first subtitle translation platform for Bazarr, and the autonomous successor to the Lingarr + Perevoditarr stack.

It uses Bazarr as the subtitle execution and file-integration layer while independently managing translation intent, provider execution, validation, provenance, budgets, safety rails, reconciliation, and future upgrades. Its integration boundary implements Bazarr's existing external-translation wire contract exactly, treats that contract as a tested and version-gated dependency, and defends against the verified failure modes of its predecessors: silent partial translations and retry-induced empty results.

The project should be built from the start with strict typing, automated repository safeguards, secret protection, Conventional Commits, deterministic quality gates, a modular monorepo structure, and its own authentication — with the riskiest integration behavior proven in a Phase 0 spike before the reconciliation engine is built.

The result should feel native to the `arr` ecosystem while solving a different class of problem: not merely translating subtitles, but continuously managing translated subtitles as durable, auditable, safe, and upgradeable artifacts.

