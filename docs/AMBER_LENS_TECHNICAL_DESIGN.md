# Amber Lens Technical Design

Version: v1.0  
Target repository: `ambercloudai/amber-lens`  
Recommended path: `docs/AMBER_LENS_TECHNICAL_DESIGN.md`  
Status: MVP design brief for September build

---

## 1. Purpose

Amber Lens is the free, open-source Foundation collector for Project Amber. Its job is to run inside the customer environment, collect read-only operational metadata, remove customer-identifying information locally, generate a normalized Workload Fingerprint, and render a local Discovery Report.

Amber Lens is not a SaaS scanner. It is the customer-controlled Trust Anchor for Amber's Local-First, Cloud-Last architecture.

The September MVP should focus on five foundations:

1. Collector framework
2. Tokenization engine
3. Workload Fingerprint generation
4. Discovery Report renderer
5. Optional cloud submission

Everything else, including Operational Risk Score, detailed TCO, Multi-Cloud What-If, and Veiled Bid Lab, belongs to Amber Navigator.

---

## 2. Design Principles

### 2.1 Local-First, Cloud-Last

Amber Lens must run locally in the customer's DMZ, VPC, or trusted environment. By default, it must not send telemetry, identifiers, credentials, or raw discovery output to Amber Cloud.

### 2.2 Read-Only by Design

All connectors must use read-only credentials. Lens should fail fast if it detects credentials or permissions that allow write operations.

### 2.3 Tokenize Before Publish

No raw hostname, IP address, database name, schema name, cluster name, account name, subscription ID, project ID, or customer-identifying metadata should leave the local runtime. Tokenization must happen before report generation or cloud submission.

### 2.4 One Contract Everywhere

Collectors should not feed the report or cloud publisher directly. All downstream functions must consume the Workload Fingerprint.

```text
Collectors
   ↓
Canonical Asset Model
   ↓
Tokenization Engine
   ↓
Workload Fingerprint
   ↓
Discovery Report / Optional Cloud Submission
```

### 2.5 Foundation vs Navigator Boundary

Amber Foundation may include inventory, configuration, utilization summaries, preliminary observations, and right-sizing recommendations.

Amber Navigator owns validated Operational Risk Score, detailed TCO, Multi-Cloud What-If, vendor bids, and paid collaboration workflows.

---

## 3. High-Level Architecture

```text
+-----------------------------+
| Customer Environment        |
|                             |
|  +-----------------------+  |
|  | Amber Lens CLI        |  |
|  +----------+------------+  |
|             |               |
|             v               |
|  +-----------------------+  |
|  | Collector Framework   |  |
|  | VMware / SQL / Oracle |  |
|  | AWS / Azure / GCP     |  |
|  +----------+------------+  |
|             |               |
|             v               |
|  +-----------------------+  |
|  | Canonical Asset Model |  |
|  +----------+------------+  |
|             |               |
|             v               |
|  +-----------------------+  |
|  | Tokenization Engine   |  |
|  | Local mapping key     |  |
|  +----------+------------+  |
|             |               |
|             v               |
|  +-----------------------+  |
|  | Workload Fingerprint  |  |
|  +----------+------------+  |
|             |               |
|      +------+-------+       |
|      |              |       |
|      v              v       |
| Discovery       Optional    |
| Report HTML     Cloud Submit|
+-----------------------------+
                         |
                         v
                 +---------------+
                 | Amber Cloud   |
                 | Navigator     |
                 +---------------+
```

---

## 4. Core Components

## 4.1 CLI

The CLI is the primary user interface for Amber Lens.

Expected commands:

```bash
amber-lens scan --config config.yaml
amber-lens report --input fingerprint.json --output discovery-report.html
amber-lens submit --input fingerprint.json --endpoint https://api.ambercloud.ai
amber-lens version
```

MVP expectations:

- Config-file driven execution
- Clear local output paths
- No background telemetry
- Human-readable errors
- Machine-readable JSON logs when requested

---

## 4.2 Collector Framework

The collector framework should expose one interface implemented by all connectors.

```go
type Collector interface {
    Name() string
    SourceClass() SourceClass
    ValidateCredentials(ctx context.Context) error
    Collect(ctx context.Context, window SampleWindow) (*AssetCollection, error)
}
```

Collectors should return a Canonical Asset Model, not a final fingerprint.

### MVP connectors

| Connector | Source | MVP data |
|---|---|---|
| VMware | vCenter REST / vim25 fallback | VM inventory, vCPU, RAM, disk, cluster, host metrics |
| SQL Server | DMVs | version, edition, database sizes, wait stats, feature flags |
| Oracle | V$ views, optional AWR | version, edition, schema sizes, wait events, RAC/Data Guard indicators |
| AWS | CloudWatch / Cost APIs | EC2, RDS, EBS metrics, basic billing metadata |
| Azure | Azure Monitor / Cost Management | VM, SQL, storage metrics, basic cost metadata |
| GCP | Cloud Monitoring / Billing Export | Compute, Cloud SQL, storage metrics, basic cost metadata |

### Connector rules

- Use read-only permissions only
- Never collect application table contents
- Never collect user data
- Never collect secrets
- Normalize source-specific output into the Canonical Asset Model
- Continue partial collection when one source fails, but mark the collection as degraded

---

## 4.3 Canonical Asset Model

The Canonical Asset Model is the internal, pre-tokenization representation used to normalize data from all connectors.

Core entities:

```text
AssetCollection
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

The Canonical Asset Model may contain raw identifiers because it exists only inside the local runtime. It must not be transmitted or written to cloud-bound output without tokenization.

---

## 4.4 Tokenization Engine

The Tokenization Engine converts customer-identifying fields into deterministic tokens.

### Responsibilities

- Tokenize hostnames
- Tokenize IP addresses
- Tokenize database names
- Tokenize schema names
- Tokenize cluster names
- Tokenize account, subscription, and project identifiers
- Preserve relationships between assets
- Keep mapping keys local
- Produce a sovereignty receipt

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

## 4.5 Workload Fingerprint Generator

The Workload Fingerprint is the canonical tokenized payload produced by Amber Lens.

It is not a raw scan. It is not the Discovery Report. It is the normalized, identity-stripped technical profile used by Foundation, Navigator, and future services.

Minimum schema fields:

```json
{
  "fingerprint_id": "uuidv7",
  "schema_version": "1.0.0",
  "token_namespace": "opaque-customer-derived-namespace",
  "source_class": "vcenter|sqlserver|oracle|cloudwatch|azuremonitor|gcpops|mixed",
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

The fingerprint should support mixed-source estates. For example, a customer may scan VMware plus SQL Server plus AWS.

---

## 4.6 Discovery Report Renderer

The Discovery Report is a local HTML artifact generated from the Workload Fingerprint.

Rules:

- Self-contained HTML
- Inline CSS
- No external assets
- No remote JavaScript
- No tracking pixels
- No telemetry
- Renderable offline

MVP sections:

1. Executive summary
2. Sovereignty receipt
3. Estate at a glance
4. Compute inventory
5. Database inventory
6. Storage summary
7. OS and end-of-life observations
8. Utilization summary
9. Preliminary right-sizing opportunities
10. Licensing and modernization observations
11. Collection warnings
12. Follow-up questions
13. Navigator upgrade boundary

The report must consume the Workload Fingerprint only. It should not call collectors directly.

---

## 4.7 Optional Cloud Submission

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

## 5. Data Flow

```text
1. User provides local config file.
2. Lens validates read-only credentials.
3. Collectors gather operational metadata.
4. Collector outputs normalize into Canonical Asset Model.
5. Tokenization Engine replaces identifiers with deterministic tokens.
6. Fingerprint Generator creates Workload Fingerprint JSON.
7. Report Renderer creates local Discovery Report HTML.
8. Optional: user submits tokenized fingerprint to Amber Cloud.
9. Navigator consumes fingerprint for paid analysis.
```

---

## 6. Security Requirements

### Must have

- Read-only credential validation
- Local-only mapping key
- No default outbound network calls
- No application data collection
- No table contents collection
- No secrets in logs
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

## 7. MVP Definition of Done

Amber Lens MVP is complete when a user can:

1. Download or build a single Go binary.
2. Configure at least VMware, SQL Server, Oracle, AWS, Azure, and GCP connectors.
3. Run collection with read-only credentials.
4. Generate a tokenized Workload Fingerprint.
5. Generate a local Discovery Report HTML.
6. Prove no mapping key or raw identifiers leave the environment.
7. Optionally submit the tokenized fingerprint to Amber Cloud.
8. Pass CI build, unit tests, license scan, SBOM generation, and security checks.

---

## 8. Recommended September MVP Scope

### Build now

- Collector framework
- Six MVP connectors
- Tokenization engine
- Fingerprint generation
- Discovery Report renderer
- Optional cloud submission
- CI tests and fixtures

### Do not build in Lens

- Operational Risk Score
- Detailed TCO
- Multi-Cloud What-If
- Veiled Bid Lab
- Vendor SA Concierge
- Expert workflow
- Production FinOps dashboard

These belong to Navigator or Services.

---

## 9. Testing Strategy

### Unit tests

- Token generation determinism
- Token salt rotation
- Mapping key persistence
- Schema validation
- Report rendering
- Credential validation logic

### Fixture tests

Use static fixtures for:

- vCenter exports
- SQL Server DMV outputs
- Oracle V$ / AWR-like outputs
- AWS CloudWatch outputs
- Azure Monitor outputs
- GCP Monitoring outputs

### Integration tests

- End-to-end sample scan
- Fingerprint generation
- Discovery Report generation
- Optional cloud submission to mock endpoint

### Security tests

- No raw identifiers in fingerprint
- No mapping key in outbound payload
- No secrets in logs
- No unauthorized outbound calls
- License scan blocks GPL/AGPL dependencies

---

## 10. Engineering Sequence

### Sprint 1: Core contracts

- Finalize Canonical Asset Model
- Finalize Workload Fingerprint schema
- Define collector interface
- Add fixture-based test harness

### Sprint 2: Tokenization and report path

- Implement tokenization engine
- Implement fingerprint generator
- Implement Discovery Report renderer
- Generate report from fixture data

### Sprint 3: First real connectors

- VMware MVP connector
- SQL Server MVP connector
- Oracle MVP connector

### Sprint 4: Cloud connectors

- AWS MVP connector
- Azure MVP connector
- GCP MVP connector

### Sprint 5: Hardening

- Read-only validation
- Packet-capture tests
- SBOM and license gates
- Release packaging
- Documentation

---

## 11. Open Questions

1. Should the Canonical Asset Model be public or internal only?
2. Should the Discovery Report companion JSON be the same as the Workload Fingerprint or a report-specific projection?
3. How much cloud cost metadata should Foundation collect before crossing into Navigator value?
4. Should Oracle AWR be optional-only to reduce licensing concern?
5. Should cloud submission use signed one-time upload URLs or direct API authentication?
6. Should customer-provided salt namespace be reused across scans or rotated by default?

---

## 12. Final Recommendation

Treat Amber Lens as a fingerprint generator, not a universal reporting tool.

The repository should optimize for one clean pipeline:

```text
Collect → Normalize → Tokenize → Fingerprint → Report → Optional Submit
```

If this pipeline is reliable, auditable, and easy to run, Amber Foundation can credibly launch by September. Navigator can then build risk, TCO, what-if, and bid workflows on top of the same fingerprint contract without forcing a redesign of the collector.
