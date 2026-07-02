# Amber Lens System Overview

## Purpose

Amber Lens is the local workload intelligence pipeline for Project Amber Foundation. It runs in the customer's environment, collects read-only metadata from VMware, SQL Server, and Oracle, creates a local Discovery Report, and optionally produces a tokenized Workload Fingerprint for Amber Navigator.

## MVP Scope

In scope for September MVP:

- VMware vCenter
- SQL Server
- Oracle

Out of scope for September MVP:

- AWS collector
- Azure collector
- GCP collector
- Operational Risk Score
- Detailed TCO
- Multi-Cloud What-If
- Veiled Bid Lab
- Vendor SA Concierge

## Core Pipeline

```text
Configuration
  ↓
Pipeline Orchestrator
  ↓
Collector Framework
  ↓
Canonical Asset Model
  ├── Local Discovery Report
  ↓
Privacy Engine
  ↓
Workload Fingerprint
  ↓
Optional Cloud Publisher
  ↓
Amber Navigator
```

## Design Intent

Lens discovers facts. Navigator makes decisions.

Lens should remain deterministic, auditable, local-first, and safe to run in sensitive enterprise environments.
