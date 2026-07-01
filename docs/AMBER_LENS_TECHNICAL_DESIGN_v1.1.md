# Amber Lens Technical Design

Version: v1.1  
Target repository: `ambercloudai/amber-lens`  
Recommended path: `docs/AMBER_LENS_TECHNICAL_DESIGN.md`  
Status: MVP design brief for September build

---

## 1. Purpose

Amber Lens is the free, open-source Foundation collector for Project Amber. Its job is to run inside the customer environment, collect read-only operational metadata from on-premises infrastructure and databases, generate a local Discovery Report, remove customer-identifying information locally when producing a cloud-bound payload, and generate a normalized Workload Fingerprint for optional Amber Navigator analysis.

Amber Lens is not a SaaS scanner. It is the customer-controlled Trust Anchor for Amber's Local-First, Cloud-Last architecture.

The September MVP should focus on five foundations:

1. Collector framework
2. Tokenization engine / privacy engine
3. Workload Fingerprint generation
4. Discovery Report renderer
5. Optional cloud submission

Everything else, including Operational Risk Score, detailed TCO, Multi-Cloud What-If, Veiled Bid Lab, Vendor SA Concierge, and cloud-to-cloud collectors, belongs to Amber Navigator or a later Amber Lens release.

---

## 2. September MVP Scope

### In scope for Amber Lens MVP

Amber Lens MVP collects from three source families:

| Source | API / Surface | MVP data |
|---|---|---|
| VMware vCenter | vSphere REST API, vim25 fallback | VM inventory, vCPU, RAM, disk, clusters, hosts, sampled utilization, vMotion / event signals where available |
| SQL Server | DMVs and system catalog views | version, edition, database sizes, wait stats, query-plan metadata where safe, patch level, feature flags such as Linked Servers, CLR, Service Broker, AlwaysOn |
| Oracle | V$ views, optional AWR where customer is licensed and explicitly enables it | version, edition, schema sizes, wait events, top SQL metadata where safe, RAC / Data Guard indicators, options / packs indicators |

### Out of scope for Amber Lens MVP

The September MVP does **not** collect from:

- AWS CloudWatch
- AWS Cost Explorer / Billing
- Azure Monitor
- Azure Cost Management
- GCP Cloud Monitoring
- GCP Billing Export

Cloud-to-cloud assessment remains strategically important, but it is not part of the initial Amber Lens Foundation collector. It should be treated as a later expansion after the on-premises VMware / SQL Server / Oracle path is reliable, auditable, and useful.

---

## 3. Design Principles

### 3.1 Local-First, Cloud-Last

Amber Lens must run locally in the customer's DMZ, data center, VPC, or trusted environment. By default, it must not send telemetry, identifiers, credentials, raw discovery output, or logs to Amber Cloud.

### 3.2 Read-Only by Design

All connectors must use read-only credentials. Lens should fail fast if it detects credentials or permissions that allow write operations.

### 3.3 Local Report First

The Discovery Report is a local artifact for the customer. It may show customer-readable names locally if the customer explicitly chooses that mode. The default open-source posture should still be privacy-safe and tokenized where practical.

The important boundary is cloud-bound output:

> Anything leaving the local environment must be tokenized and must not contain raw hostnames, IP addresses, database names, schema names, cluster names, customer names, credentials, secrets, or table contents.

### 3.4 Tokenize Before Cloud Publish

Tokenization must happen before optional cloud submission. Cloud submission can only use the tokenized Workload Fingerprint.

### 3.5 One Contract for Cloud-Bound Intelligence

Collectors should not feed the cloud publisher directly. Amber Navigator should consume the Workload Fingerprint, not raw collector output.

```text
Collectors
   ↓
Canonical Asset Model
   ├── Local Discovery Report
   ↓
Privacy / Tokenization Engine
   ↓
Workload Fingerprint
   ↓
Optional Cloud Submission
   ↓
Amber Navigator
```

### 3.6 Foundation vs Navigator Boundary

Amber Foundation may include inventory, configuration, utilization summaries, preliminary observations, and right-sizing recommendations.

Amber Navigator owns validated Operational Risk Score, detailed TCO, Multi-Cloud What-If, vendor bids, paid collaboration workflows, and cloud-to-cloud assessment logic.

---

## 4. High-Level Architecture

```text
+------------------------------------------------+
| Customer Environment                           |
|                                                |
|  +-----------------------+                     |
|  | Amber Lens CLI        |                     |
|  +----------+------------+                     |
|             |                                  |
|             v                                  |
|  +-----------------------+                     |
|  | Pipeline Orchestrator |                     |
|  +----------+------------+                     |
|             |                                  |
|             v                                  |
|  +-----------------------+                     |
|  | Collector Framework   |                     |
|  | VMware / SQL / Oracle |                     |
|  +----------+------------+                     |
|             |                                  |
|             v                                  |
|  +-----------------------+                     |
|  | Canonical Asset Model |                     |
|  +----------+------------+                     |
|             |                                  |
|      +------+------------------+               |
|      |                         |               |
|      v                         v               |
| Local Discovery          Privacy / Tokenization|
| Report Renderer          Engine                |
|                                |               |
|                                v               |
|                       Workload Fingerprint     |
|                                |               |
|                                v               |
|                       Optional Cloud Submit    |
+------------------------------------------------+
                                 |
                                 v
                         +---------------+
                         | Amber Cloud   |
                         | Navigator     |
                         +---------------+
```

---

## 5. Core Components

## 5.1 CLI

The CLI is the primary user interface for Amber Lens.

Expected commands:

```bash
amber-lens scan --config config.yaml
amber-lens report --input collection.json --output discovery-report.html
amber-lens fingerprint --input collection.json --output fingerprint.json
amber-lens submit --input fingerprint.json --endpoint https://api.ambercloud.ai
amber-lens version
```

MVP expectations:

- Config-file driven execution
- Clear local output paths
- No background telemetry
- Human-readable errors
- Machine-readable JSON logs when requested
- Explicit confirmation before any cloud submission

---

## 5.2 Pipeline Orchestrator

The Pipeline Orchestrator coordinates the end-to-end local run.

Responsibilities:

- Load and validate configuration
- Start collectors
- Coordinate the sample window
- Track progress
- Handle partial failures
- Write local artifacts
- Invoke report generation
- Invoke fingerprint generation
- Ensure no cloud submission happens unless explicitly requested

The orchestrator owns control flow. Individual collectors should not manage global execution.

---

## 5.3 Collector Framework

The collector framework should expose one interface implemented by all connectors.

```go
type Collector interface {
    Name() string
    SourceClass() SourceClass
    ValidateCredentials(ctx context.Context) error
    Collect(ctx context.Context, window SampleWindow) (*AssetCollection, error)
}
```

Collectors return the Canonical Asset Model, not a final fingerprint and not a report.

### MVP connectors

| Connector | Source | MVP data |
|---|---|---|
| VMware | vCenter REST / vim25 fallback | VM inventory, vCPU, RAM, disk, cluster, host metrics, utilization |
| SQL Server | DMVs and system catalog views | version, edition, database sizes, wait stats, patch level, feature flags |
| Oracle | V$ views, optional AWR | version, edition, schema sizes, wait events, RAC / Data Guard indicators, options / packs indicators |

### Connector rules

- Use read-only permissions only
- Never collect application table contents
- Never collect customer business data
- Never collect secrets
- Normalize source-specific output into the Canonical Asset Model
- Continue partial collection when one source fails, but mark the collection as degraded
- Make Oracle AWR explicitly opt-in because licensing posture varies by customer

---

## 5.4 Canonical Asset Model

The Canonical Asset Model is the internal representation used to normalize data from all MVP connectors.

It is local runtime data. It may contain raw identifiers because it is used to produce the customer's local report and to preserve relationships before tokenization.

Core entities:

```text
AssetCollection
├── EnvironmentMetadata
├── ComputeAssets
├── DatabaseAssets
├── StorageAssets
├── NetworkDependencies
├── MetricSeries
├── LicensingFootprint
├── SourceMetadata
└── CollectionWarnings
```

Example fields:

```go
type ComputeAsset struct {
    Source          string
    NativeID        string
    Name            string
    Hostname        string
    IPAddresses     []string
    CPUCores        int
    MemoryGB        float64
    StorageGB       float64
    OperatingSystem string
    Cluster         string
    Metrics         []MetricPoint
}
```

The Canonical Asset Model must not be transmitted to Amber Cloud unless it is transformed into a tokenized Workload Fingerprint.

---

## 5.5 Discovery Report Renderer

The Discovery Report is a local HTML artifact generated from the Canonical Asset Model and/or a local report projection.

Rules:

- Self-contained HTML
- Inline CSS
- No external assets
- No remote JavaScript
- No tracking pixels
- No telemetry
- Renderable offline
- Customer-controlled identity mode:
  - `local-readable`: may show real names because the report stays local
  - `privacy-safe`: shows tokens / obfuscated names

MVP sections:

1. Executive summary
2. Sovereignty receipt / local-run receipt
3. Estate at a glance
4. Compute inventory
5. Database inventory
6. Storage summary
7. OS and end-of-life observations
8. Utilization summary
9. Preliminary right-sizing opportunities
10. SQL Server modernization observations
11. Oracle licensing / feature observations
12. Collection warnings
13. Follow-up questions
14. Navigator upgrade boundary

The report should not call collectors directly. It should consume a stable local collection artifact or report projection generated by the pipeline.

---

## 5.6 Privacy / Tokenization Engine

The Privacy / Tokenization Engine converts customer-identifying fields into deterministic tokens for cloud-bound output.

### Responsibilities

- Tokenize hostnames
- Tokenize IP addresses
- Tokenize database names
- Tokenize schema names
- Tokenize cluster names
- Tokenize account-like identifiers if present
- Preserve relationships between assets
- Keep mapping keys local
- Produce a sovereignty receipt
- Scrub secrets and credentials from logs and payloads

### Two-pass tokenization

Pass 1: Use HMAC-SHA-256 with a customer-controlled local mapping key.

Pass 2: Apply per-publish salt rotation so tokens are stable within one publish but not easily correlatable across unrelated publishes.

Example:

```text
SQLPROD01      → host-8f31a2
PayrollDB      → db-41c9aa
10.10.15.42    → ip-29d113
```

The local mapping file remains on customer disk. Amber Cloud receives only tokenized values.

### Sovereignty receipt

Lens should emit a SHA-256 hash of the local mapping-key file as a receipt. The receipt allows the customer to verify that the key remained local without exposing the key itself.

---

## 5.7 Workload Fingerprint Generator

The Workload Fingerprint is the canonical tokenized payload produced by Amber Lens for optional cloud submission and Navigator analysis.

It is not a raw scan. It is not the Discovery Report. It is the normalized, identity-stripped technical profile used by Navigator and future services.

Minimum schema fields:

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

The fingerprint should support mixed-source estates. For example, a customer may scan VMware plus SQL Server plus Oracle in a single run.

---

## 5.8 Optional Cloud Submission

Cloud submission is customer-initiated only.

Command:

```bash
amber-lens submit --input fingerprint.json --endpoint https://api.ambercloud.ai
```

Rules:

- Never automatic
- TLS 1.3
- Submit tokenized fingerprint only
- No local mapping key
- No raw identifiers
- No credentials
- No raw logs
- Retries must not duplicate fingerprint records
- Submission should return a receipt ID

Cloud submission enables Amber Navigator but is not required for Foundation use.

---

## 6. Data Flow

```text
1. User provides local config file.
2. Lens validates read-only credentials.
3. Pipeline Orchestrator starts collectors.
4. VMware / SQL Server / Oracle collectors gather operational metadata.
5. Collector outputs normalize into the Canonical Asset Model.
6. Local Discovery Report is generated for the customer.
7. If requested, Privacy Engine tokenizes identifiers.
8. Fingerprint Generator creates Workload Fingerprint JSON.
9. Optional: user submits tokenized fingerprint to Amber Cloud.
10. Navigator consumes fingerprint for paid analysis.
```

---

## 7. Security Requirements

### Must have

- Read-only credential validation
- Local-only mapping key
- No default outbound network calls
- No application data collection
- No table contents collection
- No secrets in logs
- No raw identifiers in cloud-bound payloads
- SBOM per release
- License gate blocking GPL/AGPL dependencies
- Structured logs with PII scrubbing
- Packet-capture test proving mapping key is not transmitted

### Should have

- Signed releases
- Checksums for binaries
- Reproducible builds
- Vulnerability disclosure policy
- Security audit before GA

---

## 8. MVP Definition of Done

Amber Lens MVP is complete when a user can:

1. Download or build a single Go binary.
2. Configure VMware, SQL Server, and Oracle connectors.
3. Run collection with read-only credentials.
4. Generate a local Discovery Report HTML.
5. Generate a tokenized Workload Fingerprint.
6. Prove no mapping key or raw identifiers leave the environment during optional cloud submission.
7. Optionally submit the tokenized fingerprint to Amber Cloud.
8. Pass CI build, unit tests, license scan, SBOM generation, and security checks.

---

## 9. Recommended September MVP Scope

### Build now

- Pipeline orchestrator
- Collector framework
- VMware MVP connector
- SQL Server MVP connector
- Oracle MVP connector
- Canonical Asset Model
- Local Discovery Report renderer
- Privacy / tokenization engine
- Workload Fingerprint generator
- Optional cloud submission
- CI tests and fixtures

### Do not build in Lens now

- AWS collector
- Azure collector
- GCP collector
- Operational Risk Score
- Detailed TCO
- Multi-Cloud What-If
- Veiled Bid Lab
- Vendor SA Concierge
- Expert workflow
- Production FinOps dashboard

These belong to Navigator, Services, or a later Lens release.

---

## 10. Testing Strategy

### Unit tests

- Token generation determinism
- Token salt rotation
- Mapping key persistence
- Schema validation
- Report rendering
- Credential validation logic
- Canonical Asset Model mapping

### Fixture tests

Use static fixtures for:

- vCenter exports
- SQL Server DMV outputs
- Oracle V$ outputs
- Oracle AWR-like outputs where licensed and enabled

### Integration tests

- End-to-end sample scan using fixtures
- Local Discovery Report generation
- Fingerprint generation
- Optional cloud submission to mock endpoint

### Security tests

- No raw identifiers in fingerprint
- No mapping key in outbound payload
- No secrets in logs
- No unauthorized outbound calls
- License scan blocks GPL/AGPL dependencies

---

## 11. Engineering Sequence

### Sprint 1: Core contracts

- Finalize Canonical Asset Model
- Finalize Workload Fingerprint schema
- Define collector interface
- Add fixture-based test harness

### Sprint 2: Local report path

- Implement pipeline orchestrator
- Generate local collection artifact
- Implement Discovery Report renderer
- Generate report from fixture data

### Sprint 3: Privacy and fingerprint path

- Implement tokenization engine
- Implement sovereignty receipt
- Implement fingerprint generator
- Validate no raw identifiers in fingerprint

### Sprint 4: First real connectors

- VMware MVP connector
- SQL Server MVP connector
- Oracle MVP connector

### Sprint 5: Hardening

- Read-only validation
- Packet-capture tests
- SBOM and license gates
- Release packaging
- Documentation

---

## 12. Open Questions

1. Should the Canonical Asset Model be public or internal only?
2. Should the Discovery Report companion JSON be the Canonical Asset Model, a report-specific projection, or a tokenized fingerprint projection?
3. Should local reports default to `privacy-safe` mode or `local-readable` mode?
4. Should Oracle AWR be optional-only to reduce licensing concern?
5. Should cloud submission use signed one-time upload URLs or direct API authentication?
6. Should customer-provided salt namespace be reused across scans or rotated by default?

---

## 13. Final Recommendation

Treat Amber Lens as a local workload intelligence pipeline, not a universal reporting tool and not a cloud scanner.

The repository should optimize for one clean pipeline:

```text
Collect → Normalize → Report Locally → Tokenize → Fingerprint → Optional Submit
```

If this pipeline is reliable, auditable, and easy to run for VMware, SQL Server, and Oracle, Amber Foundation can credibly launch by September. Navigator can then build risk, TCO, what-if, cloud-to-cloud assessment, and bid workflows on top of the same fingerprint contract without forcing a redesign of the collector.
