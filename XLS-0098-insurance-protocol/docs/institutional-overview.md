# Ward Protocol — Institutional Overview

**The open specification for XLS-66 vault default protection on the XRP Ledger.**

This document is for institutions — InsurTechs, Lloyd's syndicates, parametric insurance firms, and institutional capital allocators — evaluating Ward Protocol as an integration target.

---

## The Problem

XRPL's XLS-66 standard introduces institutional-grade lending vaults to the XRP Ledger. Institutions will not deposit significant capital into uninsured lending protocols. Without a standard for default protection, XRPL DeFi remains limited to retail participants.

The gap is not capital — it is infrastructure. Institutions have capital, underwriting capability, and regulatory licenses. What they lack is a standardized, on-chain mechanism to offer that protection in a way that is:

- Auditable by any third party from the ledger alone
- Settlement-guaranteed without counterparty dependency
- Legally structured as software, not an insurance product

Ward Protocol is that mechanism.

---

## What Ward Protocol Is

Ward Protocol is a **software specification** — not an insurance company, not a fund, not a service provider.

Ward defines the open standard for how default protection on XLS-66 vaults is structured, verified, and settled. Institutions that implement the specification bring their own:

- Capital (insurance pool)
- Underwriting models and risk pricing
- Regulatory licenses and compliance infrastructure
- Wallet keys and operational infrastructure

**Ward provides the rails. Institutions run on them.**

This is structurally identical to how Stripe operates in payments — Stripe defines the protocol that moves money. Stripe doesn't hold funds. Institutions that integrate Stripe bring the money, the customers, and the licenses. Ward is the same model for on-chain default protection.

---

## The Core Invariant

Ward never signs a transaction on behalf of an institution or depositor.

Every Ward SDK method that modifies XRPL state returns an **unsigned transaction**:

```python
# Ward builds. Institution signs. XRPL settles.
tx = TxBuilder.create_escrow(
    institution_address=institution_wallet.address,
    beneficiary=depositor_address,
    amount_xrp=payout_amount,
)
# tx.ward_signed == False — always, by design

# Institution signs with their own wallet:
signed = institution_wallet.sign(tx.unsigned_tx)
await client.submit(signed)
```

Once the escrow transaction is confirmed on-chain, Ward's servers are irrelevant. The XRP Ledger enforces payout automatically at maturity. This property is the foundation of the legal opinion that Ward is a software protocol, not an insurance operator.

---

## Integration Model

### What the Institution Provides
- Insurance pool capital (XRP, held in institution's own wallet)
- Underwriting decisions and risk pricing
- XLS-70 credential issuance for depositor KYC/AML
- Regulatory compliance for their jurisdiction
- Operational infrastructure (servers, monitoring, keys)

### What Ward Provides
- Open specification for policy structure (XLS-0098 draft)
- SDK for unsigned XRPL transaction construction
- Vault health monitoring module (embeddable in institution's infra)
- 9-step claim validation algorithm
- PREIMAGE-SHA-256 escrow settlement pattern
- NFT policy metadata schema (taxon=281)

### What the XRPL Ledger Provides
- Immutable policy certificates (XLS-20 NFTs)
- Automatic escrow settlement (native Escrow)
- Credential verification (XLS-70)
- Vault health data (XLS-66)
- Compliance domain registry (XLS-80)

---

## Protocol Stack

| Layer | Standard | Role |
|-------|----------|------|
| Application | XLS-0098 (Ward) | Default protection specification |
| Identity | XLS-80 + XLS-70 | Permissioned domains + credentials |
| Asset | XLS-20 | NFT policy certificates |
| Vault | XLS-66 | Lending vault health monitoring |
| Settlement | XRPL Escrow | PREIMAGE-SHA-256 conditioned payout |
| Network | XRP Ledger | Immutable state, ~4s settlement |

---

## What's Been Built and Proven

**Current status: Testnet-proven SDK, seeking first institutional integration.**

| Deliverable | Status |
|-------------|--------|
| SDK v0.1.0 (5 hardened modules) | ✓ Complete |
| 75/75 unit tests passing | ✓ Complete |
| 5 on-chain transactions confirmed (XRPL Altnet) | ✓ Complete |
| PREIMAGE-SHA-256 escrow settlement | ✓ Confirmed on-chain |
| Replay protection via NFT burn | ✓ Confirmed on-chain |
| XRPLF Discussion #474 — active engagement | ✓ Live |
| XLS-0098 draft specification | ✓ Filed |
| Security audit | Planned Q2 2026 |
| Legal opinion (software protocol, not insurer) | Planned Q2 2026 |
| First institutional integration | Seeking partner |

---

## Confirmed On-Chain Transactions

All 5 transactions verified on XRPL Testnet Explorer — 2026-03-01:

| Step | Transaction Hash |
|------|-----------------|
| Premium Payment | `D541B6A2156E4BB3B22D9BD1D451598DF2D0387A25B73A5918A8779D76783169` |
| Policy NFT Mint | `B323815A6C7BA98935D2C2AA3CFC94BB956E59BA716A59430F2183D2AE148CDF` |
| Escrow Create | `9BB570DBC6CB9EB11339FBBDA4920E03EC2CC49EC547CBF0D031C8AABC48B0A3` |
| Escrow Finish | `E65C35A568AE93E6D8A628F36A217DACB1B2A7E1A8F0A7B0912E510AED0A3DBB` |
| NFT Burn (replay protection) | `A5A0652C4DA629F0D46D2A3504FDC22E410848AF5D27E956E3997346A7B464D8` |

---

## Security Properties

Eight invariants enforced by the XRPL ledger — not by Ward's server:

| Invariant | How Enforced |
|-----------|--------------|
| Institution holds all keys | Architecture — Ward has no wallet |
| Policies non-transferable | `tfBurnable` only, XRPL enforces |
| No front-running on claims | PREIMAGE-SHA-256 — claimant holds the key |
| Clock manipulation impossible | XRPL ledger time for all expiry |
| Multi-confirmation before default | 3 ledger closes required |
| Replay attacks impossible | NFT burns on settlement |
| Rate limiting on claims | 3 attempts per NFT per 5 minutes |
| Address validation | Ledger codec validates all inputs |

Full attack surface analysis: [`security_notes.md`](../security_notes.md)

---

## Integration Timeline

A first institutional integration follows this sequence:

**Week 1–2:** Technical integration
- Clone SDK, run 75 unit tests
- Connect to XRPL Testnet
- Register test vault, mint test policy, simulate default
- Verify end-to-end flow against testnet_proof.md

**Week 3–4:** Configuration
- Set pool address and initial capital
- Configure XLS-80 domain and XLS-70 credential schema
- Set premium pricing (Ward provides reference formula)
- Define monitoring thresholds and alert handlers

**Q2 2026:** Mainnet deployment
- Security audit completion
- Legal opinion confirming software protocol structure
- First mainnet vault integration
- Live coverage ratio monitoring

---

## Why This Structure Matters Legally

The institution's legal team will want to know: is Ward an insurance company?

The answer depends entirely on whether Ward signs transactions. If Ward signs or submits transactions on behalf of depositors, Ward is intermediating a financial transaction — and regulators may treat it as operating an insurance product.

Ward never signs. The `ward_signed: false` field in every policy NFT's metadata documents this on-chain, permanently, for every policy ever issued.

This means:
- Ward can obtain a legal opinion confirming it is a software protocol
- Institutions can integrate Ward without taking on Ward's regulatory surface
- The institution's existing insurance licenses cover the product they offer using Ward's rails

---

## Why XRPL

XRPL's native escrow with PREIMAGE-SHA-256 conditions provides a settlement mechanism that no EVM chain can match today:

- **~4 second finality** — faster than any EVM L1
- **Native time-lock** — no smart contract required, no audit surface
- **Cryptographic payout control** — only the claimant's preimage finishes the escrow
- **Ledger-enforced NFT non-transferability** — no workaround possible
- **Zero gas cost volatility** — fixed fee structure, predictable operations

For an insurance product, predictable and auditable settlement is not a nice-to-have. It is the product.

---

## Next Steps

If you are an institution evaluating Ward Protocol:

1. **Run the SDK** — `pip install ward-protocol` and run the 75 unit tests
2. **Review testnet proof** — all 5 transactions are on the public XRPL explorer
3. **Read the spec** — XLS-0098 draft at https://github.com/XRPLF/XRPL-Standards
4. **Contact Ward** — wflores@wardprotocol.org

The spec is open. The rails are yours.

---

*Ward Protocol · Software Specification · Not an insurance company*
*GitHub: github.com/wflores9/ward-protocol · XRPLF Discussion #474*
