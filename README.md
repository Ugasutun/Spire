# Spire

**Decentralized invoice financing protocol on Stellar.**

Spire enables SMEs to unlock cash from unpaid invoices instantly — no bank, no credit history requirement, no weeks of waiting. Invoices are tokenized as Soroban NFTs, posted to a liquidity pool, and purchased by liquidity providers at a discount. When the debtor pays the face value, the LP earns the spread. Built on Soroban smart contracts with native Stellar path payments and anchor integration.

---

## Table of contents

- [Why Spire](#why-spire)
- [How it works](#how-it-works)
- [Architecture](#architecture)
- [Invoice NFT](#invoice-nft)
- [Liquidity pool](#liquidity-pool)
- [Contract interface](#contract-interface)
- [Pricing model](#pricing-model)
- [Default handling](#default-handling)
- [SDK — Spire.js](#sdk--spirejs)
- [Fee structure](#fee-structure)
- [Security model](#security-model)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Why Spire

Invoice financing is a $3 trillion global market almost entirely controlled by banks. An SME with a confirmed, signed invoice from a creditworthy buyer has a guaranteed future cash flow — but banks require established credit history, charge 15–30% annualised, and take 2–4 weeks to process. For a market vendor in Lagos or a supplier in Dhaka, that delay is not an inconvenience. It kills the business.

Spire removes every step that requires a bank. The invoice is tokenized on-chain with a cryptographic attestation proving both parties signed the underlying debt. Liquidity providers bid on it in seconds. The SME receives cash immediately. When the debtor pays, the LP receives the face value. The discount between purchase price and face value is the LP's yield.

Stellar makes this work at a scale no other chain supports. Path payments convert currencies atomically — an LP holding USDC can buy an invoice denominated in KES, with conversion happening in the same transaction. Stellar's fees ($0.0001 per transaction) make invoices of $100 economically viable. On Ethereum, gas costs would consume the entire spread.

---

## How it works

### 1. Invoice creation and attestation

The SME and debtor agree on an invoice off-chain (goods delivered, services rendered). Both parties sign the invoice document. The SME uploads the signed invoice to a decentralised storage provider and submits the document hash to the Spire contract, along with the debtor's Stellar address and the payment due date. The contract stores this as a verifiable `InvoiceNFT`.

### 2. Listing on the marketplace

The SME lists the NFT on the Spire marketplace, setting a minimum acceptable discount rate. The pool contract and individual LPs can see the listing and bid.

### 3. Purchase at a discount

A liquidity provider — either the automated pool or a direct buyer — purchases the NFT at a price below face value. The purchase price is determined by the invoice tenor, debtor creditworthiness, and market rates. On purchase, the SME receives the discounted amount immediately via Stellar path payment.

### 4. Repayment

On or before the due date, the debtor makes payment to the `RepaymentContract` by submitting the face value amount to the contract keyed by invoice ID. The contract verifies the amount, transfers it to the current NFT holder (the LP), and burns the NFT.

### 5. LP earns the spread

The LP paid the discounted price and received the face value. The difference is their yield. For a $10,000 invoice bought at $9,704 with a 60-day tenor, that is a 3.05% return in 60 days — approximately 18.5% annualised.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                       Spire Protocol                             │
│                                                                  │
│  ┌──────────────────┐    ┌──────────────────────────────────┐   │
│  │  InvoiceNFT      │    │  Marketplace                     │   │
│  │  Contract        │    │                                  │   │
│  │                  │    │  Listing registry                │   │
│  │  Mint            │───►│  Bid management                  │   │
│  │  Transfer        │    │  Price discovery                 │   │
│  │  Burn            │    │  Auto-pool integration           │   │
│  └──────────────────┘    └──────────────────────────────────┘   │
│                                    │                            │
│  ┌──────────────────┐              ▼                            │
│  │  RepaymentContract│    ┌──────────────────────────────────┐  │
│  │                  │    │  LiquidityPool                   │  │
│  │  Receive payment │    │                                  │  │
│  │  Verify amount   │    │  LP deposits · Auto-bid          │  │
│  │  Route to holder │    │  Risk scoring · Yield dist.      │  │
│  │  Burn NFT        │    │  Reserve buffer                  │  │
│  └──────────────────┘    └──────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
         ▲                                      ▼
   SME + Debtor wallets                   LP wallets
   (via Stellar anchors)                  (USDC or any asset)
```

### Core components

**`InvoiceNFT`** — Soroban token contract representing a single invoice. Stores: invoice ID, SME address, debtor address, face value, due ledger, attestation hash, current holder, and state. Has no transfer restriction by default but can be flagged non-transferable at mint time for regulated markets.

**`Marketplace`** — Contract managing the listing, bidding, and sale of invoice NFTs. Supports direct LP purchases and automated pool purchases. Emits events for all bid and sale activity.

**`LiquidityPool`** — Shared pool contract that LP depositors fund. The pool scores listed invoices and auto-bids on those meeting the quality threshold. Yield from repayments is distributed to LPs proportionally.

**`RepaymentContract`** — Receives debtor payments keyed by invoice ID. Verifies the amount against the face value, routes payment to the current NFT holder, and burns the NFT. Handles partial payments and default state transitions.

**`RiskScorer`** — Internal module used by the pool to score invoices before bidding. Reads debtor on-chain transaction history, invoice size relative to debtor history, tenor, and currency anchor quality.

---

## Invoice NFT

### Structure

```rust
pub struct InvoiceNFT {
    pub id: InvoiceId,                  // BytesN<32> — unique invoice identifier
    pub sme: Address,                   // invoice issuer (seller)
    pub debtor: Address,                // invoice payer (buyer)
    pub face_value: i128,               // total amount owed (7-decimal precision)
    pub asset: Address,                 // payment asset contract address (e.g. USDC)
    pub due_ledger: u32,                // ledger by which payment must arrive
    pub attestation_hash: BytesN<32>,   // SHA-256 of signed invoice document
    pub holder: Address,                // current NFT holder (SME until sold, then LP)
    pub state: InvoiceState,
    pub non_transferable: bool,         // if true, holder cannot re-sell
    pub minted_ledger: u32,
}

pub enum InvoiceState {
    Minted,       // created, not yet listed
    Listed,       // on marketplace, accepting bids
    Sold,         // purchased by LP, SME paid
    Repaid,       // debtor paid face value to LP
    Defaulted,    // due ledger passed without payment
    Disputed,     // under dispute resolution
    Burned,       // final settled state
}
```

### Attestation

The attestation hash is the critical fraud-prevention mechanism. Before minting, both the SME and debtor must sign the invoice document (PDF or structured JSON). The Spire frontend hashes the signed document and submits the hash to the contract. Any party can verify the invoice is genuine by hashing the original document and comparing to the stored hash. The debtor's signature on the document constitutes an on-chain commitment to pay — this is the legal basis for the financing.

---

## Liquidity pool

### How LPs participate

Liquidity providers deposit USDC (or any supported asset) into the `LiquidityPool` contract and receive pool shares proportional to their deposit. The pool contract auto-bids on invoices that meet the configured quality threshold and target APY. Yield from repaid invoices accumulates in the pool and is claimable proportionally.

### Invoice scoring

Before bidding, the `RiskScorer` evaluates each listed invoice on four criteria:

- **Debtor on-chain history** — transaction volume, payment consistency, and wallet age on Stellar. Higher scores for debtors with long anchor histories.
- **Invoice amount** — large invoices from unknown debtors score lower. Invoices within a reasonable range of the debtor's historical transaction size score higher.
- **Tenor** — shorter-dated invoices score higher. 30-day invoices score materially better than 90-day invoices.
- **Asset quality** — USDC and EURC score highest. Invoices in local anchor currencies score lower due to FX risk.

Invoices below the minimum score threshold are not auto-purchased by the pool but remain available for direct LP purchase.

### Reserve buffer

The pool maintains a 10% reserve buffer — 10% of pool capital is not deployed into invoices and instead held as a default absorber. When a default occurs, the loss is first absorbed by the reserve before touching LP principal. The reserve is replenished from incoming yield before any yield is distributed to LPs.

### Pool APY calculation

```
pool_apy = (total_yield_earned / total_capital_deployed) × (365 / avg_tenor_days)
```

The pool targets a configurable APY (default: 14%) by adjusting the auto-bid discount rate dynamically. If the pool is underdeployed, it bids more aggressively (higher discount) to attract invoices. If the pool is overdeployed, it tightens bids.

---

## Contract interface

### `mint_invoice`

Called by the SME to tokenize a new invoice.

```rust
pub fn mint_invoice(
    env: Env,
    sme: Address,
    debtor: Address,
    face_value: i128,
    asset: Address,
    due_ledger: u32,
    attestation_hash: BytesN<32>,
    non_transferable: bool,
) -> InvoiceId;
```

Requires `sme` auth. Requires `debtor` to have countersigned the attestation (verified off-chain, hash submitted). Emits `InvoiceMinted` event.

---

### `list_invoice`

Called by the SME to list a minted invoice on the marketplace.

```rust
pub fn list_invoice(
    env: Env,
    invoice_id: InvoiceId,
    sme: Address,
    min_discount_rate: u32,     // basis points — minimum acceptable discount
) -> ();
```

Requires `sme` auth. Transfers NFT custody to the `Marketplace` escrow. Emits `InvoiceListed` event.

---

### `bid_invoice`

Called by an LP to bid on a listed invoice.

```rust
pub fn bid_invoice(
    env: Env,
    invoice_id: InvoiceId,
    bidder: Address,
    bid_amount: i128,           // amount LP is willing to pay (< face_value)
    expiry_ledger: u32,         // bid expires if not accepted
) -> BidId;
```

Emits `BidPlaced` event. Bids are non-binding until accepted.

---

### `accept_bid`

Called by the SME to accept a bid and complete the sale.

```rust
pub fn accept_bid(
    env: Env,
    invoice_id: InvoiceId,
    bid_id: BidId,
    sme: Address,
) -> ();
```

Requires `sme` auth. Executes atomically: transfers `bid_amount` to SME via path payment, transfers NFT to bidder, sets state to `Sold`. Emits `InvoiceSold` event.

---

### `repay_invoice`

Called by the debtor to repay the invoice face value.

```rust
pub fn repay_invoice(
    env: Env,
    invoice_id: InvoiceId,
    debtor: Address,
    amount: i128,
) -> RepaymentResult;
```

Requires `debtor` auth. Transfers `face_value` to current NFT holder via path payment. Burns NFT. Emits `InvoiceRepaid` event. Returns `RepaymentResult::Success` or `RepaymentResult::Partial`.

---

### `mark_defaulted`

Callable permissionlessly after due ledger has passed without repayment.

```rust
pub fn mark_defaulted(
    env: Env,
    invoice_id: InvoiceId,
) -> ();
```

Transitions state to `Defaulted`. Emits `InvoiceDefaulted` event. Triggers reserve buffer absorption if the NFT is held by the pool.

---

### `deposit_to_pool`

Called by LPs to deposit capital into the liquidity pool.

```rust
pub fn deposit_to_pool(
    env: Env,
    provider: Address,
    asset: Address,
    amount: i128,
) -> PoolShares;
```

---

### `withdraw_from_pool`

Called by LPs to redeem pool shares for underlying capital plus yield.

```rust
pub fn withdraw_from_pool(
    env: Env,
    provider: Address,
    shares: PoolShares,
) -> i128;     // returned amount
```

---

### `claim_yield`

Called by LPs to claim accrued yield without withdrawing principal.

```rust
pub fn claim_yield(
    env: Env,
    provider: Address,
) -> i128;
```

---

## Pricing model

The discount rate applied to an invoice determines how much the SME receives and how much the LP earns. The formula is:

```
discount = annual_rate × (tenor_days / 365)
purchase_price = face_value × (1 - discount)
lp_profit = face_value - purchase_price
```

The pool's auto-bid annual rate is derived from:

```
base_rate = risk_free_benchmark + credit_spread + liquidity_premium
```

Where:
- `risk_free_benchmark` is configurable by governance (default: 6%)
- `credit_spread` is the debtor's risk score contribution (0–15%)
- `liquidity_premium` is added when the pool is underdeployed (0–5%)

Individual LPs bidding directly can set any discount rate they choose.

---

## Default handling

When a debtor fails to pay by the due ledger:

1. Any caller submits `mark_defaulted()` to transition the invoice to `Defaulted` state
2. If held by the pool, the loss is first absorbed by the reserve buffer
3. If the reserve is insufficient, LP shares are marked down proportionally
4. The NFT holder (pool or direct LP) now holds an on-chain claim against the debtor
5. If the debtor is a registered anchor user, the anchor can be notified to pursue recourse through their off-chain mechanisms
6. A `dispute_invoice()` function allows the SME or LP to initiate a dispute if the default was due to a contract error

Late payment is also supported — a debtor can call `repay_invoice()` after the due ledger has passed. The contract accepts late payment, routes proceeds to the holder, and closes the default.

---

## SDK — Spire.js

### Installation

```bash
npm install @spire-protocol/sdk
```

### Mint and list an invoice (SME)

```typescript
import { SpireClient } from '@spire-protocol/sdk';

const spire = new SpireClient({ network: 'mainnet' });

const invoiceId = await spire.mintInvoice({
  wallet,                         // Freighter or keypair
  debtor: 'GBTZ…3MWP',
  faceValue: 10_000_00,           // $10,000.00 USDC (7 decimals)
  asset: 'USDC',
  dueDays: 60,
  attestationHash: hashInvoiceDoc(signedPdf),
});

await spire.listInvoice({
  wallet,
  invoiceId,
  minDiscountRate: 1000,          // 10% annualised minimum
});
```

### Purchase an invoice (LP)

```typescript
const bidId = await spire.bidInvoice({
  wallet,
  invoiceId,
  bidAmount: 9_704_00,            // $9,704.00 USDC
  expiryDays: 2,
});
```

### Repay an invoice (debtor)

```typescript
await spire.repayInvoice({
  wallet,
  invoiceId,
  amount: 10_000_00,
});
```

### Pool LP participation

```typescript
import { SpireLPClient } from '@spire-protocol/sdk';

const lp = new SpireLPClient({ wallet, network: 'mainnet' });

await lp.deposit({ asset: 'USDC', amount: 5_000_00 });
const yield_ = await lp.claimYield();
```

---

## Fee structure

| Action | Who pays | Amount |
|---|---|---|
| `mint_invoice` tx | SME | ~0.001 XLM once |
| `list_invoice` tx | SME | ~0.001 XLM |
| `bid_invoice` tx | LP | ~0.001 XLM |
| `accept_bid` tx | SME | ~0.001 XLM |
| `repay_invoice` tx | Debtor | ~0.001 XLM |
| Protocol fee on repayment | Pool | 0.5% of face value (v1) |
| `deposit_to_pool` tx | LP | ~0.001 XLM |

The 0.5% protocol fee on repayment is taken from the LP's proceeds, not the SME or debtor. It is the primary protocol revenue mechanism and is governed by the protocol fee parameter. It may be reduced or removed for specific anchor partnerships.

---

## Security model

### What Spire can do

- Hold invoice NFTs in escrow during listing
- Transfer the purchase price to the SME on bid acceptance
- Transfer face value to the LP holder on repayment
- Mark invoices as defaulted after due ledger passes
- Distribute LP yield proportionally from the pool

### What Spire cannot do

- Force a debtor to pay (repayment is voluntary and enforced by attestation + anchor recourse)
- Modify the face value or due date of a minted invoice
- Transfer an invoice NFT marked as non-transferable
- Access LP principal other than to route repayments and absorb defaults per the reserve policy

### Threat model

**Fake invoice fraud** — An SME could attempt to mint an invoice for a non-existent debt. Mitigated by the attestation hash requirement: the debtor must have signed the original document. The pool's `RiskScorer` also cross-references debtor on-chain history — unknown debtors receive low scores and high discount rates.

**Collusion** — An SME and a fake "debtor" could collude to mint invoices knowing the debtor will default. Mitigated by the risk scoring model, the reserve buffer, and the protocol fee that makes repeated fraud expensive.

**Double-spend** — An SME cannot list the same invoice twice. The `mint_invoice` function checks that the `attestation_hash` is unique — a hash already stored in contract state cannot be minted again.

**Path payment manipulation** — All currency conversions use Stellar's native DEX path payments. No external price oracle is required, and no MEV-style manipulation of path payment prices is possible at Stellar's settlement layer.

### Audit status

- [ ] Internal review — in progress
- [ ] External audit — planned pre-mainnet
- [ ] Bug bounty — planned post-audit

---

## Roadmap

### v0.1 — Testnet alpha
- [ ] `InvoiceNFT` contract
- [ ] `Marketplace` contract with direct LP bidding
- [ ] `RepaymentContract` with default handling
- [ ] Testnet deployment

### v0.2 — Liquidity pool + pricing
- [ ] `LiquidityPool` contract with auto-bid
- [ ] `RiskScorer` module
- [ ] Dynamic discount rate engine
- [ ] Spire.js SDK
- [ ] Backend API

### v0.3 — Frontend + anchor integrations
- [ ] SME dashboard (mint, list, track)
- [ ] LP dashboard (pool, direct bids, yield)
- [ ] Anchor partnerships for debtor recourse
- [ ] Multi-asset invoice support

### v1.0 — Mainnet
- [ ] External security audit
- [ ] Bug bounty program
- [ ] Mainnet deployment
- [ ] Anchor integration for M-PESA, bKash, Wave

### v2.0 — Secondary market + governance
- [ ] Secondary market for invoice NFT re-trading
- [ ] Protocol fee governance
- [ ] Invoice fractionalisation (split large invoices across multiple LPs)
- [ ] Cross-border multi-currency invoice support

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for full guidelines.

```bash
git clone https://github.com/your-org/spire
cd spire
cargo build
cargo test
```

---

## License

MIT — see [LICENSE](./LICENSE).

---

*Built on [Stellar](https://stellar.org) and [Soroban](https://soroban.stellar.org).*
