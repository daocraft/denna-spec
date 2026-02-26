# Denna Specification

Denna is an open standard for storing protocol configuration, parameters, and governance documents as structured, validated, and publicly versioned JSON files.

## What is a `.denna-spec.json` file?

A self-describing JSON document containing:
- A `$schema` field pointing to a JSON Schema that validates the file
- A `metadata` block identifying what the file is and who owns it
- Any domain-defined fields containing the actual data — chains, rates, addresses, governance documents, anything

```json
{
  "$schema": "../schemas/star-config.schema.json",
  "metadata": {
    "id": "spark-config",
    "name": "Spark",
    "kind": "io.denna.defi.star-config",
    "version": "1.0.0",
    "lastUpdated": "2026-02-21"
  },
  "chains": [...],
  "allocations": {...}
}
```

## Read the Specification

See [denna-spec.md](denna-spec.md) for the full specification, or visit [spec.denna.io](https://spec.denna.io).

## Shared Type Definitions

The file `schemas/v1/denna-types.schema.json` provides reusable JSON Schema `$defs` for common value types:

- **address** — blockchain address with format (`{ "value": "0x...", "format": "evm" }`)
- **rate** — value with unit (`{ "value": 30, "unit": "bps" }`)
- **amount** — value with currency (`{ "value": 1000000000, "currency": "USD" }`)
- **duration** — value with time unit (`{ "value": 24, "unit": "months" }`)
- **date** — ISO 8601 date string (`"2026-01-01"`)
- **chain** — lowercase blockchain identifier (`"ethereum"`)
- **metadata** — the standard metadata block

Import these in your domain schemas via `$ref`:

```json
{
  "$ref": "https://spec.denna.io/schemas/v1/denna-types.schema.json#/$defs/address"
}
```

## GitHub Action

Use the Denna validator in your CI pipeline:

```yaml
- uses: denna-spec/denna-spec@v1
  with:
    patterns: "**/*.denna-spec.json"
    exclude: "_template/**"
```

## Kind Identifiers

The `metadata.kind` field uses a reverse-domain convention (inspired by AT Protocol NSIDs):

| Kind | Description |
|------|-------------|
| `io.denna.defi.star-config` | DeFi protocol chain config and allocations |
| `io.denna.defi.rates` | Shared rate parameters |
| `io.denna.defi.address-registry` | Multi-chain address registries |
| `io.denna.governance.document` | Governance and constitutional documents |

Use your own namespace for custom schemas: `io.sky.star.config`, `com.spark.pnl.params`, etc.

## Official Schema Library

Official Denna schemas are maintained separately at [schemas.denna.io](https://schemas.denna.io) (source: [denna-spec/denna-labs](https://github.com/denna-spec/denna-labs)).

## License

Creative Commons — CC BY 4.0
