# CLAUDE.md — Amber Lens

Project brief for Claude Code. Read this first, every session.

## What this is
**Amber Lens** is the open-source, **Foundation (free)** tier of the Amber Cloud
platform: a local-first, read-only, agentless **migration discovery scanner**.
It scans source workloads, emits a versioned **Workload Fingerprint**, and renders
a free **Discovery Report**. Positioning: the neutral "Switzerland of cloud
migration" — customer-controlled, auditable, vendor-agnostic.

This repo is **Apache-2.0 and public**. Proprietary paid code (Amber Navigator)
lives in a **separate private repo** — never add it here (clean-room separation).

## Non-negotiable principles (apply to all code)
1. **Read-only & agentless.** Collectors only *read* metadata (vCenter API, SQL
   Server DMVs, Oracle v$/AWR, WMI/SSH). Never write to, install on, or mutate a
   target. No agents.
2. **Local-first, tokens-only.** Two-pass tokenization: the token↔real-name map
   **never leaves the customer network**. Only salted tokens go in the fingerprint.
   Never emit real hostnames, IPs, or secrets.
3. **Destination-neutral.** Nothing favors a specific cloud. Output supports an
   objective rehost/replatform/refactor/repurchase/retire decision.
4. **Zero telemetry. Ever.** Amber Lens emits no analytics, pings, or usage data.
5. **Free vs paid line (enforced in the data contract):** the Lens produces
   **facts + preliminary, non-monetary recommendations** (the free hook).
   **Validated risk scores, TCO/cost ($), and reverse-bid proposals are PAID
   (Navigator) and must NEVER appear in the Lens or the fingerprint.** Directional
   `impact` (ranges/counts) is fine; dollars are not.
6. **License hygiene.** Apache-2.0 only. No GPL/AGPL dependencies (CI gates this).
   Prefer the Go standard library; add dependencies sparingly and deliberately.

## Architecture
```
collectors → tokenize (two-pass) → fingerprint v1 → report (HTML)
   read-only        keys stay local       tokens only        free Discovery Report
        ▲ 5-min sampling over 72h (~95% confidence)
```
- `cmd/amber-lens` — single-binary entrypoint
- `internal/collectors` — one package per source workload (vmware, sqlserver, oracle, osmeta, deps); all implement the read-only `Collector` interface
- `internal/fingerprint` — Go types + `Validate()` for the Workload Fingerprint
- `internal/tokenize` — two-pass tokenization
- `internal/sampling` — 5-min × 72h scheduler
- `internal/report` — Discovery Report (HTML) generator
- `pkg/schema` — canonical, versioned Fingerprint JSON Schema (+ examples)
- `profiles` — open-source collector profiles (community/vendor)
- `docs` — architecture, fingerprint schema, "what we read"

## The Fingerprint contract (most important)
- Canonical: `pkg/schema/fingerprint.schema.json` (JSON Schema, Draft 2020-12).
- Go types in `internal/fingerprint` and the example in
  `pkg/schema/examples/fingerprint.example.json` **must stay in lockstep** with it.
- **v1 core freezes 2026-07-01.** Additive, backward-compatible fields → bump the
  minor version. New workload *kinds* arrive via `extensions` or a future major —
  do not silently break v1.
- If you change the schema, also update the Go types, the example, the test, and
  `docs/fingerprint-schema.md` in the same change.

## Conventions
- Go 1.22+; run `gofmt`. Keep collectors behind the `Collector` interface.
- Every collector must document exactly what it reads in `docs/what-we-read.md`
  (transparency is a feature). Update it whenever a collector reads something new.
- Oracle: AWR/ASH needs the Diagnostic Pack — detect entitlement and fall back to
  Statspack/v$ so the scan never creates a licensing exposure.
- Add a test for new logic. Keep schema/example/Go types/test consistent.

## Build, test, CI
- `make build` · `make test` · `make lint` (or `go build ./...`, `go test ./...`).
- CI (`.github/workflows/ci.yml`) runs build-test, lint, **CycloneDX SBOM**, and a
  **GPL/AGPL license-gate** on every push/PR. Keep all four green.
- An external security audit is published before GA; P0/P1 findings block release.

## Governance & safety
- Do not commit secrets, tokens, or real scan outputs (see `.gitignore`).
- `.claude/` (Claude Code local config) is gitignored — never commit it.
- Contributors work under the clean-room protocol and sign a CIIAA **before**
  contributing (per the Entity Formation Plan). Title ≠ legal cover.

## Vocabulary
Amber Lens (Foundation, free) · Amber Navigator (paid) · Veiled Bid Lab (reverse
blind bidding) · Workload Fingerprint · Discovery Report (free) · Operational Risk
Score & TCO (paid). Use these consistently.
