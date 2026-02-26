# Denna Specification 1.0.0

## Summary

Denna Spec is a convention for expressing any structured data as self-describing JSON files validated by [JSON Schema](https://json-schema.org/). A Denna file (`.denna-spec.json`) is a JSON document containing a metadata block that identifies what it is, and any number of domain-defined fields containing the actual data. Its structure is validated by a standard JSON Schema referenced via the `$schema` field.

Denna Spec does not replace JSON Schema — it is a thin orchestration layer on top of it. JSON Schema handles all structural validation. Denna defines the conventions JSON Schema alone cannot express: file naming, a standard metadata block, semantic value types, a schema namespace convention, and organizational guidance.

## Introduction

Protocols and financial systems manage critical data — wallet addresses, rate spreads, governance rules, asset allocations, constitutional documents — that often lives in unstructured form: spreadsheets, markdown files, Notion pages, or scattered across codebases. This makes the data hard to validate, impossible to audit, and incompatible with each other. Every team builds the same tooling from scratch.

Denna Spec solves this by providing a minimal standard for expressing any structured data as self-describing JSON. Because all Denna Spec files share the same envelope (`$schema` + `metadata`), any Denna Spec-aware tool — a governance UI, a validation pipeline, an analytics dashboard, a risk monitor — can discover, parse, and validate them without prior knowledge of their contents. The schema tells the tool everything it needs to know.

This follows the same design principle as the [AT Protocol](https://atproto.com/): standardize how data is described once, and any application built on the standard gains interoperability for free. Data lives in the owner's repository. Any tool that speaks Denna Spec can read it.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

### 1. File Naming

1. A Denna file MUST use the extension `.denna-spec.json`.
2. The file name preceding the extension SHOULD be descriptive and use kebab-case. Examples: `config.denna-spec.json`, `pnl-params.denna-spec.json`, `stablecoin-addresses.denna-spec.json`.
3. Files intended to be validated by automated tooling MUST use the `.denna-spec.json` extension to be discovered by validators and CI pipelines.

### 2. File Structure

A Denna file MUST be a valid JSON object containing the following top-level fields:

```json
{
  "$schema": "path/or/uri/to/schema.json",
  "metadata": { }
}
```

Beyond `$schema` and `metadata`, all other fields are defined by the domain schema referenced in `$schema`. The domain schema MAY define any additional fields using any names.

#### 2.1. `$schema` (REQUIRED)

A string containing a relative path or absolute URI to a JSON Schema file that validates the entire Denna file. This is standard JSON Schema behavior — it enables IDE auto-completion, inline validation, and tells validators which schema to use.

The referenced schema MUST validate the complete file including `metadata` and any domain content fields.

#### 2.2. `metadata` (REQUIRED)

An object describing the file's identity and provenance. The following fields are defined:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | REQUIRED | A unique identifier for this file within its repository. MUST be kebab-case (`^[a-z0-9]+(-[a-z0-9]+)*$`). |
| `name` | string | REQUIRED | A human-readable display name. |
| `kind` | string | REQUIRED | A reverse-domain identifier classifying the type of data (e.g., `io.denna.defi.prime-config`, `io.sky.prime.config`). See [Section 3](#3-kind-identifiers). |
| `description` | string | OPTIONAL | A brief description of what this file contains. |
| `version` | string | OPTIONAL | A [Semantic Version](https://semver.org/) string for the data. SHOULD be incremented when parameters change. |
| `lastUpdated` | string | OPTIONAL | An ISO 8601 date (`YYYY-MM-DD`) indicating when the data was last verified or updated. |
| `tags` | array of strings | OPTIONAL | Freeform tags for categorization and filtering. |
| `source` | object | OPTIONAL | Provenance information. MAY contain `repository` (string) and `references` (array of source code references). |

##### Source References

The `source.references` field is an array of strings. References SHOULD use the format `path/to/file.ts:lineNumber` to point to the exact source code location from which a value was derived. Example: `"src/lib/constants.ts:17"`.

#### 2.3. Domain Content Fields

Any fields beyond `$schema` and `metadata` are defined entirely by the domain schema. The domain schema determines what is required, what is optional, and what structure each field takes.

Common field name conventions (not required by Denna):

| Field Name | Typical Use |
|------------|-------------|
| `parameters` | Numeric configuration values, rates, flags |
| `addresses` | Contract or token address registries |
| `chains` | Blockchain network configuration |
| `allocations` | Asset positions or fund deployments |
| `content` | Document body or article text |
| `sections` | Governance document sections |

Implementations SHOULD choose field names that match their data's nature. A file containing governance articles SHOULD use `content` or `sections`. A file containing financial rates SHOULD use `rates` or `parameters`.

### 3. Kind Identifiers

The `metadata.kind` field is a reverse-domain identifier that uniquely classifies the type of data in a Denna file. It follows a naming convention inspired by [AT Protocol NSIDs](https://atproto.com/specs/nsid).

#### 3.1. Format

A `kind` value MUST consist of at least three dot-separated segments. The first two segments SHOULD identify the schema publisher in reverse-domain order.

```
io.denna.defi.prime-config
^^ ^^^^^ ^^^^ ^^^^^^^^^^^
|  |      |    name
|  |      namespace
|  publisher domain
TLD
```

Pattern: `^[a-z][a-z0-9]*(\.[a-z][a-z0-9-]*){2,}$`

#### 3.2. Namespace Conventions

| Namespace | Usage |
|-----------|-------|
| `io.denna.*` | Official schemas published by Denna |
| `io.denna.defi.*` | Official DeFi protocol schemas |
| `io.denna.governance.*` | Official governance and document schemas |
| `io.denna.finance.*` | Official financial parameter schemas |
| `{org-domain}.*` | Custom schemas published by an organization |

Examples:

```
io.denna.defi.prime-config       → official DeFi prime configuration schema
io.denna.defi.rates             → official DeFi rates schema
io.denna.governance.document    → official governance document schema
io.sky.prime.config              → Sky Protocol custom prime config
com.spark.pnl.params            → Spark custom PnL parameters
```

Organizations SHOULD use a namespace derived from a domain they control (reversed). Custom schemas can reference official Denna types via `$ref`.

### 4. Semantic Value Types

When representing the following common value types inside data fields, implementations SHOULD use these conventions. Denna provides reusable JSON Schema definitions at `https://spec.denna.io/schemas/v1/denna-types.schema.json`.

#### 4.1. Address

A blockchain address with its format identifier.

```json
{ "value": "0x1601843c5e9bc251a3272907010afa41fa18347e", "format": "evm" }
{ "value": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", "format": "solana" }
```

The `format` field identifies the address family so it can be validated and displayed correctly. Known formats are pattern-validated by the schema. Unknown formats are accepted without pattern enforcement to allow extensibility.

**Defined formats:**

| Format | Pattern | Description |
|--------|---------|-------------|
| `evm` | `^0x[0-9a-fA-F]{40}$` | Ethereum, Base, Arbitrum, Optimism, Avalanche, and other EVM-compatible chains. |
| `solana` | `^[1-9A-HJ-NP-Za-km-z]{32,44}$` | Solana. Base58-encoded public key. |
| `cosmos` | `^[a-z]{1,20}1[a-z0-9]{38,58}$` | Cosmos SDK chains. Bech32 with human-readable prefix. |
| `bitcoin` | `^(1\|3\|bc1)[a-zA-HJ-NP-Z0-9]{25,62}$` | Bitcoin P2PKH, P2SH, or Bech32. |
| `aptos` | `^0x[0-9a-f]{64}$` | Aptos. 32-byte hex. |
| `sui` | `^0x[0-9a-f]{64}$` | Sui. 32-byte hex. |
| `tron` | `^T[1-9A-HJ-NP-Za-km-z]{33}$` | Tron. Base58Check starting with `T`. |

Implementations MAY use custom format strings not listed above.

#### 4.2. Rate

A rate or spread value with an explicit unit.

```json
{ "value": 30, "unit": "bps" }
{ "value": 4.5, "unit": "percent" }
```

The `unit` field MUST be one of: `bps` (basis points), `percent`.

#### 4.3. Amount

A monetary amount with a currency denomination.

```json
{ "value": 1000000000, "currency": "USD" }
```

The `currency` field SHOULD be an ISO 4217 currency code or a well-known token symbol (e.g., `USDC`, `ETH`).

#### 4.4. Duration

A time duration with an explicit unit.

```json
{ "value": 24, "unit": "months" }
{ "value": 7, "unit": "days" }
```

The `unit` field MUST be one of: `months`, `days`, `hours`, `seconds`.

#### 4.5. Date

A calendar date. MUST be an ISO 8601 date string in `YYYY-MM-DD` format.

```json
"2026-01-01"
```

#### 4.6. Chain Identifier

A blockchain network identifier. MUST be a lowercase string starting with a letter.

```json
"ethereum"
```

Common values: `ethereum`, `base`, `arbitrum`, `optimism`, `unichain`, `avalanche`, `plume`, `monad`, `solana`. Implementations MAY define additional chain identifiers as needed.

### 5. Domain Schemas

A domain schema is a JSON Schema file that defines the complete structure of a Denna file for a specific use case. Domain schemas:

1. MUST validate the full file including `$schema`, `metadata`, and all domain content fields.
2. SHOULD import shared type definitions from `https://spec.denna.io/schemas/v1/denna-types.schema.json` via `$ref`.
3. SHOULD use JSON Schema draft 2020-12 or later.
4. MAY be stored in the same repository as the data files or published separately.
5. SHOULD be versioned at stable paths (e.g., `schemas/v1/`, `schemas/v2/`).

Example of a domain schema importing Denna types:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.denna.io/v1/defi/prime-config.schema.json",
  "type": "object",
  "required": ["$schema", "metadata", "chains"],
  "properties": {
    "$schema": { "type": "string" },
    "metadata": {
      "$ref": "https://spec.denna.io/schemas/v1/denna-types.schema.json#/$defs/metadata"
    },
    "chains": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {
            "$ref": "https://spec.denna.io/schemas/v1/denna-types.schema.json#/$defs/chain"
          },
          "almProxy": {
            "$ref": "https://spec.denna.io/schemas/v1/denna-types.schema.json#/$defs/address"
          }
        }
      }
    }
  }
}
```

### 6. Schema Ecosystem

Denna Spec is designed around an open schema ecosystem. Any organization can publish schemas that other Denna Spec files reference. No central registry is required.

#### 6.1. Official Schemas (Denna)

The Denna project maintains an open-source library of reusable schemas for common use cases, published at `https://schemas.denna.io/`. These use the `io.denna.*` namespace.

Currently available:

| Schema | Kind | Description |
|--------|------|-------------|
| `io.denna.defi.prime-config` | DeFi prime/protocol configuration | Chain configuration, wallet addresses, asset allocations |
| `io.denna.defi.rates` | Shared rate parameters | Interest rates, spreads, subsidy programs |
| `io.denna.defi.address-registry` | Address registry | Multi-chain token and contract address sets |
| `io.denna.governance.document` | Governance document | Articles, clauses, constitutional documents |

#### 6.2. Custom Schemas

Organizations are encouraged to publish their own schemas. Custom schemas:

- MAY reference official Denna types as building blocks
- SHOULD be hosted at a stable, public URL
- SHOULD use the organization's reverse-domain namespace
- MAY extend official schemas with additional fields

#### 6.3. Local Schemas

During development or for private use cases, schemas MAY be stored locally in the same repository as the data files and referenced with relative paths:

```json
{ "$schema": "../../schemas/prime-config.schema.json" }
```

### 7. File Organization

Denna does not mandate a directory structure, but RECOMMENDS the following:

1. Group data files by entity — one directory per protocol, prime, or logical unit.
2. Place shared or global parameters in a `shared/` directory.
3. Place domain schemas in a `schemas/` directory at the repository root.
4. Use descriptive file names that reflect content: `config.denna-spec.json`, `rates.denna-spec.json`, `allocations.denna-spec.json`.

### 8. Versioning

1. The Denna Specification itself follows [Semantic Versioning](https://semver.org/). The current version is **1.0.0**.
2. Individual data files SHOULD include a `metadata.version` field to track parameter changes over time.
3. Domain schemas SHOULD be versioned at stable paths (`schemas/v1/`, `schemas/v2/`).
4. A MAJOR version increment in a domain schema indicates breaking changes to the `parameters` structure. Consumers SHOULD check schema compatibility before processing files.

### 9. Validation

1. A Denna file is valid if and only if it passes JSON Schema validation against the schema referenced in its `$schema` field.
2. Validators MUST resolve the `$schema` field and validate the entire file against the resolved schema.
3. If the `$schema` field is missing or the referenced schema cannot be resolved, the file MUST be treated as invalid.

## Complete Example

This example shows a DeFi protocol managing chain configuration and rate parameters using Denna.

### Repository structure

```
primes-parameters/
├── schemas/
│   ├── prime-config.schema.json        # Defines structure for prime config files
│   └── shared-rates.schema.json        # Defines structure for shared rate files
├── spark/
│   └── config.denna-spec.json          # Spark's chain config and allocations
├── shared/
│   └── rates.denna-spec.json           # Cross-protocol rates and constants
└── .github/workflows/
    └── validate.yml                    # Validates *.denna-spec.json on every PR
```

### Data file: `spark/config.denna-spec.json`

```json
{
  "$schema": "../schemas/prime-config.schema.json",
  "metadata": {
    "id": "spark-config",
    "name": "Spark",
    "kind": "io.denna.defi.prime-config",
    "description": "Spark chain configuration, wallets, and allocation positions.",
    "version": "1.0.0",
    "lastUpdated": "2026-02-21",
    "tags": ["prime", "defi"],
    "source": {
      "repository": "primes-parameters",
      "references": ["src/lib/constants.ts", "TOKENS.ts"]
    }
  },
  "chains": [
    {
      "id": "ethereum",
      "name": "Ethereum",
      "almProxy": { "value": "0x1601843c5e9bc251a3272907010afa41fa18347e", "format": "evm" },
      "vaultProxy": { "value": "0x691a6c29e9e96dd897718305427ad5d534db16ba", "format": "evm" },
      "features": { "alm": true, "psm3": false, "lending": true },
      "notes": "Primary chain. Debt sourced from Vat."
    },
    {
      "id": "base",
      "name": "Base",
      "almProxy": { "value": "0x2917956eff0b5eaf030abdb4ef4296df775009ca", "format": "evm" },
      "psm3Contract": { "value": "0x1601843c5e9bc251a3272907010afa41fa18347e", "format": "evm" },
      "features": { "alm": true, "psm3": true, "lending": true }
    }
  ],
  "allocations": {
    "ethereum": [
      {
        "contract": { "value": "0x4dedf26112b3ec8ec46e7e31ea5e123490b05b8b", "format": "evm" },
        "protocol": "SparkLend",
        "type": "atoken",
        "underlying": "DAI",
        "notes": "spDAI"
      },
      {
        "contract": { "value": "0x6a9da2d710bb9b700acde7cb81f10f1ff8c89041", "format": "evm" },
        "protocol": "Blackrock",
        "type": "buidl",
        "underlying": "USDC",
        "notes": "BUIDL. Sky takes all profit."
      }
    ]
  }
}
```

### Data file: `shared/rates.denna-spec.json`

```json
{
  "$schema": "../schemas/shared-rates.schema.json",
  "metadata": {
    "id": "shared-rates",
    "name": "Shared Rates and Parameters",
    "kind": "io.denna.defi.rates",
    "version": "1.0.0",
    "lastUpdated": "2026-02-21",
    "source": {
      "repository": "primes-parameters",
      "references": ["src/lib/debt-pnl/constants.ts:17-22"]
    }
  },
  "rates": {
    "ssrSpread": { "value": 30, "unit": "bps" },
    "fallbackSsr": { "value": 4.5, "unit": "percent" },
    "baseRate": { "value": 4.66, "unit": "percent" }
  },
  "borrowRateSubsidy": {
    "start": "2026-01-01",
    "duration": { "value": 24, "unit": "months" },
    "cap": { "value": 1000000000, "currency": "USD" },
    "eligibleEntities": ["spark", "grove"]
  }
}
```

### GitHub Action: `.github/workflows/validate.yml`

```yaml
name: Validate Denna Spec Files

on:
  pull_request:
    paths: ['**/*.denna-spec.json', 'schemas/**']

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: daocraft/denna-spec@v1
        with:
          patterns: "**/*.denna-spec.json"
          exclude: "_template/**"
```

When someone opens a PR to change a wallet address, adjust a rate, or add a new allocation, the GitHub Action validates every changed file against its schema. Invalid addresses, missing required fields, or malformed values are caught automatically before the change is merged.

## FAQ

### What is the relationship between Denna and JSON Schema?

Denna Spec is built on top of JSON Schema. JSON Schema handles all structural validation — types, required fields, patterns, enums, nested objects. Denna Spec adds conventions JSON Schema cannot express: file naming, a standard metadata block, semantic value types, and an ecosystem for interoperability. Think of Denna Spec as the convention layer, and JSON Schema as the enforcement mechanism.

### Can I use Denna for documents, not just parameters?

Yes. Denna Spec is not limited to configuration parameters. It is designed for any structured data: governance documents, constitutional articles, asset registries, financial reports. Choose field names that match your data — `sections` and `content` for documents, `rates` for financial parameters, `chains` and `allocations` for protocol configuration.

### Do I need to use official Denna schemas?

No. Official Denna schemas are a convenience. You can define your own domain schemas, host them anywhere accessible by URL, and use them in your `.denna-spec.json` files. The only requirement is that your files have a `$schema` field that resolves to a valid JSON Schema.

### Can I use Denna for non-blockchain data?

Yes. While Denna's shared type definitions include blockchain-specific types like `address` and `chain`, these are optional. Any structured data that benefits from validation, versioning, and auditability is a good candidate for Denna.

### How do I propose changes to parameters in a Denna-powered repository?

Edit the relevant `.denna-spec.json` file, open a pull request. If the repository has a Denna validation action, the PR is automatically checked against the schema. Invalid changes are rejected before merge. Valid changes are reviewed, approved, and merged — with a permanent, public record of who changed what and why.

### Is `*.denna-spec.json` the only valid extension?

Yes. The `.denna-spec.json` extension is how tooling discovers Denna files. Using a different extension means validators and CI pipelines will not recognize the file.

### What is the relationship between Denna and AT Protocol?

Both follow the same design principle: standardize how data is described, and any tool built on the standard gains interoperability for free. AT Protocol defines a standard for social data (posts, profiles, follows) that any social app can read. Denna defines a standard for protocol data (parameters, addresses, governance docs) that any governance or analytics tool can read. The key difference is the storage layer: AT Protocol uses specialized PDS servers; Denna uses Git repositories.

## About

The Denna Specification was created to bring structure, auditability, and interoperability to protocol parameter management. It is designed to be minimal, generic, and built entirely on existing standards.

## License

Creative Commons — CC BY 4.0
