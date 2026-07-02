# ADR 0002: Use a Canonical Asset Model

## Status

Accepted

## Context

VMware, SQL Server, and Oracle expose different APIs and data structures.

## Decision

All collectors normalize output into a Canonical Asset Model.

## Consequences

Positive: Discovery Report, Privacy Engine, and Workload Fingerprint generation all consume one model.

Tradeoff: the model must be carefully designed and versioned.
