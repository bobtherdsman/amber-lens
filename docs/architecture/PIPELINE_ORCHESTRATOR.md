# Pipeline Orchestrator

## Purpose

The Pipeline Orchestrator coordinates a complete Amber Lens run. It is the control plane for local execution.

## Responsibilities

- Load configuration.
- Validate source definitions.
- Validate read-only credentials.
- Start collectors.
- Coordinate the sample window.
- Aggregate collector output.
- Track progress and warnings.
- Write local artifacts.
- Trigger local Discovery Report generation.
- Trigger privacy/tokenization only when a fingerprint is needed.
- Ensure cloud submission never happens automatically.

## Non-Responsibilities

The orchestrator does not collect source-specific data directly, tokenize identifiers itself, render HTML itself, submit cloud payloads directly, or perform Navigator analysis.

## Failure Handling

Amber Lens should allow partial success. If SQL Server collection fails but VMware succeeds, the report should still be generated and marked degraded.
