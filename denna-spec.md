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
| `kind` | string | REQUIRED | A reverse-domain identifier classifying the type of data (e.g., `io.denna.defi.protocol-config`, `io.sky.prime.config`). See [Section 3](#3-kind-identifiers). |
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
io.denna.defi.protocol-config
^^ ^^^^^ ^^^^ ^^^^^^^^^^^^^
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
io.denna.defi.protocol-config    → official DeFi protocol configuration schema
io.denna.defi.rates              → official DeFi rates schema
io.denna.governance.document     → official governance document schema
io.sky.prime.config              → Sky Protocol custom prime config
com.spark.pnl.params             → Spark custom PnL parameters
```

Organizations SHOULD use a namespace derived from a domain they control (reversed). Custom schemas can reference official Denna types via `$ref`.

### 4. Semantic Value Types

When representing the following common value types inside data fields, implementations SHOULD use these conventions. Denna provides reusable JSON Schema definitions at `https://spec.denna.io/v1/denna-types.schema.json`.

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
2. SHOULD import shared type definitions from `https://spec.denna.io/v1/denna-types.schema.json` via `$ref`.
3. SHOULD use JSON Schema draft 2020-12 or later.
4. MAY be stored in the same repository as the data files or published separately.
5. SHOULD be versioned at stable paths (e.g., `v1/`, `v2/`).

Example of a domain schema importing Denna types:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://spec.denna.io/v1/defi/protocol-config.schema.json",
  "type": "object",
  "required": ["$schema", "metadata", "chains"],
  "properties": {
    "$schema": { "type": "string" },
    "metadata": {
      "$ref": "https://spec.denna.io/v1/denna-types.schema.json#/$defs/metadata"
    },
    "chains": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id": {
            "$ref": "https://spec.denna.io/v1/denna-types.schema.json#/$defs/chain"
          },
          "almProxy": {
            "$ref": "https://spec.denna.io/v1/denna-types.schema.json#/$defs/address"
          }
        }
      }
    }
  }
}
```

### 6. Schema Ecosystem

Denna Spec ships with a canonical set of kind definitions for web3/DeFi and governance use cases. These are the official schemas tools can rely on for semantic interoperability.

#### 6.1. Canonical Kind Definitions

All canonical schemas are published at `https://spec.denna.io/v1/` and use the `io.denna.*` namespace.

##### `io.denna.defi.address-registry`

A registry of named, multi-chain contract or token addresses. Suitable for tracking token deployments, shared contracts, or protocol infrastructure addresses across multiple blockchains.

Schema: `https://spec.denna.io/v1/defi/address-registry.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `addresses` | object | REQUIRED | Named address groups. Each key is a logical name (e.g., USDS, USDC). Each value has `entries[]` with per-chain address objects. |
| `addresses.{name}.description` | string | OPTIONAL | What this address group represents. |
| `addresses.{name}.entries[].chain` | chain | REQUIRED | Chain identifier. |
| `addresses.{name}.entries[].address` | address | REQUIRED | The address on this chain. |
| `addresses.{name}.entries[].notes` | string | OPTIONAL | Chain-specific notes. |

##### `io.denna.defi.rates`

Shared rate parameters, fallback values, and subsidy programs for a DeFi protocol. Suitable for cross-entity rate constants that multiple entities reference.

Schema: `https://spec.denna.io/v1/defi/rates.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rates` | object | OPTIONAL | Named rate values (each a `rate` type with `value` + `unit`). |
| `fallbackRates` | array | OPTIONAL | Fallback rate values used when live data is unavailable. Each has `name`, `rate`, optional `description` and `source`. |
| `rateHierarchy` | array | OPTIONAL | Documented relationships between rates. Each has `name`, optional `formula`, `value`, `reference`. |
| `subsidyPrograms` | array | OPTIONAL | Time-bounded subsidy programs. Each has `name`, `start`, optional `duration`, `cap`, `capPeriod`, `eligibleEntities`, `formula`. |
| `timeConstants` | object | OPTIONAL | Computational time constants (`secondsPerYear`, `secondsPerDay`). |
| `externalDataSources` | array | OPTIONAL | External APIs or feeds used for rate data. Each has `name`, optional `url`, `field`, `lookbackDays`, `description`. |

##### `io.denna.defi.protocol-config`

Configuration for a DeFi protocol entity. Defines the chains it operates on, wallet/proxy addresses per chain, and asset allocation positions.

Schema: `https://spec.denna.io/v1/defi/protocol-config.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `chains` | array | OPTIONAL | Chains the entity operates on. Each entry has `id` (chain), optional `name`, `almProxy`, `vaultProxy`, `psm3Contract`, `susdsOracle`, `features`, `notes`. |
| `debtSource` | object | OPTIONAL | Where on-chain debt data is sourced from. Has `chain`, `vatAddress`, optional `notes`. |
| `allocations` | object | OPTIONAL | Asset allocation positions grouped by chain identifier. Each value is an array of allocation objects with `contract`, `protocol`, `type` (required), and optional `symbol`, `underlying`, `tags`, `notes`, etc. |

##### `io.denna.defi.pnl-config`

PnL configuration for a DeFi protocol entity. Covers calculation modules, subsidy programs, address classifications, asset caps, lending protocol configs, LP pool handling, stability module composition, direct exposures, and pricing config.

Schema: `https://spec.denna.io/v1/defi/pnl-config.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `parameters.calculationModules` | array | REQUIRED | PnL calculation modules. Each has `id`, `name`, `enabled`, optional `notes`. |
| `parameters.chains` | array | OPTIONAL | Active chains with proxy addresses and feature flags. |
| `parameters.debtSource` | object | OPTIONAL | Debt source config with `chain` and `vatAddress`. |
| `parameters.allocations` | object | OPTIONAL | Allocation positions keyed by chain ID. |
| `parameters.subsidyPrograms` | array | OPTIONAL | Subsidy programs this entity participates in. Each has `eligible`, optional `name`, `cap`, `capPeriod`, `start`, `duration`. |
| `parameters.assetCaps` | array | OPTIONAL | Per-asset allocation caps. Each has `asset`, `cap`, optional `effect`. |
| `parameters.lpPoolHandling` | array | OPTIONAL | Special handling rules for LP pools. Each has `lpAddress`, `name`, optional `protocol`, `activeFrom`, `treatment`. |
| `parameters.oneTimeAdjustments` | array | OPTIONAL | One-time revenue adjustments. Each has `month`, `address`, `amount`, `description`. |
| `parameters.prepayments` | array | OPTIONAL | Prepayment amounts. Each has `asset`, `address`, `amount`. |
| `parameters.addressClassifications` | object | OPTIONAL | Special address classifications affecting PnL treatment (`billAlways`, `skyTakesAll`, `simplePeriodReturn`, `directRate`, `ssrFunded`, `usdsSsr`). |
| `parameters.lendingProtocols` | array | OPTIONAL | Lending protocol configs. Each has optional `protocol`, `poolAddress`, `underlyingMappings`. |
| `parameters.pricingConfig` | object | OPTIONAL | Pricing sources, Centrifuge token mappings, and period buffers. |
| `parameters.stabilityModules` | array | OPTIONAL | Stability module (PSM-style) token composition configs. Each has optional `address`, `chain`, `tokens`. |
| `parameters.directExposures` | array | OPTIONAL | Direct exposure positions within stability modules. Each has `chain`, `moduleAddress`, `assetAddress`, optional `notes`. |
| `parameters.accountingNotes` | string | OPTIONAL | General accounting methodology notes including settlement formulas. |
| `parameters.knownIssues` | array | OPTIONAL | Documented known issues. Each has `id`, `description`, `status`, optional `source`. |

##### `io.denna.defi.vault-config`

Frontend-facing configuration for a single DeFi vault. Covers vault identity, share and deposit tokens, deposit/withdraw limits, attribution metadata, sub-markets or allocation buckets, fees, and compliance gating (terms of service, geo-blocking). Designed for frontend consumption — the shape captures what the UI needs to render, validate, and gate vault interactions without hardcoding addresses or business rules.

Schema: `https://spec.denna.io/v1/defi/vault-config.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `vault` | object | REQUIRED | Vault identity. Has `address`, `chain`, optional `protocol` (e.g. `kamino`, `morpho`, `yearn`), `notes`. |
| `shareToken` | object | REQUIRED | Vault receipt token. Has `address`, `symbol`, optional `decimals`, `name`. |
| `depositToken` | object | REQUIRED | Underlying asset accepted for deposit. Has `address`, `symbol`, optional `decimals`, `name`. |
| `limits` | object | OPTIONAL | Frontend-enforced deposit/withdraw limits. Has optional `minDeposit`, `minWithdraw`, `maxDeposit` (each a denna `amount`). |
| `attribution` | object | OPTIONAL | Referral/distribution tagging. Has optional `referralCode`, `memoProgram` (address, Solana), `notes`. |
| `markets` | array | OPTIONAL | Sub-markets or allocation buckets. Each has `id` (kebab-case), `name`, optional `description`, `collateralTypes[]`, `address`. |
| `fees` | object | OPTIONAL | Vault-level fees. Has optional `current`, `performance` (each a denna `rate`), `notes`. |
| `compliance` | object | OPTIONAL | Frontend gating. Has optional `termsOfService` (`version`, `required`, optional `url`) and `geoBlocking` (`enabled`, optional `blockedCountries[]` / `allowedCountries[]` as ISO 3166-1 alpha-2, `notes`). |

##### `io.denna.governance.document`

A governance or constitutional document. Suitable for protocol constitutions, governance articles, policy documents, and proposals.

Schema: `https://spec.denna.io/v1/governance/document.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | REQUIRED | Full title of the document. |
| `sections` | array | REQUIRED | Ordered sections. Each has `id`, `title`, optional `content`, `clauses[]`, `subsections[]`. |
| `effectiveDate` | date | OPTIONAL | Date this document became or becomes effective. |
| `status` | string | OPTIONAL | Document lifecycle status: `draft`, `active`, `superseded`, `deprecated`. |
| `supersedes` | string | OPTIONAL | ID of the document this one supersedes. |
| `definitions` | array | OPTIONAL | Defined terms. Each has `term` and `definition`. |

##### `io.denna.governance.registry`

A registry of governance participants. Suitable for tracking delegates, committee members, integrators, legal counsels, and other recognized roles.

Schema: `https://spec.denna.io/v1/governance/registry.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `registryType` | string | REQUIRED | Category of participants (e.g., `aligned-delegates`, `committee-members`, `integrators`). |
| `entries` | array | REQUIRED | List of participants. |
| `entries[].id` | string | REQUIRED | Unique kebab-case identifier. |
| `entries[].name` | string | REQUIRED | Display name. |
| `entries[].status` | string | REQUIRED | Current standing: `active`, `inactive`, or `derecognized`. |
| `entries[].aliases` | array | OPTIONAL | Known aliases or alternative names. |
| `entries[].role` | string | OPTIONAL | Specific role within the registry. |
| `entries[].address` | address | OPTIONAL | Primary wallet address. |
| `entries[].joinedDate` | date | OPTIONAL | Date the participant was recognized. |
| `entries[].exitDate` | date | OPTIONAL | Date the participant exited or was derecognized. |
| `entries[].references` | array | OPTIONAL | Governance post URLs or document references. |
| `entries[].notes` | string | OPTIONAL | Additional notes. |

##### `io.denna.governance.payment-record`

A record of governance-related payments. Suitable for tracking distribution rewards, integration boosts, governance rewards, and other protocol disbursements.

Schema: `https://spec.denna.io/v1/governance/payment-record.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `category` | string | REQUIRED | Type of payments (e.g., `distribution-reward`, `integration-boost`, `governance-reward`). |
| `payments` | array | REQUIRED | List of payment entries. |
| `payments[].period` | string | REQUIRED | Coverage period (YYYY-MM or ISO date range). |
| `payments[].recipient` | string | REQUIRED | Name or identifier of the recipient. |
| `payments[].amount` | amount | REQUIRED | Payment amount with currency. |
| `payments[].status` | string | REQUIRED | Payment status: `pending`, `paid`, or `cancelled`. |
| `payments[].recipientAddress` | address | OPTIONAL | Wallet address of the recipient. |
| `payments[].chain` | chain | OPTIONAL | Chain on which the payment was made. |
| `payments[].txHash` | string | OPTIONAL | On-chain transaction hash. |
| `payments[].notes` | string | OPTIONAL | Additional notes. |

##### `io.denna.governance.incident-log`

A log of governance incidents. Suitable for tracking security incidents, breach registries, dispute resolutions, and other significant events requiring formal documentation.

Schema: `https://spec.denna.io/v1/governance/incident-log.schema.json`

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `incidents` | array | REQUIRED | List of documented incidents. |
| `incidents[].id` | string | REQUIRED | Unique kebab-case identifier. |
| `incidents[].date` | date | REQUIRED | Date the incident occurred or was reported. |
| `incidents[].type` | string | REQUIRED | Category: `security`, `breach`, `dispute`, or `other`. |
| `incidents[].parties` | array | REQUIRED | Names or identifiers of involved parties. |
| `incidents[].description` | string | REQUIRED | Factual description of what occurred. |
| `incidents[].finding` | string | REQUIRED | Current status: `resolved`, `ongoing`, `dismissed`, or `pending`. |
| `incidents[].resolution` | string | OPTIONAL | Description of how the incident was resolved. |
| `incidents[].references` | array | OPTIONAL | Governance post URLs or document references. |
| `incidents[].notes` | string | OPTIONAL | Additional notes or context. |

#### 6.2. Adding New Kinds

The canonical kinds above cover the core web3/DeFi and governance use cases. New kinds should be proposed via the spec contribution process rather than created ad-hoc. When proposing a new kind:

- Open an issue or PR describing the use case
- The kind should be generic enough to serve multiple organizations
- Field names should avoid organization-specific terminology
- The kind should not duplicate an existing canonical kind

Organizations with internal-only needs SHOULD use a namespace derived from a domain they control (e.g., `io.sky.*`, `com.spark.*`) rather than the `io.denna.*` namespace.

#### 6.3. Local Schemas

During development or for private use cases, schemas MAY be stored locally in the same repository as the data files and referenced with relative paths:

```json
{ "$schema": "../../schemas/protocol-config.schema.json" }
```

### 7. File Organization

Denna does not mandate a directory structure, but RECOMMENDS the following:

1. Group data files by entity — one directory per protocol, prime, or logical unit.
2. Place shared or global parameters in a `shared/` directory.
3. Place domain schemas in a `schemas/` directory at the repository root.
4. Use descriptive file names that reflect content: `config.denna-spec.json`, `rates.denna-spec.json`, `allocations.denna-spec.json`.

#### 7.1. Repository Manifest

A parameter repository MAY include a **repository manifest** at its root: `denna-repo.denna-spec.json`. The manifest is a Denna file with kind `io.denna.repository` that declares the repo's version, release strategy, and an explicit list of all data files.

**Purpose:**

- **Discovery:** Consumers read `entries` to know every data file without scanning the filesystem.
- **Versioning:** `metadata.version` reflects the latest semantic version of the repository.
- **Consistency:** CI tooling can verify that `entries` matches the actual files on disk.

**Schema:** `https://spec.denna.io/v1/repository.schema.json`

**Required fields beyond the standard metadata block:**

| Field | Type | Constraint |
|-------|------|------------|
| `metadata.id` | string | Kebab-case repository identifier |
| `metadata.name` | string | Human-readable name |
| `metadata.version` | string | Strict semver: `^\d+\.\d+\.\d+$` |
| `metadata.kind` | string | Must be `"io.denna.repository"` |
| `repository.release.strategy` | string | Enum: `"semver"` |
| `repository.release.convention` | string | Enum: `"conventional-commits"` |
| `repository.entries` | array | Relative paths ending in `.denna-spec.json`, min 1, unique |

Entries use forward-slash separators and are relative to the repository root. Order is not significant.

**Example:**

```json
{
  "$schema": "https://spec.denna.io/v1/repository.schema.json",
  "metadata": {
    "id": "sky-parameters",
    "kind": "io.denna.repository",
    "name": "Sky Parameters",
    "version": "1.2.0"
  },
  "repository": {
    "release": {
      "strategy": "semver",
      "convention": "conventional-commits"
    },
    "entries": [
      "grove/pnl-config.denna-spec.json",
      "spark/protocol-config.denna-spec.json"
    ]
  }
}
```

### 8. Versioning

1. The Denna Specification itself follows [Semantic Versioning](https://semver.org/). The current version is **1.0.0**.
2. Individual data files SHOULD include a `metadata.version` field to track parameter changes over time.
3. Domain schemas SHOULD be versioned at stable paths (`v1/`, `v2/`). A new folder is introduced only on breaking changes — removing fields, narrowing types, or making optional fields required.
4. Within a schema folder (e.g., `v1/`), changes are additive only: new kinds may be added and new optional fields may be introduced. The canonical URL at `spec.denna.io/v1/` always serves the latest published v1 schemas.

#### 8.1. Spec Version and Tool Compatibility

The Denna Specification is versioned via semantic release. Each release is tagged (e.g., `v1.2.3`) and corresponds to a known set of canonical kinds and fields. Tool builders SHOULD document which spec version their tool supports.

**Schema URL as version declaration.** The `$schema` field in a Denna file implicitly declares the schema version through its URL:

| `$schema` URL | Meaning |
|---------------|---------|
| `https://spec.denna.io/v1/defi/rates.schema.json` | Latest published v1 — tracks new additions |
| `https://raw.githubusercontent.com/daocraft/denna-spec/v1.2.3/v1/defi/rates.schema.json` | Pinned to spec release v1.2.3 |

Tools that need strict compatibility SHOULD resolve `$schema` against a pinned release tag rather than the live URL. Tools that track the latest SHOULD handle unknown fields and kinds gracefully.

### 9. Validation

1. A Denna file is valid if and only if it passes JSON Schema validation against the schema referenced in its `$schema` field.
2. Validators MUST resolve the `$schema` field and validate the entire file against the resolved schema.
3. If the `$schema` field is missing or the referenced schema cannot be resolved, the file MUST be treated as invalid.

### 10. API Response Profile

The `$schema` + `metadata.kind` envelope is equally useful in HTTP API responses. The API Response Profile is a lightweight variant that omits file-centric metadata fields.

#### 10.1. Envelope

An API response MUST include `metadata.kind`. The `$schema` field SHOULD be included so consumers and tooling can validate the response. All other metadata fields are OPTIONAL and SHOULD be omitted unless meaningful in the API context.

```json
{
  "$schema": "https://spec.denna.io/v1/defi/rates.schema.json",
  "metadata": {
    "kind": "io.denna.defi.rates"
  },
  "rates": {
    "ssrSpread": { "value": 30, "unit": "bps" },
    "fallbackSsr": { "value": 4.5, "unit": "percent" }
  }
}
```

#### 10.2. Metadata Fields

| Field | File Profile | API Profile |
|-------|-------------|-------------|
| `kind` | REQUIRED | REQUIRED |
| `id` | REQUIRED | OPTIONAL — use if the response represents a named resource |
| `name` | REQUIRED | OPTIONAL |
| `version` | OPTIONAL | OPTIONAL — useful for cache invalidation |
| `description` | OPTIONAL | OPTIONAL |
| `lastUpdated` | OPTIONAL | OMIT |
| `tags` | OPTIONAL | OMIT |
| `source` | OPTIONAL | OMIT |

#### 10.3. Compatibility

Domain content fields (`rates`, `addresses`, `chains`, etc.) are identical between profiles. Any schema or tool that processes a Denna file of a given `kind` can process an API response of the same `kind` without modification. The canonical schemas at `spec.denna.io/v1/` validate both.

## Complete Example

This example shows a DeFi protocol managing chain configuration and rate parameters using Denna.

### Repository structure

```
primes-parameters/
├── schemas/
│   ├── protocol-config.schema.json     # Defines structure for protocol config files
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
  "$schema": "../schemas/protocol-config.schema.json",
  "metadata": {
    "id": "spark-config",
    "name": "Spark",
    "kind": "io.denna.defi.protocol-config",
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
