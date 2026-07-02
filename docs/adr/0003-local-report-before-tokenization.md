# ADR 0003: Generate Local Report Before Cloud-Bound Tokenization

## Status

Accepted

## Context

The Discovery Report is generated locally and is meant for the customer.

## Decision

The local Discovery Report can be generated from the Canonical Asset Model before cloud-bound tokenization is required. Cloud-bound output must always be tokenized.

## Consequences

Positive: local report is more useful and privacy boundary is clearer.

Tradeoff: implementation must clearly separate local artifacts from cloud-bound artifacts.
