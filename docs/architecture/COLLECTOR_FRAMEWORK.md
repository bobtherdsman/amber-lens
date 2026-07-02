# Collector Framework

## Purpose

The Collector Framework provides one common interface for all source-specific collectors.

September MVP collectors:

- VMware vCenter
- SQL Server
- Oracle

## Collector Interface

```go
type Collector interface {
    Name() string
    SourceClass() SourceClass
    ValidateCredentials(ctx context.Context) error
    Collect(ctx context.Context, window SampleWindow) (*AssetCollection, error)
}
```

## Collector Rules

Collectors must use read-only credentials, avoid application data/table contents/secrets, normalize into the Canonical Asset Model, return warnings where possible, and never write reports, create fingerprints, or upload anything.

## VMware MVP

Collect VM inventory, vCPU, RAM, disk, cluster, host, OS metadata where available, and sampled utilization.

## SQL Server MVP

Collect version, edition, patch level, database sizes, wait stats, and feature flags such as Linked Servers, CLR, Service Broker, and AlwaysOn.

## Oracle MVP

Collect version, edition, schema sizes, wait events, RAC indicators, Data Guard indicators, options/packs indicators, and optional AWR only when explicitly enabled.
