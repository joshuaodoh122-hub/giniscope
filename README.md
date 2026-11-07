# 📊 giniscope

**See who really holds a Soroban token.**

Measure token concentration with the Gini coefficient, top-holder shares and the Nakamoto coefficient, from a single Go binary.

[![Go](https://img.shields.io/badge/Go-1.22%2B-00ADD8?logo=go&logoColor=white)](https://go.dev)
[![Stellar](https://img.shields.io/badge/Stellar-Soroban-7D00FF?logo=stellar&logoColor=white)](https://soroban.stellar.org)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-early%20development-orange)](#-status)

[Why](#-why-giniscope) · [Metrics](#-metrics) · [Quick start](#-quick-start) · [Excluding addresses](#-excluding-addresses) · [Data sources](#-data-sources) · [Status](#-status)

---

> **📝 Note:** giniscope is in **early development**. Features marked *(planned)* describe the intended design and are not implemented yet. The [status table](#-status) shows what exists today.

## ✨ Why giniscope

When a new token launches on Soroban, a raw holder list tells you very little. Treasuries, issuers, liquidity pools and other contracts can dominate the balances, so headline numbers look worse (or better) than reality.

giniscope turns a list of balances into a small set of standard distribution metrics, and lets you decide **which addresses count as real holders**.

- 🎯 **Honest numbers**: exclude issuers, treasuries, pools and dust, and see results with and without the filter
- 🔢 **Exact math**: balances use `math/big`, so 7-decimal token amounts never overflow or lose precision
- 🧩 **Simple to run**: one binary, with CSV input that works fully offline
- 🔍 **Transparent**: every metric has a documented formula, and edge cases return errors instead of guesses

## 📐 Metrics

| Metric | What it tells you |
| :-- | :-- |
| **Gini coefficient** | `0.0` = every holder has the same balance, near `1.0` = one holder has almost everything |
| **Top-N share** | Percentage of counted supply held by the top 10, 50 or 100 holders |
| **Nakamoto coefficient** | Fewest holders whose combined balance is **strictly more than** 50% of counted supply |
| **Holder count** | Addresses with a non-zero balance after filtering |
| **Lorenz curve** | Cumulative supply share against cumulative holder share *(planned JSON/CSV export)* |

All percentages are relative to the **counted supply**: the sum of balances that remain after exclusions. They are not relative to the token's total supply unless nothing is excluded.

### How the Gini coefficient is computed

Only holders with a balance greater than zero are counted. Sort their balances ascending, `x₁ ≤ x₂ ≤ … ≤ xₙ`, then:

```
        Σ (2i − n − 1) · xᵢ
G  =  ─────────────────────
            n · Σ xᵢ
```

This equals the common form `(2·Σ i·xᵢ) / (n·Σ xᵢ) − (n+1)/n`. The sort makes it O(n log n). Everything stays in exact integer arithmetic until the final division, which is done once with `big.Rat` and converted to a float for display.

This is the standard population form. It is not the small-sample correction `n/(n−1)`, so a maximally unequal distribution approaches `(n−1)/n` rather than exactly `1.0`.

**Worked examples** (also used as tests)

| Balances | n | Gini |
| :-- | :-: | :-: |
| `1, 1, 1, 1` | 4 | `0.0` |
| `1, 3` | 2 | `0.25` |
| `1, 1, 1, 97` | 4 | `0.72` |

**Edge cases return errors**

- Fewer than two holders with a non-zero balance
- Total balance of zero
- A negative balance
- A balance that is not a valid integer

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

The simplest mode needs no network. Provide a CSV with an `address,balance` header, with balances as integers in the token's smallest unit (for a 7-decimal token, `1.5` tokens is `15000000`):

```csv
address,balance
GAAA...,15000000
CBBB...,250000000
GCCC...,100
```

```bash
./giniscope analyze \
  --input balances.csv \
  --decimals 7 \
  --exclude exclude.txt \
  --min-balance 1000000
```

`exclude.txt` holds one address per line. Lines starting with `#` are comments.

| Flag | Purpose |
| :-- | :-- |
| `--input` | Path to the balances CSV |
| `--decimals` | Token decimals, used only to display amounts |
| `--exclude` | File of addresses to leave out |
| `--min-balance` | Drop holders below this amount, in smallest units |
| `--exclude-contracts` | Drop every `C...` address *(planned)* |
| `--format` | `json` (default) or `table` *(planned)* |

### Fetch holders from the network *(planned)*

```bash
./giniscope fetch --token <C...contract-address> --rpc <stellar-rpc-url> > balances.csv
```

Rebuilding holders from chain is the hardest part of the project. See [Data sources](#-data-sources).

### Sample output

Reports include the metrics **with and without** your exclusions.

```json
{
  "token_address": "C...",
  "decimals": 7,
  "all_holders": {
    "total_supply": "10000000.0000000",
    "holders": 1426,
    "gini": 0.9134,
    "top_10_pct": 91.80,
    "nakamoto_coefficient": 1
  },
  "after_exclusions": {
    "counted_supply": "3200000.0000000",
    "holders": 1420,
    "gini": 0.6821,
    "top_10_pct": 54.21,
    "nakamoto_coefficient": 4
  },
  "excluded_addresses": 6
}
```

*Values are illustrative.*

## 🧹 Excluding addresses

Gini is very sensitive to a few large non-user addresses. giniscope can exclude:

| Exclude | How | Why |
| :-- | :-- | :-- |
| Issuer or admin | `--exclude` list | Often holds the unissued supply |
| Treasury and team wallets | `--exclude` list | Listed explicitly by you |
| Liquidity pools and other contracts | `--exclude` list, or `--exclude-contracts` *(planned)* | Custody, not individual holders |
| Dust below a minimum balance | `--min-balance` | Removes noise from thousands of tiny wallets |

Contract addresses (`C...`) cannot be reliably classified as pools, vaults or real users from the address alone, so excluding them is always an explicit choice and never the default.

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
│   ├── calculator/        Gini, top-N, Nakamoto, Lorenz (pure functions, no I/O)
│   ├── ingest/            Reads balances or events from Stellar RPC
│   └── report/            JSON and table output
└── testdata/              Distributions with known expected results
```

The calculator takes balances in and returns metrics out, with no knowledge of where the balances came from. That keeps the math testable and lets the data source change freely. A database for historical snapshots is optional and out of scope for the first version.

## 🛣 Status

| Area | State |
| :-- | :-: |
| Gini calculator with table-driven tests (all-equal → 0, worked examples above, cross-checked against NumPy) | ⬜ planned |
| CSV input, exclude file and `--min-balance` | ⬜ planned |
| Top-N, Nakamoto coefficient and Lorenz curve output | ⬜ planned |
| With/without exclusions comparison in reports | ⬜ planned |
| Holder fetching for a Soroban token | ⬜ planned |
| HTTP endpoint | ⬜ planned |
| Optional snapshots over time | ⬜ planned |

**v0.1 target:** the `analyze` command working end to end on CSV input, plus one worked example on a real Soroban token.

## ⚠️ Limitations

- Gini describes concentration among **addresses**, not people. One person can control many addresses, and one address, such as an exchange, can represent many people.
- Results depend on which addresses you exclude.
- Only SEP-41 tokens are in scope. Classic Stellar assets are not covered unless wrapped as a Soroban token.
- giniscope reports statistics. It is not investment advice.

## 🤝 Contributing

Issues and pull requests are welcome. Please add tests for any change to the calculations, and include a worked example when you change a formula or an edge case.

## 📄 License

[Apache 2.0](LICENSE)
