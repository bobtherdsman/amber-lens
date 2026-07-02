# Canonical Asset Model

## Purpose

The Canonical Asset Model is Amber Lens' internal normalized representation of the customer's environment.

It is the single internal language used by the Discovery Report renderer, Privacy Engine, Workload Fingerprint generator, and future local recommendation logic.

## Important Distinction

The Canonical Asset Model is local and may contain raw identifiers. The Workload Fingerprint is cloud-bound and must not contain raw identifiers.

```text
Collectors
  ↓
Canonical Asset Model
  ├── Local Discovery Report
  ↓
Privacy Engine
  ↓
Workload Fingerprint
```

## Top-Level Shape

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

## ComputeAsset

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

## DatabaseAsset

```go
type DatabaseAsset struct {
    Source        string
    NativeID      string
    Engine        string
    Version       string
    Edition       string
    InstanceName  string
    DatabaseNames []string
    SizeGB        float64
    FeatureFlags  []string
    Metrics       []MetricPoint
}
```

## Versioning

The model should be versioned from the beginning, for example `canonical_model_version: "1.0.0"`.
