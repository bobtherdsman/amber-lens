# Engineering Standards

## Purpose

Amber Lens must be easy to audit, easy to build, and safe to run in enterprise environments.

## Go Standards

Prefer simple Go, keep functions small and testable, avoid hidden network calls, avoid global mutable state, and return structured errors.

## Dependency Standards

Apache-2.0 compatible dependencies only, block GPL/AGPL dependencies, generate SBOM per release, and keep dependency tree small.

## Logging Standards

Logs must avoid secrets, raw credentials, and raw identifiers in cloud-bound logs while preserving enough diagnostic context.

## Security Standards

Read-only credentials only, no table contents, no application data, no default outbound calls, explicit cloud submission only, and local mapping keys.
