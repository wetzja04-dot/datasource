# CLAUDE.md

This file provides guidance for AI assistants working with this repository.

## Project Overview

This repository is a **blockchain data source** for the 0xtorch ecosystem. It contains structured JSON data files and TypeScript scripts for registering metadata about:

- **EVM (Ethereum-compatible) chains**: contract addresses, ABI definitions, transaction analyzers
- **Solana**: program analyzers
- **Apps**: DeFi protocol / exchange metadata with icons
- **Symbols**: ticker-to-ID mappings for cryptocurrency exchanges

Data is deployed to a private API server via GitHub Actions when changes are merged to `main`.

## Tech Stack

- **Runtime**: [Bun](https://bun.sh) (v1.0.25+) — used for scripts, tests, and file I/O
- **Language**: TypeScript (strict mode, ESNext, no emit)
- **Linter/Formatter**: [Biome](https://biomejs.dev) v1.9.4
- **Formatter for package.json**: Prettier with `prettier-plugin-packagejson`
- **Key dependencies**:
  - `@0xtorch/core` — core types and utilities (e.g., `stringify`)
  - `@0xtorch/evm` — EVM chain helpers, schemas (`evmAddressSchema`, `analyzerSchema`, chain factory functions)
  - `@0xtorch/solana` — Solana analyzer schemas (`solanaAnalyzersJsonSchema`)
  - `abitype` — Ethereum ABI type utilities (with Zod schemas via `abitype/zod`)
  - `viem` — Ethereum utilities (selectors, signatures)
  - `zod` — runtime schema validation

## Repository Structure

```
datasource/
├── apps/                        # DeFi app/protocol metadata
│   ├── *.json                   # One file per app (validated by appSchema)
│   └── icons/                   # App icon files (SVG/PNG)
├── evms/                        # EVM chain data
│   ├── analyzers/               # Transaction analyzers keyed by function selector
│   │   └── 0x{8hex}.json        # Array of analyzerSchema objects
│   ├── chains/                  # On-chain address metadata, keyed by chainId
│   │   └── {chainId}/
│   │       └── 0x{40hex}.json   # Per-address JSON (evmAddressWithoutChainIdSchema)
│   ├── events/                  # Event ABI definitions keyed by topic0
│   │   └── 0x{64hex}.json       # Array of AbiEvent objects
│   └── functions/               # Function ABI definitions keyed by selector
│       └── 0x{8hex}.json        # Array of AbiFunction objects
├── solanas/                     # Solana data
│   └── analyzers/               # Program analyzers keyed by program ID
│       └── {programId}.json     # Array of Solana analyzer objects
├── symbols/                     # Exchange ticker-to-currency-ID mappings
│   └── {exchange}.json          # Record<string, string> (ticker -> crypto ID)
├── tests/                       # Test data and test files
│   ├── apps.test.ts             # Validates all apps/*.json against appSchema
│   ├── evmAddresses.test.ts     # Validates all evms/chains/**/*.json
│   ├── solanas.test.ts          # Validates all solanas/analyzers/*.json
│   ├── crypto.json              # Cryptocurrency list for testing
│   └── evm/                     # Decoded EVM transaction test fixtures
│       └── {chainId}/
│           └── {txHash}.json
├── scripts/                     # TypeScript scripts run with Bun
│   ├── constants.ts             # Env vars: API_ENDPOINT, USERNAME, PASSWORD
│   ├── schemas.ts               # Zod schemas: appSchema, evmAddressWithoutChainIdSchema
│   ├── registerApps.ts          # POST apps/*.json to API
│   ├── registerAppIcons.ts      # PUT apps/icons/* to API
│   ├── registerEvmAddresses.ts  # POST evms/chains/**/*.json to API
│   ├── registerEvmAnalyzers.ts  # POST evms/analyzers/*.json to API
│   ├── registerEvmEvents.ts     # POST evms/events/*.json to API
│   ├── registerEvmFunctions.ts  # POST evms/functions/*.json to API
│   ├── registerSolanaAnalyzers.ts # POST solanas/analyzers/*.json to API
│   ├── registerSymbols.ts       # POST symbols/*.json to API
│   ├── createEvmAbiFiles.ts     # Extract ABIs from address JSONs into events/functions dirs
│   ├── createEvmAnalyzerByTransaction.ts # Generate analyzer JSON from a real tx
│   └── checkEvmAddressIsSpamByGemini.ts  # Use Gemini CLI to classify spam addresses
├── .github/workflows/
│   ├── pull_request.yml         # CI: runs `bun test` on PRs
│   └── deploy.yml               # CD: runs tests then registers changed files to API on main push
├── biome.json                   # Biome linter/formatter config
├── prettier.config.js           # Prettier config (for package.json only)
├── tsconfig.json                # TypeScript config
└── package.json                 # Bun project config
```

## Development Commands

```bash
# Install dependencies
bun install

# Run all tests
bun test

# Run linter and formatter (auto-fix)
bun run check   # runs: biome check --write

# Run a specific script manually
FILES=apps/uniswap.json bun run scripts/registerApps.ts

# Generate EVM analyzer from a transaction
CHAIN=1 HASH=0x... ADDRESS=0x... ACTION=trade \
  ETHEREUM_API_KEY=... API_ENDPOINT=... \
  bun run scripts/createEvmAnalyzerByTransaction.ts

# Extract ABI files from address JSON files
bun run scripts/createEvmAbiFiles.ts

# Check EVM addresses for spam using Gemini CLI (requires gemini CLI installed)
bun run scripts/checkEvmAddressIsSpamByGemini.ts
```

## Code Style & Conventions

Enforced by Biome (`biome.json`):

- **Indentation**: 2 spaces
- **Quotes**: single quotes (`'`) in JavaScript/TypeScript
- **Semicolons**: omitted (ASI style) — no trailing semicolons
- **Filenames**: `camelCase` or matching the export name (`filenameCases: ["camelCase", "export"]`)
- **Imports**: auto-organized by Biome
- **Line endings**: LF (enforced by Prettier for `package.json`)

Biome ignores `evms/**/*.json` and `tests/**/*.json` (these are data files, not source).

TypeScript is strict (`"strict": true`) with `verbatimModuleSyntax`. Use `import type` for type-only imports.

## Data File Schemas

### `apps/*.json` — App metadata

```jsonc
{
  "id": "uniswap",                              // unique slug
  "categories": ["dex"],                        // "bridge" | "cex" | "cross-chain" | "dex" | "gaming" | "lending" | "nft-marketplace" | "other"
  "name": "Uniswap",
  "description": "...",                         // optional
  "website": "https://uniswap.org",             // optional
  "icon": "apps/icons/uniswap.svg"             // optional, relative path
}
```

### `evms/chains/{chainId}/0x{address}.json` — EVM address metadata

```jsonc
{
  "address": "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2",
  "label": "WETH9",               // optional human-readable label
  "isSpam": false,                // optional boolean
  "app": "uniswap",               // optional app ID reference
  "abi": "[...]"                  // optional ABI JSON string (stringified JSON)
}
```

Note: `chainId` is derived from the directory name, not stored in the file.

### `evms/analyzers/0x{functionId}.json` — EVM transaction analyzers

Array of `analyzerSchema` objects from `@0xtorch/evm`. Each analyzer defines:
- `example`: reference transaction (chainId, hash, addresses)
- `condition`: matching rules (function selector, log patterns, value ranges, related address checks)
- `generators`: action generators (trade, swap, bridge, etc.) with transfer patterns

### `evms/events/0x{topic0}.json` — Event ABI definitions

Array of `AbiEvent` objects (from `abitype/zod`). File name is the keccak256 topic0 hash.

### `evms/functions/0x{selector}.json` — Function ABI definitions

Array of `AbiFunction` objects (from `abitype/zod`). File name is the 4-byte function selector.

### `solanas/analyzers/{programId}.json` — Solana program analyzers

Array of Solana analyzer objects validated by `solanaAnalyzersJsonSchema` from `@0xtorch/solana`. Each entry defines:
- `description`: human-readable description
- `example`: URL to example transaction
- `programId`: Solana program public key
- `instructions`: matching rules for instructions
- `transfers`: expected transfer patterns
- `generators`: action generators with transfer mappings

### `symbols/{exchange}.json` — Symbol mappings

`Record<string, string>` mapping exchange ticker symbols to 0xtorch currency IDs.

```json
{ "BTCUSD": "crypto/bitcoin", "ETHUSD": "crypto/ethereum" }
```

## CI/CD Pipeline

### Pull Requests (`pull_request.yml`)
Runs `bun test` to validate all JSON files against their schemas.

### Deploy to main (`deploy.yml`)
On push to `main`, the workflow:
1. Runs `bun test`
2. Detects which files changed (using `git diff` between before/after commits)
3. Registers only the changed files to the API server using the corresponding register script
4. Requires GitHub secrets: `API_ENDPOINT`, `USERNAME`, `PASSWORD`

### Environment Variables for Scripts

| Variable | Description |
|---|---|
| `API_ENDPOINT` | Base URL of the 0xtorch API server |
| `USERNAME` | Basic auth username |
| `PASSWORD` | Basic auth password |
| `FILES` | Comma-separated list of file paths to process |
| `CHAIN` | Chain ID (for `createEvmAnalyzerByTransaction.ts`) |
| `HASH` | Transaction hash (for `createEvmAnalyzerByTransaction.ts`) |
| `ADDRESS` | Comma-separated wallet addresses (for `createEvmAnalyzerByTransaction.ts`) |
| `ACTION` | Action type (for `createEvmAnalyzerByTransaction.ts`) |
| `ETHEREUM_API_KEY`, `BASE_API_KEY`, etc. | Etherscan-compatible API keys per chain |

## Adding New Data

### New App
1. Create `apps/{app-id}.json` matching `appSchema`
2. Add icon to `apps/icons/{app-id}.svg` (or `.png`)
3. Reference icon path in the JSON: `"icon": "apps/icons/{app-id}.svg"`

### New EVM Contract Address
1. Create `evms/chains/{chainId}/0x{address}.json` matching `evmAddressWithoutChainIdSchema`
2. Filename must be the checksummed or lowercase address with `.json` extension
3. If the contract has an ABI, include it as a stringified JSON in the `abi` field
4. Run `bun run scripts/createEvmAbiFiles.ts` to extract event/function ABIs

### New EVM Analyzer
- Use the automated script: set env vars and run `createEvmAnalyzerByTransaction.ts`
- Or manually create/edit `evms/analyzers/0x{functionId}.json` following the `analyzerSchema`

### New Solana Analyzer
Create or append to `solanas/analyzers/{programId}.json` following `solanaAnalyzersJsonSchema`.

### New Symbol Mapping
Create or update `symbols/{exchange}.json` with a `Record<string, string>` mapping.

## Supported EVM Chain IDs

Key chain IDs used across the codebase:

| Chain ID | Network |
|---|---|
| 1 | Ethereum Mainnet |
| 10 | Optimism |
| 56 | BNB Smart Chain |
| 137 | Polygon PoS |
| 324 | zkSync Era |
| 592 | Astar |
| 1284 | Moonbeam |
| 2020 | Ronin |
| 5000 | Mantle |
| 8453 | Base |
| 42161 | Arbitrum One |
| 43114 | Avalanche C-Chain |

## Testing

Tests are located in `tests/` and use Bun's built-in test runner.

```bash
bun test                        # run all tests
bun test tests/apps.test.ts     # run only app schema tests
```

Test files validate all data files against their Zod schemas. If a JSON file fails schema validation, the test will fail with details.

The `tests/evm/` directory contains decoded transaction fixtures (generated by `createEvmAnalyzerByTransaction.ts`) used as reference data — not directly tested by the test suite.
