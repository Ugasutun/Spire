# Contributing to Spire

Thank you for your interest in contributing to Spire. This document covers everything you need to get involved. Spire handles real financial flows — invoice payments, LP capital, and debtor obligations — so the bar for contract changes is high. Please read the financial safety sections carefully before contributing to contract code.

---

## Table of contents

- [Code of conduct](#code-of-conduct)
- [Ways to contribute](#ways-to-contribute)
- [Getting started](#getting-started)
- [Project structure](#project-structure)
- [Development workflow](#development-workflow)
- [Writing Soroban contracts](#writing-soroban-contracts)
- [Working with the pricing model](#working-with-the-pricing-model)
- [Writing the SDK](#writing-the-sdk)
- [Testing](#testing)
- [Pull request process](#pull-request-process)
- [Issue guidelines](#issue-guidelines)
- [Security vulnerabilities](#security-vulnerabilities)
- [Community](#community)

---

## Code of conduct

Spire follows the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/) Code of Conduct. Report unacceptable behaviour via the repository's security policy email.

---

## Ways to contribute

**Code**
- Implement contract features from the roadmap
- Improve the risk scoring model
- Fix bugs and improve test coverage
- Optimise Soroban resource usage

**Research**
- Propose and model discount rate algorithms
- Research default rate data for invoice financing in target markets
- Analyse fraud vectors and propose mitigations
- Research anchor recourse mechanisms per jurisdiction

**Documentation**
- Improve SME onboarding guides
- Write LP yield expectation guides
- Document the attestation process for different invoice types

**Community**
- Answer questions in GitHub Discussions
- Review open pull requests
- Report bugs with clear reproduction steps

---

## Getting started

### Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Rust | `>=1.74` | Soroban contract development |
| `soroban-cli` | latest | Contract build, deploy, invoke |
| Node.js | `>=18` | Spire.js SDK and backend |
| pnpm | `>=8` | SDK package management |
| Docker | any | Local Stellar testnet |

### Install Rust and Soroban toolchain

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32-unknown-unknown
cargo install --locked soroban-cli
```

### Clone and build

```bash
git clone https://github.com/your-org/spire
cd spire
cargo build
```

### Start a local testnet

```bash
docker run --rm -it \
  -p 8000:8000 \
  stellar/quickstart:latest \
  --testnet \
  --enable-soroban-rpc
```

### Run the full test suite

```bash
cargo test
```

---

## Project structure

```
spire/
├── contracts/
│   ├── invoice-nft/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── storage.rs
│   │   │   ├── attestation.rs
│   │   │   ├── events.rs
│   │   │   ├── errors.rs
│   │   │   └── types.rs
│   │   └── Cargo.toml
│   ├── marketplace/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── listing.rs
│   │   │   ├── bidding.rs
│   │   │   └── settlement.rs
│   │   └── Cargo.toml
│   ├── liquidity-pool/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   ├── pool.rs
│   │   │   ├── scorer.rs
│   │   │   ├── yield.rs
│   │   │   └── reserve.rs
│   │   └── Cargo.toml
│   └── repayment/
│       ├── src/
│       │   ├── lib.rs
│       │   ├── routing.rs
│       │   └── default.rs
│       └── Cargo.toml
├── backend/
│   └── src/
├── frontend/
│   └── src/
├── sdk/
│   └── src/
├── docs/
├── CONTRIBUTING.md
├── README.md
└── LICENSE
```

---

## Development workflow

Spire uses a standard fork-and-branch workflow.

### 1. Fork and clone

```bash
git clone https://github.com/YOUR_USERNAME/spire
cd spire
git remote add upstream https://github.com/your-org/spire
```

### 2. Branch naming

```
feat/short-description        # new feature
fix/short-description         # bug fix
pricing/short-description     # pricing model change
docs/short-description        # documentation only
test/short-description        # tests only
refactor/short-description    # no behaviour change
```

### 3. Commit conventions

```
[invoice-nft] Add non-transferable flag enforcement
[marketplace] Fix bid expiry ledger validation
[pricing] Adjust credit spread curve for low-history debtors
[sdk] Implement repayInvoice with partial payment support
```

### 4. Sync with upstream

```bash
git fetch upstream
git rebase upstream/main
```

---

## Writing Soroban contracts

### Financial safety rules

Spire moves real money. These rules are non-negotiable for any contract contribution:

- Every function that transfers tokens must be atomic — either the full transfer succeeds or the entire transaction reverts. Never split a token transfer across multiple calls.
- Every arithmetic operation on token amounts must use checked arithmetic. Use `i128::checked_add`, `checked_mul`, etc. Never allow silent overflow.
- No function may reduce an LP's principal balance except `withdraw_from_pool` (LP-initiated) or default absorption (governed by reserve policy, capped).
- The `repay_invoice` function must verify the incoming amount matches the stored `face_value` exactly before routing. Short payments must not be silently accepted — return `RepaymentResult::Partial` and do not burn the NFT.
- The `attestation_hash` stored on an `InvoiceNFT` is immutable after mint. No function may modify it.

### Error handling

All errors must be variants of `SpireError`:

```rust
#[contracterror]
#[derive(Copy, Clone, Debug, Eq, PartialEq, PartialOrd, Ord)]
#[repr(u32)]
pub enum SpireError {
    InvoiceNotFound           = 1,
    InvoiceNotListed          = 2,
    InvoiceAlreadySold        = 3,
    InvoiceAlreadyRepaid      = 4,
    InvoiceDefaulted          = 5,
    InvoiceNotTransferable    = 6,
    DuplicateAttestation      = 7,
    BidExpired                = 8,
    BidBelowMinimum           = 9,
    InvalidRepaymentAmount    = 10,
    InsufficientPoolLiquidity = 11,
    PoolSharesInsufficient    = 12,
    DueLedgerNotPassed        = 13,
    Unauthorised              = 14,
    AttestationMissing        = 15,
}
```

### Events

All state-changing functions must emit typed events. Event schemas are public API — once deployed, they are immutable. Define events in the `events.rs` module of each contract and document all fields.

---

## Working with the pricing model

The pricing model lives in `contracts/liquidity-pool/src/scorer.rs`. Changes here directly affect LP returns and SME costs, so the bar is high.

### Before changing the risk scorer

- Open a governance issue describing the proposed change, the rationale, and the expected impact on pool APY
- Include backtested results from the `/pricing-research` directory if available
- Changes to the risk scorer require two maintainer approvals

### Scoring components

The risk scorer combines four components into a score from 0–100:

- `score_debtor_history(debtor)` — reads on-chain Stellar activity for the debtor
- `score_invoice_size(invoice, debtor_avg_tx)` — compares invoice to debtor's typical transaction size
- `score_tenor(due_ledger, current_ledger)` — prefers shorter tenors
- `score_asset(asset)` — prefers USDC/EURC over anchor local currencies

Each scorer is a pure function. Test any change to a scoring component independently before integration.

### Discount rate calculation

The auto-bid discount rate is calculated as:

```rust
let annual_rate = BASE_RATE + credit_spread(score) + liquidity_premium(pool_utilisation);
let discount = annual_rate * tenor_days / 365;
let purchase_price = face_value - (face_value * discount);
```

Any change to `credit_spread()` or `liquidity_premium()` must be accompanied by a table of example inputs and outputs demonstrating the effect across the score range.

---

## Writing the SDK

The Spire.js SDK lives in `/sdk` and is written in TypeScript with strict mode.

### Conventions

- `camelCase` for functions and variables, `PascalCase` for classes and types
- All public methods and types must have JSDoc comments
- Explicit return types on all public API surfaces
- No `any` — use `unknown` and narrow explicitly

### Building

```bash
cd sdk
pnpm install
pnpm build
pnpm test
```

---

## Testing

### Contract tests

Live in each contract's `tests/` directory. Use the Soroban test environment — no network required.

Every PR touching contract logic must include:

- Happy path test
- All relevant `SpireError` variants
- Edge cases: zero amounts, due-ledger boundary, partial payment, double mint with same attestation

**Financial invariant tests are required for any change to payment routing:**

- Test: SME receives exactly `bid_amount` on `accept_bid`
- Test: LP holder receives exactly `face_value` on `repay_invoice`
- Test: Pool shares correctly reflect total deposited capital plus accrued yield
- Test: Reserve buffer correctly absorbs defaults up to the buffer cap

```bash
cargo test -p invoice-nft
cargo test -p marketplace
cargo test -p liquidity-pool
cargo test -p repayment
```

### Integration tests

```bash
cargo test --test integration
```

Required for any change to `settlement.rs`, `routing.rs`, `pool.rs`, or `scorer.rs`.

---

## Pull request process

1. `cargo test` and `pnpm test` must pass locally before opening a PR
2. All PRs require one approving review from a maintainer
3. Maintainers squash-merge approved PRs into `main`

### PR checklist

- [ ] Code compiles without warnings
- [ ] All tests pass
- [ ] Financial invariant tests added for any payment routing change
- [ ] New tests added for new behaviour
- [ ] Doc comments updated for changed public API
- [ ] PR description links the issue it resolves

---

*Built on [Stellar](https://stellar.org) and [Soroban](https://soroban.stellar.org).*
