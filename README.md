# Predicate-Based Data Sharing — Research Prototype

This repository is a fork of the thesis prototype by Bilal Ahmad and Arin Faraj, developed as part of their degree project with Fixmeapp.ai as host company.

The project explores predicate-based data sharing for privacy-preserving user data verification in centralized social platforms, using WibsonTree-inspired predicates, OBDD structures, and comparison with zero-knowledge proofs.

Shared and hosted by FIXMEAPP with permission from the original authors.

---

A benchmarking prototype comparing three data verification architectures:

| System | Name | Data Exposure | Mechanism |
|--------|------|---------------|-----------|
| **A** | Traditional (Raw) | Full | Client sends raw JSON; server evaluates predicate |
| **B** | Predicate-Based | Minimal | OBDD-based witness with XOR hash reconstruction |
| **C** | ZK | Zero | Groth16 proof via Circom circuits |

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   NOTARY     │     │   SELLER     │     │    BUYER     │
│  (Issuer)    │     │  (Client)    │     │  (Server)    │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ A: no-op     │     │ A: raw JSON  │────▶│ A: evaluate  │
│ B: root hash │────▶│ B: witness   │────▶│ B: verify    │
│ C: vkeys     │────▶│ C: proof     │────▶│ C: verify    │
└──────────────┘     └──────────────┘     └──────────────┘
```

**Predicate under test**: `age >= 18 AND location === 'SE'`

## Quick Start (Docker)

```bash
# Build and run the full benchmark
docker-compose up --build

# Results will appear in ./results/results.csv
```

## Quick Start (Local)

### Prerequisites
- Node.js 20+
- PostgreSQL 16+
- Circom 2.1.6+ (for ZKP, optional)

### Setup

```bash
# Install dependencies
npm install

# Start PostgreSQL (or use docker-compose up -d postgres)
# Ensure DATABASE_URL is set or use default: postgres://bench:bench123@localhost:5432/benchmark

# Generate synthetic data
npm run generate-data

# (Optional) Build Circom circuits for ZKP
npm run build-circuits

# Run the benchmark
npm run test-runner

# Or run without ZKP
ENABLE_ZKP=false npm run test-runner

# Export results to CSV
npm run export-csv
```

## Project Structure

```
├── circuits/                      # Circom ZK circuits
│   ├── age_check.circom           # age >= threshold
│   ├── location_check.circom      # location === target
│   ├── combined_check.circom      # age AND location
│   ├── build-circuits.sh          # Compilation + trusted setup
│   └── build/                     # Compiled artifacts (generated)
├── data/                          # Generated datasets (JSON)
├── results/                       # Benchmark output (CSV)
├── src/
│   ├── shared/
│   │   ├── types.ts               # All type definitions
│   │   ├── measurement.ts         # High-res timer utilities
│   │   ├── db.ts                  # PostgreSQL connection & CSV export
│   │   └── export-csv.ts          # Standalone export script
│   ├── data-generator/
│   │   └── generator.ts           # Seeded PRNG data generator
│   ├── systems/
│   │   ├── system-a/              # Traditional (raw JSON)
│   │   │   ├── notary.ts
│   │   │   ├── seller.ts
│   │   │   └── buyer.ts
│   │   ├── system-b/              # Predicate-Based (OBDD + XOR hash)
│   │   │   ├── obdd.ts
│   │   │   ├── notary.ts
│   │   │   ├── seller.ts
│   │   │   └── buyer.ts
│   │   └── system-c/              # ZK (Circom + snarkjs)
│   │       ├── notary.ts
│   │       ├── seller.ts
│   │       └── buyer.ts
│   └── benchmark/
│       ├── config.ts              # Benchmark parameters
│       └── test-runner.ts         # Main orchestration script
├── package.json
├── tsconfig.json
├── Dockerfile                     # Multi-stage (Circom + Node.js)
└── docker-compose.yml             # App + PostgreSQL
```

## Measurement Metrics

| Metric | Method | Unit |
|--------|--------|------|
| Total Latency | `process.hrtime.bigint()` | nanoseconds |
| CPU Time | `process.cpuUsage()` | milliseconds |
| Payload Size | `Buffer.byteLength(JSON.stringify(payload))` | bytes |

All measurements are stored in PostgreSQL and exported to `results/results.csv`.

## Predicate-Based Protocol

System B implements the OBDD-based predicate verification:

1. **Notary** builds an OBDD for `age >= 18`, computes hashes bottom-up:
   - Leaf: `H = SHA256(result_byte)`
   - Node: `H = SHA256( SHA256(H_low ∥ L_i) ∥ SHA256(H_high ∥ R_i) )`
   - Aux: `Aux = SHA256(H_low ∥ L_i) ⊕ SHA256(H_high ∥ R_i)`

2. **Seller** traverses the OBDD with the user's age bits, collecting path keys + Aux values

3. **Buyer** reconstructs the root hash bottom-up and compares with the Notary's commitment

## ZK Protocol

System C uses Circom + snarkjs for Groth16 proofs:

1. **Circuits**: `age_check` (≥), `location_check` (===), `combined_check` (both)
2. **Seller**: `snarkjs.groth16.fullProve(input, wasm, zkey)` → `{proof, publicSignals}`
3. **Buyer**: `snarkjs.groth16.verify(vkey, publicSignals, proof)` → `boolean`

## Configuration

| Parameter | Default | Environment Variable |
|-----------|---------|---------------------|
| Database URL | `postgres://bench:bench123@localhost:5432/benchmark` | `DATABASE_URL` |
| CSV output | `results/results.csv` | `CSV_OUTPUT` |
| Enable ZKP | `true` | `ENABLE_ZKP` |
| Sample sizes | `100, 1000, 10000` | CLI: `--sample-sizes 100,1000` |
| PRNG seed | `12345` | Hardcoded in config |

Research prototype — KTH Royal Institute of Technology (II142X)
