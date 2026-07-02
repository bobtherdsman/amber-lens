# Testing Strategy

## Purpose

Testing must prove Amber Lens is correct, safe, deterministic, and privacy-preserving.

## Unit Tests

- Token generation determinism
- Salt rotation
- Mapping key persistence
- Schema validation
- Report rendering
- Credential validation
- Canonical Asset Model mapping

## Fixture Tests

Use static fixtures for vCenter exports, SQL Server DMV outputs, Oracle V$ outputs, and optional AWR-like outputs where licensed and enabled.

## Integration Tests

End-to-end fixture scan, local Discovery Report generation, Workload Fingerprint generation, and optional cloud submission to mock endpoint.

## Security Tests

No raw identifiers in fingerprint, no mapping key in outbound payload, no secrets in logs, no unauthorized outbound calls, and GPL/AGPL dependency block.
