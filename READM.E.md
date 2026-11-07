# 📊 giniscope

**See who really holds a Soroban token.**

Measure token concentration with the Gini coefficient, top-holder shares and more, from a single Go binary.

[![Go](https://img.shields.io/badge/Go-1.22%2B-00ADD8?logo=go&logoColor=white)](https://go.dev)
[![Stellar](https://img.shields.io/badge/Stellar-Soroban-7D00FF?logo=stellar&logoColor=white)](https://soroban.stellar.org)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-early%20development-orange)](#-roadmap)

[Why](#-why-giniscope) · [Metrics](#-metrics) · [Quick start](#-quick-start) · [Data sources](#-data-sources) · [Roadmap](#-roadmap)

---

> **📝 Note:** giniscope is in **early development**. Sections marked *(planned)* describe the intended design and are not implemented yet. The [roadmap](#-roadmap) shows what exists today.

## ✨ Why giniscope

When a new token launches on Soroban, a raw holder list tells you very little. Treasuries, issuers, liquidity pools and other contracts can dominate the balances, so headline numbers look worse (or better) than reality.

giniscope turns a list of balances into a small set of standard distribution metrics, and lets you decide **which addresses count as real holders**.

- 🎯 **Honest numbers**: exclude issuers, treasuries, pools and dust, and see results with and without the filter
- 🔢 **Exact math**: balances use `math/big`, so 7-decimal token amounts never overflow
- 🧩 **Simple to run**: one binary, with CSV input that works offline
- 🔍 **Transparent**: every metric has a documented formula, and edge cases return errors instead of guesses

## 📐 Metrics

| Metric | What it tells you |
| :-- | :-- |
| **Gini coefficient** | `0.0` = every holder has the same balance, near `1.0` = one holder has almost everything |
| **Top-N share** | Percentage of supply held by the top 10, 50 or 100 holders |
| **Nakamoto coefficient** | Fewest holders whose combined balance exceeds 50% of supply |
| **Holder count** | Addresses with a non-zero balance after filtering |
| **Lorenz curve** | Cumulative supply share against cumulative holder share *(planned JSON/CSV export)* |

### How the Gini coefficient is computed

Sort balances ascending, `x₁ ≤ x₂ ≤ … ≤ xₙ`, then:

```
        Σ (2i − n − 1) · xᵢ
G  =  ─────────────────────
            n · Σ xᵢ
```

This equals the common form `(2·Σ i·xᵢ) / (n·Σ xᵢ) − (n+1)/n`. The sort makes it O(n log n).

**Edge cases**

- Fewer than two holders → error
- Total balance of zero → error
- Negative balance → error

### Reading a Gini value (rough guide)

| Gini | Rough reading |
| :-: | :-- |
| 0.0 – 0.4 | Broadly spread |
| 0.4 – 0.6 | Moderately concentrated |
| 0.6 – 0.8 | Highly concentrated |
| 0.8 – 1.0 | Dominated by a few addresses |

These bands are a convenience, not a verdict. A young token, or one with a large staking contract, can look concentrated for ordinary reasons.

## 🚀 Quick start

**Requires** Go 1.22 or newer.

```bash
git clone https://github.com/<owner>/giniscope
cd giniscope
go build -o giniscope ./cmd/giniscope
```

### Analyze a balance file

The simplest mode needs no network. Provide a CSV with `address,balance` columns, with balances as integers in the token's smallest unit:

```bash
./giniscope analyze --input balances.csv --decimals 7 --exclude exclude.txt
```

### Fetch holders from the network *(planned)*

```bash
./giniscope fetch --token <C...contract-address> --rpc <stellar-rpc-url> > balances.csv
```

Rebuilding holders from chain is the hardest part of the project. See [Data sources](#-data-sources).

### Sample output

```json
{
  "token_address": "C...",
  "total_supply": "10000000.0000000",
  "holders": 1420,
  "gini": 0.6821,
  "top_holders": {
    "top_10_pct": 54.21,
    "top_50_pct": 78.90
  },
  "nakamoto_coefficient": 4,
  "excluded_addresses": 6
}
```

*Values are illustrative.*

## 🧹 Excluding addresses

Gini is very sensitive to a few large non-user addresses. giniscope can exclude:

| Exclude | Why |
| :-- | :-- |
| Issuer or admin | Often holds the unissued supply |
| Treasury and team wallets | Listed explicitly by you |
| Liquidity pools and other contracts | Custody, not individual holders |
| Dust below a minimum balance | Removes noise from thousands of tiny wallets |

Reports show results **with and without** exclusions, so you can see how much they matter.

## 🔌 Data sources

Soroban contracts can't list their own holders, so the holder set has to be rebuilt from ledger data.

```
Ledger data  →  Rebuild balances  →  Filter addresses  →  Compute metrics  →  CLI / JSON report
```

| Source | Good for | Limit |
| :-- | :-- | :-- |
| **Stellar RPC** | Live tracking, recent history | Retains only a short window of ledgers and events |
| **Archive or ledger-metadata datasets** | Full history from token launch | More setup, and may have its own access terms |
| **Indexer APIs** | Quick start | Rate limits and trust in a third party |

Balances can be rebuilt by replaying `transfer`, `mint`, `burn` and `clawback` events, or by reading the token's balance ledger entries directly. Reading entries is more reliable because it doesn't depend on every event type being handled.

> **💡 Tip:** Check your provider's retention limit before choosing a start ledger.

## 🗂 Project layout *(planned)*

```
giniscope/
├── cmd/giniscope/         CLI entry point
├── internal/
│   ├── calculator/        Gini, top-N, Nakamoto, Lorenz (pure functions)
│   ├── ingest/            Reads balances or events from Stellar RPC
│   └── report/            JSON and table output
└── testdata/              Distributions with known expected results
```

A database for historical snapshots is optional and out of scope for the first version.

## 🛣 Roadmap

- [ ] Gini calculator with table-driven tests (all-equal → 0, one holder → (n−1)/n, cross-checked against NumPy)
- [ ] CSV input and address filtering
- [ ] Top-N, Nakamoto coefficient and Lorenz curve output
- [ ] Holder fetching for a Soroban token
- [ ] HTTP endpoint
- [ ] Optional snapshots over time

## ⚠️ Limitations

- Gini describes concentration among **addresses**, not people. One person can control many addresses, and one address, such as an exchange, can represent many people.
- Results depend on which addresses you exclude.
- Only SEP-41 tokens are in scope. Classic Stellar assets are not covered unless wrapped as a Soroban token.
- giniscope reports statistics. It is not investment advice.

## 🤝 Contributing

Issues and pull requests are welcome. Please add tests for any change to the calculations.

## 📄 License

[Apache 2.0](LICENSE)
