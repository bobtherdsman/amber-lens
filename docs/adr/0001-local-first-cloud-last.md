# ADR 0001: Local-First, Cloud-Last

## Status

Accepted

## Context

Enterprise customers are reluctant to run SaaS scanners that export infrastructure and database telemetry before they can audit what is collected.

## Decision

Amber Lens runs locally by default. It generates local value before any cloud interaction. Cloud submission is explicit and optional.

## Consequences

Positive: better customer trust, easier security review, and clear differentiation from SaaS-first scanners.

Tradeoff: more functionality must run locally.
