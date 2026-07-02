# ADR 0004: Workload Fingerprint as Cloud-Bound Contract

## Status

Accepted

## Context

Amber Navigator needs a stable, privacy-preserving input contract.

## Decision

Amber Lens produces a tokenized Workload Fingerprint. Navigator consumes the fingerprint, not raw scans.

## Consequences

Positive: stable Lens-to-Navigator boundary, better privacy, easier future expansion.

Tradeoff: fingerprint schema must be versioned and validated.
