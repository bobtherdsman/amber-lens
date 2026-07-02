# Privacy Engine

## Purpose

The Privacy Engine converts local customer-identifying data into deterministic tokens for cloud-bound payloads.

## Responsibilities

- Tokenize hostnames, IP addresses, database names, schema names, cluster names, and account-like identifiers.
- Preserve relationships.
- Keep mapping keys local.
- Generate sovereignty receipt.
- Scrub secrets from logs and payloads.

## Two-Pass Tokenization

Pass 1 uses HMAC-SHA-256 with a customer-controlled local mapping key.

Pass 2 applies per-publish salt rotation so tokens are stable within one publish but not easily correlatable across unrelated publishes.

## Example

```text
SQLPROD01      → host-8f31a2
PayrollDB      → db-41c9aa
10.10.15.42    → ip-29d113
```

## Security Rule

No cloud-bound artifact may contain raw identifiers.
