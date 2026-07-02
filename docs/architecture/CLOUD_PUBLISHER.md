# Cloud Publisher

## Purpose

The Cloud Publisher optionally submits the tokenized Workload Fingerprint to Amber Cloud for Navigator analysis.

## Rules

Cloud submission is customer-initiated only, never automatic, explicitly confirmed, limited to tokenized fingerprints, and excluded from the default local run.

## Command

```bash
amber-lens submit --input fingerprint.json --endpoint https://api.ambercloud.ai
```

## Payload Must Not Include

Local mapping key, raw identifiers, credentials, raw logs, secrets, application data, or table contents.

## Transport Requirements

TLS 1.3, retry-safe, idempotent submission, receipt returned to customer, and structured error handling.
