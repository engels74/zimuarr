# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository State

This repository is **pre-implementation**. The entire tracked tree is:

```
.augment/rules/{backend,frontend}-dev-pro.md   stack coding references
docs/zimuarr-idea.md                           normative project spec (846 lines)
.gitignore  LICENSE (AGPL-3.0)
```

There is no `backend/`, `frontend/`, `prek.toml`, `README.md`, or CI workflow yet. **There are no runnable build, test, or lint commands.** Do not invent them, and do not report a command as verified until the corresponding manifest exists.

`docs/zimuarr-idea.md` is the authoritative specification. Read it before making any design or implementation decision — it contains verified findings (Bazarr source inspection) that are normative, not aspirational.

## Architecture Overview

Zimuarr is a self-hosted subtitle translation control plane for Bazarr. It replaces the three-party Lingarr + Perevoditarr topology with two parties: Bazarr owns subtitle/media file integration; Zimuarr owns translation intent, execution, validation, provenance, budgets, and safety.

The model is **declarative reconciliation**, not queue processing. Each cycle: poll Bazarr into an observed-state mirror → resolve policies → compute desired subtitles → diff desired/observed/managed → classify drift → produce an explainable plan → evaluate safety admission → dispatch → verify. The dry-run planner and the dispatch planner must share one decision path.

The headline invariant: **a translation is either provably complete or it did not happen.** Both predecessor systems shipped silent partial translations that Bazarr recorded as successes; Zimuarr's validation is the only defense, not defense-in-depth.

## Bazarr Integration Constraints

These come from verified Bazarr source inspection (`docs/zimuarr-idea.md` § "Bazarr Wire Contract") and are the highest-risk part of the system. Violating one produces silently corrupt subtitle files.

| Constraint | Required approach |
| --- | --- |
| Bazarr's `POST /api/translate/content` callback is **synchronous and blocking**, 1800 s client timeout | Hold the request open; enforce a per-session deadline of ~28 min covering chunking and all provider retries. Configure Granian and any proxy for long-lived requests. |
| Callback metadata cannot identify a session — for episodes `arrMediaId` is the Sonarr **series** ID | Correlate by **source-content fingerprint** plus per-instance credential and language pair. Replicate pysubs2 `plaintext` stripping exactly, or fingerprints will not match raw SRT from `GET /api/subtitles/contents`. |
| Bazarr accepts partial responses and logs them as success | Return `200` only when every completeness invariant passes; `422` for validation failure and quarantine (Bazarr will not retry); `429`/`5xx` only when retry could genuinely succeed. |
| Bazarr auto-retries after its own timeout | Idempotency must serve the **full cached completed output**. Never return an empty or deduplicated result — that exact sequence is the predecessor bug and must exist as an adversarial regression test. |
| Bazarr's translator is one global setting per instance | Onboarding must verify Bazarr points at Zimuarr; issue a **distinct** API key per Bazarr instance. |

Bazarr exposes no event stream or delta API — the observed-state mirror is polling-based, and must be incremental per-series, paginated, and rate-limited.

## Toolchain

Locked by the spec. Do not substitute alternatives, and do not add ESLint, Prettier, or a `tailwind.config.js` with content in it.

| Layer | Use | Never use |
| --- | --- | --- |
| Backend | Python 3.14, uv, Litestar, Granian, msgspec, SQLAlchemy 2 async + asyncpg, PostgreSQL, Alembic, Advanced Alchemy, httpx, structlog, Ruff, basedpyright, pytest, Hypothesis | Pydantic for request/response models, FastAPI `Depends()`, uvicorn/gunicorn, sync HTTP clients, legacy `session.query()`, `create_all` in production |
| Frontend | Bun, Svelte 5 runes, SvelteKit 2, TypeScript, UnoCSS `presetWind4`, shadcn-svelte, Bits UI, Biome, svelte-check | Svelte 4 idioms (`export let`, `$:`, stores, `on:click`), `$app/stores`, npm/pnpm, ESLint, Prettier |
| Frontend tests | Bun's test runner for pure TS modules; **Vitest + Testing Library** for Svelte component tests | Mixing the two roles |

PostgreSQL is deliberate (durable worker claims, dispatch-session leases, reconciliation locks) despite the `arr` ecosystem's SQLite norm.

## Bootstrapping Workflow

When scaffolding the monorepo, follow the spec's sequence rather than improvising:

1. Establish repository quality controls **first** (`docs/zimuarr-idea.md` § "Engineering Baseline" contains a copy-ready `prek.toml`), not after code accumulates.
2. Add the backend `[tool.basedpyright]` table exactly as specified — `typeCheckingMode = "recommended"`, `pythonVersion = "3.14"`, allowlisted keys only — plus `backend/tools/check_basedpyright_config.py`, the gate that rejects global suppressions and baselines.
3. Scaffold `backend/` (src layout under `src/zimuarr/`) and `frontend/` per the spec's repository shape, introducing packages only where a real boundary exists.
4. Mirror every local hook as an independent CI check; local hooks must not be the only enforcement boundary.
5. Do Phase 0 (the Bazarr contract spike) before building reconciliation infrastructure — the riskiest bet is proven first.

Once `prek.toml` exists, the gate commands are `uv run --project backend ruff format|check --fix`, `uv run --project backend basedpyright`, `bun run --cwd frontend lint`, and `bun run --cwd frontend check`. Use `uv run --project backend`, **not** `--directory backend`: prek passes repo-relative paths and `--directory` breaks them.

## Critical Gotchas

- **`docs/` and `artifacts/` are meant to be gitignored.** Commit `6edf1d7` ("temporary allow docs") commented both entries out at the bottom of `.gitignore`. Before re-enabling them, note that `docs/zimuarr-idea.md` is already tracked and would need `git rm --cached` handling.
- **The spec's `prek.toml` includes `no-commit-to-branch --branch main`.** Once installed, work on a branch; the current direct-to-`main` history predates the hook.
- **No predecessor terminology may enter the product.** Lingarr and Perevoditarr names must not appear in domain terms, database models, UI, configuration, events, or service names. Bazarr's wire vocabulary is confined to the outer integration boundary; internally use translation sessions, subtitle translation requests, translation attempts, and managed artifacts.
- **Perevoditarr is a structural quarry, not a dependency.** Its module decomposition, auth/first-admin flow, SSE infrastructure, and basedpyright config gate should be ported structurally — but audited for three-party assumptions, and stripped of any Lingarr client, health-check, or version-gating logic that assumes a third external system.
- **Raw provider request templating is a permanent non-goal.** Offer typed prompt configuration, typed provider parameters, and glossaries. New providers are a thin adapter plus a capability declaration; the engine branches on capability flags, never per-provider forks.
- **AGPL-3.0 network clause applies.** The UI must link to corresponding source (footer or About item) from the first release onward.

## Additional Documentation

- `docs/zimuarr-idea.md` — The normative spec. Read before any implementation or design work; § "Bazarr Wire Contract" before touching the integration boundary, § "Suggested Delivery Sequence" before starting a new area of work.
- `.augment/rules/backend-dev-pro.md` — Read before writing Python. Covers Litestar routing/DI, msgspec modeling, SQLAlchemy async pitfalls (`MissingGreenlet`, eager loading), Granian, Alembic, and the uv/Ruff/basedpyright workflow.
- `.augment/rules/frontend-dev-pro.md` — Read before writing Svelte or TypeScript. Covers runes, SvelteKit routing and form actions, Bun tooling, and the non-obvious UnoCSS `presetWind4` + shadcn-svelte integration (do not run `shadcn-svelte init`).
