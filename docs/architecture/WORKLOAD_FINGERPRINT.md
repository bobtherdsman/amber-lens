# Workload Fingerprint

## Purpose

The Workload Fingerprint is the tokenized, normalized, cloud-bound contract between Amber Lens and Amber Navigator.

It is not a raw scan. It is not the Discovery Report. It is the identity-stripped technical profile used for paid analysis.

## Minimum Schema

```json
{
  "fingerprint_id": "uuidv7",
  "schema_version": "1.0.0",
  "token_namespace": "opaque-customer-derived-namespace",
  "source_class": "vcenter|sqlserver|oracle|mixed",
  "workload_tokens": [],
  "metric_series": [],
  "feature_flags": [],
  "licensing_footprint": {},
  "sample_window": {
    "start": "timestamp",
    "end": "timestamp",
    "cadence_seconds": 300
  },
  "sovereignty_receipt": "sha256"
}
```

## Must Not Include

Customer names, hostnames, raw IPs, database names, schema names, usernames, secrets, credentials, application table contents, or raw logs.

## Relationship to Navigator

Navigator consumes the Workload Fingerprint for risk, TCO, what-if analysis, and future bid workflows. Navigator should never require raw collector output.
