# XLS-0098: Default Protection Protocol for XLS-66 Vaults

| Field | Value |
|-------|-------|
| Title | Default Protection Protocol for XLS-66 Vaults |
| Type | Standard |
| Status | Draft |
| Author | Will Flores (wflores@wardprotocol.org) |
| Created | 2026-03-01 |
| Requires | XLS-66, XLS-70, XLS-80, XLS-20 |
| Discussion | https://github.com/XRPLF/XRPL-Standards/discussions/474 |

---

## Abstract

This document specifies an open protocol for default protection on XLS-66 lending vaults. The specification defines how institutions may offer coverage to vault depositors using existing XRPL primitives — permissioned domains (XLS-80), on-chain credentials (XLS-70), NFT policy certificates (XLS-20), and native escrow settlement — without introducing any off-chain intermediary or trusted third party.

**The protocol is a specification, not a service.** Any institution may implement it. All authoritative state lives on the XRP Ledger. Settlement is automatic and trustless once an escrow transaction is confirmed on-chain.

---

## Motivation

XLS-66 introduces institutional-grade lending vaults to the XRP Ledger. For institutions to deploy significant capital into these vaults, they require protection against borrower default. Without a standard for how that protection is structured and settled, each institution would build bespoke solutions — creating fragmentation, audit surface, and counterparty risk.

This specification defines the standard that makes institutional participation predictable:

- **Depositors** know exactly what coverage means, how it is verified, and how claims settle
- **Vault operators** know exactly what compliance requirements enable coverage eligibility
- **Protocol implementers** know exactly which XRPL transactions constitute a valid policy lifecycle

A single open standard benefits the entire XRPL DeFi ecosystem more than competing proprietary implementations.

---

## Specification

### 1. Roles

| Role | Description |
|------|-------------|
| **Protocol Implementer** | Any entity that deploys this specification — brings capital, underwriting, and regulatory licenses |
| **Vault Operator** | Operates an XLS-66 lending vault registered in a compliant XLS-80 domain |
| **Depositor** | Provides liquidity to an XLS-66 vault and may hold a coverage policy |
| **Claimant** | A depositor holding a valid policy NFT at the time of a confirmed vault default |

The specification does not define or require any central protocol operator. A reference implementation (Ward Protocol SDK) exists at https://github.com/wflores9/ward-protocol but is not authoritative — any conformant implementation is valid.

---

### 2. Protocol Stack

This specification layers on four existing XRPL standards:

```
┌─────────────────────────────────────────────────┐
│           XLS-0098 Default Protection           │
│         (this specification — the rules)        │
├──────────────┬──────────────┬───────────────────┤
│   XLS-80     │   XLS-70     │      XLS-20       │
│  Permissioned│  On-Chain    │   NFT Policy      │
│  Domains     │  Credentials │   Certificates    │
├──────────────┴──────────────┴───────────────────┤
│              XLS-66 Lending Vaults              │
├─────────────────────────────────────────────────┤
│         XRP Ledger Native Escrow                │
│      (PREIMAGE-SHA-256 + time condition)        │
└─────────────────────────────────────────────────┘
```

---

### 3. Compliance Domain (XLS-80)

A vault is eligible for coverage under this specification if and only if it is registered in a permissioned XLS-80 domain that meets the following requirements:

**3.1 Domain Requirements**
- The domain must require XLS-70 credentials for depositor participation
- The domain operator must maintain a coverage ratio of ≥ 200% (pool TVL / outstanding coverage)
- The domain configuration must be publicly verifiable on-chain

**3.2 Domain Registration**
Registration is an `AccountSet` transaction from the institution's wallet:

```json
{
  "TransactionType": "AccountSet",
  "Account": "<institution_address>",
  "Domain": "<ward_domain_hex>"
}
```

The domain hex encodes: `ward:vault:<vault_address>:domain:<domain_id>`

---

### 4. Credential Verification (XLS-70)

Depositor eligibility is gated by XLS-70 on-chain credentials:

**4.1 Credential Requirements**
- Each depositor wallet must hold a valid XLS-70 credential issued by a domain-authorized institution
- Credential validity is verified directly from the ledger — no off-chain oracle required
- Credentials have an on-chain expiry enforced by XRPL ledger time

**4.2 Credential Schema**
Credentials encode at minimum:
- `domain_id` — the XLS-80 domain this credential grants access to
- `credential_type` — KYC, AML, or accredited investor classification
- `issued_at` — XRPL ledger close time at issuance
- `expires_at` — XRPL ledger close time at expiry

---

### 5. Policy NFT (XLS-20)

Coverage is represented as an XLS-20 NFT minted by the institution from their own wallet.

**5.1 NFT Parameters**

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `NFTokenTaxon` | 281 | Ward Protocol reserved taxon |
| Flags | `tfBurnable` only | Policies are non-transferable — cannot be sold or forged |
| `tfTransferable` | Absent | Prevents secondary market; policy bound to issuing institution |
| URI | Policy metadata (hex) | Encodes all coverage terms on-chain |

**5.2 Policy Metadata Schema**

The NFT URI hex encodes a JSON object with the following fields:

```json
{
  "ward": "0098",
  "vault": "<xls66_vault_address>",
  "coverage_xrp": <integer>,
  "period_days": <30|90|180|365>,
  "beneficiary": "<depositor_address>",
  "institution": "<institution_address>",
  "issued_at": <xrpl_ledger_close_time>,
  "expiry": <xrpl_ledger_close_time>,
  "ward_signed": false
}
```

**Constraints:**
- Total URI must fit within XRPL's 256-byte NFT URI limit (verified in reference implementation)
- `ward_signed: false` is a required field — it documents that no protocol intermediary signed the policy
- `expiry` must use XRPL ledger close time, not local system time — immune to clock manipulation

---

### 6. Default Detection

A vault default is defined as a sustained breach of the XLS-66 health ratio threshold.

**6.1 Default Criteria**
A default event is valid under this specification if ALL of the following are true:

1. The XLS-66 vault health ratio has been below 1.5 for ≥ 3 consecutive ledger closes
2. The vault's `LoanManage` transaction shows the `tfLoanDefault` flag set
3. The vault address matches the `vault` field in the claimant's policy NFT URI
4. The policy NFT has not been previously burned (replay protection)
5. The policy expiry (XRPL ledger time) has not passed

**6.2 Multi-Confirmation Requirement**
Single-ledger default signals must not trigger settlement. Implementations MUST wait for ≥ 3 ledger closes before emitting a default event. This prevents manipulation via temporary balance changes.

**6.3 On-Chain Verification**
All 6 criteria above are verifiable directly from XRPL ledger state. No off-chain data source is required or permitted under this specification.

---

### 7. Claim Validation (9 Steps)

Before creating a settlement escrow, implementations MUST verify all 9 steps. Each step queries XRPL directly. If any step fails, the claim is rejected.

| Step | Check | XRPL Query |
|------|-------|------------|
| 1 | Policy NFT exists and is owned by claimant | `account_nfts` |
| 2 | Policy not expired (XRPL ledger time) | `server_info` → `validated_ledger.close_time` |
| 3 | Vault address matches policy NFT URI | Parse NFT URI hex |
| 4 | XLS-66 vault object exists on-chain | `ledger_entry` → Vault object |
| 5 | Vault health ratio below threshold | Parse Vault object fields |
| 6 | `tfLoanDefault` flag confirmed | Parse `LoanManage` tx flags |
| 7 | Depositor's loan position verifiable | `ledger_entry` → Loan object |
| 8 | Pool has sufficient balance for payout | `account_info` → pool balance |
| 9 | Policy NFT not previously burned | Absence from `account_nfts` |

**Note on Steps 4–7:** XLS-66 `Vault`, `Loan`, and `LoanBroker` ledger objects are not yet deployed on XRPL Altnet. The reference implementation validates steps 1–3 and 8–9 against live testnet; steps 4–7 are validated against mock fixtures in the unit test suite (75/75 passing). When XLS-66 is live, all 9 steps run end-to-end.

---

### 8. Settlement (Escrow)

Settlement uses XRPL native escrow with a dual condition: time lock + PREIMAGE-SHA-256 cryptographic condition.

**8.1 Escrow Creation**

```json
{
  "TransactionType": "EscrowCreate",
  "Account": "<institution_address>",
  "Destination": "<claimant_address>",
  "Amount": "<payout_in_drops>",
  "Condition": "<sha256_condition_hex>",
  "FinishAfter": <xrpl_ledger_time_plus_48h>
}
```

**8.2 Security Properties**

| Property | Mechanism |
|----------|-----------|
| No front-running | PREIMAGE-SHA-256 — only claimant holds the fulfillment |
| 48-hour dispute window | `FinishAfter` enforced by XRPL ledger time |
| Automatic settlement | `EscrowFinish` is permissionless after `FinishAfter` |
| Protocol intermediary not required | Once on-chain, escrow settles without any server |

**8.3 Replay Protection**

The policy NFT MUST be burned in the same settlement flow as the `EscrowFinish`. Implementations MUST verify NFT absence from `account_nfts` after burn. A burned policy cannot trigger a second claim — enforced by the ledger.

---

### 9. Coverage Ratio

Protocol implementers MUST maintain a pool coverage ratio of ≥ 200% at all times:

```
coverage_ratio = pool_tvl_xrp / outstanding_coverage_xrp ≥ 2.0
```

New policy minting MUST be halted if the ratio would fall below 200% after issuance.

**Dynamic premium pricing** is left to implementer discretion. The reference implementation uses:

```
premium = coverage_xrp × base_rate × risk_multiplier × period_multiplier
```

Where `risk_multiplier` is 0.5x–2.0x based on vault health ratio, and `period_multiplier` discounts longer commitments.

---

### 10. Conformance Requirements

An implementation MUST:
- Never sign XRPL transactions on behalf of institutions or depositors
- Never store authoritative state off-chain — all state reads from XRPL directly
- Use XRPL ledger close time for all expiry logic
- Require ≥ 3 ledger confirmations before emitting default events
- Validate all 9 claim steps before creating settlement escrow
- Burn the policy NFT atomically with settlement
- Maintain coverage ratio ≥ 200%

An implementation MUST NOT:
- Hold or custody wallet private keys
- Use a centralized database as the source of truth for policy state
- Permit policy NFT transfer (no `tfTransferable` flag)
- Allow `EscrowFinish` without valid PREIMAGE-SHA-256 fulfillment

---

## Rationale

### Why not a smart contract?
XRPL does not support general smart contracts. This specification is designed to use only XRPL's native transaction types — AccountSet, NFTokenMint, EscrowCreate, EscrowFinish, NFTokenBurn — which are available today and audited by Ripple's core engineering team.

### Why PREIMAGE-SHA-256 instead of simple time-lock?
A time-only escrow can be front-run — any party who knows the claim is pending could attempt to finish it. PREIMAGE-SHA-256 ensures only the party holding the secret preimage can collect the payout. This eliminates an entire class of MEV-style attacks.

### Why NFT non-transferability?
Insurance policies that can be sold create adverse selection — holders would sell policies on vaults they believe are healthy and retain policies on vaults they believe are distressed. Non-transferability ensures the policy holder is always the party with direct exposure to the vault.

---

## Backwards Compatibility

This specification introduces no changes to existing XRPL transaction types or ledger objects. It defines a usage pattern over existing primitives. Existing XLS-66, XLS-70, XLS-80, and XLS-20 implementations are fully compatible.

---

## Test Vectors

Five confirmed on-chain transactions (XRPL Altnet, 2026-03-01):

| Type | Transaction Hash |
|------|-----------------|
| `Payment` (premium) | `D541B6A2156E4BB3B22D9BD1D451598DF2D0387A25B73A5918A8779D76783169` |
| `NFTokenMint` (policy) | `B323815A6C7BA98935D2C2AA3CFC94BB956E59BA716A59430F2183D2AE148CDF` |
| `EscrowCreate` | `9BB570DBC6CB9EB11339FBBDA4920E03EC2CC49EC547CBF0D031C8AABC48B0A3` |
| `EscrowFinish` | `E65C35A568AE93E6D8A628F36A217DACB1B2A7E1A8F0A7B0912E510AED0A3DBB` |
| `NFTokenBurn` (replay protection) | `A5A0652C4DA629F0D46D2A3504FDC22E410848AF5D27E956E3997346A7B464D8` |

NFT Token ID: `000100004341F329192D196167C1C6F3D728D98731BE61AE9BEE37B200E41D10`

Full details: https://github.com/wflores9/ward-protocol/blob/main/testnet_proof.md

---

## Reference Implementation

- **Repository:** https://github.com/wflores9/ward-protocol
- **SDK:** `pip install ward-protocol`
- **Tests:** 75/75 unit tests passing, asyncio_mode=auto, no network required
- **License:** MIT

---

## Copyright

This specification is released under CC0. The reference implementation is MIT licensed.
