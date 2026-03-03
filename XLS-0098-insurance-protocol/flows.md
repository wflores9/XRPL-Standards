# XLS-0098 — Protocol Flows

Transaction flows for the Ward Protocol default protection specification.
All flows use existing XRPL native transaction types. No custom ledger objects are introduced.

---

## Core Principle

```
Institution integrates Ward SDK
         │
         ▼
Ward builds unsigned XRPL transaction
         │
         ▼                          Ward stops here.
Institution signs with own wallet   ───────────────
         │
         ▼
Institution submits to XRPL
         │
         ▼
XRPL ledger enforces outcome (automatic, permanent, auditable)
```

Ward never holds keys. Ward never submits transactions. Once a transaction is on-chain, Ward's operational status is irrelevant.

---

## Flow 1 — Vault Registration

Registers an XLS-66 vault in the Ward Protocol XLS-80 compliance domain.

```
Institution Wallet          Ward SDK              XRPL Ledger
      │                        │                       │
      │  register_vault()      │                       │
      │───────────────────────►│                       │
      │                        │                       │
      │  unsigned AccountSet   │                       │
      │◄───────────────────────│                       │
      │  (ward_signed=false)   │                       │
      │                        │                       │
      │  sign + submit ────────────────────────────────►
      │                        │                       │
      │                        │   AccountSet confirmed │
      │◄───────────────────────────────────────────────│
      │                        │  Vault in domain       │
      │                        │  registry on-chain     │
```

**Transaction type:** `AccountSet`
**Signer:** Institution wallet (never Ward)
**Result:** Vault address appears in XLS-80 domain registry on XRPL

---

## Flow 2 — Credential Issuance (XLS-70)

Institution issues a KYC/AML credential to a depositor wallet.

```
Institution Wallet          Ward SDK              XRPL Ledger
      │                        │                       │
      │  issue_credential()    │                       │
      │───────────────────────►│                       │
      │                        │                       │
      │  unsigned NFTokenMint  │                       │
      │◄───────────────────────│                       │
      │  (credential schema)   │                       │
      │                        │                       │
      │  sign + submit ────────────────────────────────►
      │                        │                       │
      │                        │  Credential NFT live   │
      │◄───────────────────────────────────────────────│
      │                        │  Depositor now eligible│
```

**Transaction type:** `NFTokenMint`
**Signer:** Institution wallet
**Result:** XLS-70 credential NFT in depositor's account, verifiable on-chain

---

## Flow 3 — Policy Purchase

Depositor pays premium; institution mints coverage policy NFT.

```
Depositor Wallet    Institution Wallet    Ward SDK         XRPL Ledger
      │                    │                 │                  │
      │  pay premium       │                 │                  │
      │───────────────────►│                 │                  │
      │  (Payment tx)      │                 │                  │
      │                    │                 │                  │
      │                    │  mint_policy()  │                  │
      │                    │────────────────►│                  │
      │                    │                 │                  │
      │                    │ unsigned        │                  │
      │                    │ NFTokenMint     │                  │
      │                    │◄────────────────│                  │
      │                    │ (ward_signed=   │                  │
      │                    │  false)         │                  │
      │                    │                 │                  │
      │                    │  sign + submit ─────────────────────►
      │                    │                 │                  │
      │                    │                 │  Policy NFT live  │
      │◄───────────────────────────────────────────────────────│
      │  Policy NFT in     │                 │                  │
      │  depositor account │                 │                  │
```

**Transaction types:** `Payment` (premium), `NFTokenMint` (policy)
**NFT flags:** `tfBurnable` only — non-transferable by design
**NFT taxon:** 281 (Ward Protocol reserved)
**Ward signs:** Never

---

## Flow 4 — Default Detection

WardMonitor runs in institution's own infrastructure. Ward provides detection logic; institution acts on events.

```
XRPL Ledger          WardMonitor              Institution Handler
     │            (institution's infra)              │
     │                    │                          │
     │  LoanManage tx     │                          │
     │  (tfLoanDefault)   │                          │
     │───────────────────►│                          │
     │                    │                          │
     │  ledger close 1    │                          │
     │───────────────────►│  (1/3 confirmations)     │
     │  ledger close 2    │                          │
     │───────────────────►│  (2/3 confirmations)     │
     │  ledger close 3    │                          │
     │───────────────────►│  (3/3 — threshold met)   │
     │                    │                          │
     │                    │  on_verified_default()   │
     │                    │─────────────────────────►│
     │                    │                          │
     │                    │  DefaultEvent emitted    │  Institution
     │                    │  vault_address           │  decides what
     │                    │  health_ratio: 0.82      │  to do next
     │                    │  vault_loss_xrp: N       │
```

**Minimum confirmations:** 3 ledger closes (≈12 seconds)
**Runs in:** Institution's own infrastructure — not Ward's servers
**Ward's role:** Provides detection algorithm only

---

## Flow 5 — Claim Validation (9 Steps)

All 9 steps query XRPL directly. Any failure rejects the claim.

```
ClaimValidator              XRPL Ledger
      │                          │
      │  Step 1: NFT ownership   │
      │  account_nfts ──────────►│
      │◄─────────────────────────│ ✓ NFT exists, owned by claimant
      │                          │
      │  Step 2: Expiry check    │
      │  server_info ───────────►│
      │◄─────────────────────────│ ✓ ledger_time < policy expiry
      │                          │
      │  Step 3: Vault match     │
      │  (parse NFT URI hex)     │ ✓ vault address matches
      │                          │
      │  Step 4: Vault exists    │
      │  ledger_entry ──────────►│
      │◄─────────────────────────│ ✓ XLS-66 Vault object found
      │                          │
      │  Step 5: Health ratio    │
      │  (parse Vault object)    │ ✓ ratio < 1.5 threshold
      │                          │
      │  Step 6: Default flag    │
      │  (parse LoanManage tx)   │ ✓ tfLoanDefault confirmed
      │                          │
      │  Step 7: Loan position   │
      │  ledger_entry ──────────►│
      │◄─────────────────────────│ ✓ Depositor's Loan object found
      │                          │
      │  Step 8: Pool balance    │
      │  account_info ──────────►│
      │◄─────────────────────────│ ✓ Pool has sufficient XRP
      │                          │
      │  Step 9: Replay check    │
      │  account_nfts ──────────►│
      │◄─────────────────────────│ ✓ NFT not previously burned
      │                          │
      │  → CLAIM APPROVED        │
      │  → proceed to escrow     │
```

**Note:** Steps 4–7 require XLS-66 ledger objects not yet on Altnet. Validated with mock fixtures in unit tests (75/75). Will run end-to-end when XLS-66 is deployed.

---

## Flow 6 — Settlement (Escrow)

PREIMAGE-SHA-256 + time lock. Only claimant can finish. Ward server irrelevant after EscrowCreate.

```
Claimant            Institution Wallet    Ward SDK         XRPL Ledger
    │                      │                 │                  │
    │  generate preimage   │                 │                  │
    │  (offline, secret)   │                 │                  │
    │                      │                 │                  │
    │  share condition_hex │                 │                  │
    │─────────────────────►│                 │                  │
    │  (NOT the preimage)  │                 │                  │
    │                      │                 │                  │
    │                      │  create_escrow()│                  │
    │                      │────────────────►│                  │
    │                      │                 │                  │
    │                      │ unsigned        │                  │
    │                      │ EscrowCreate    │                  │
    │                      │◄────────────────│                  │
    │                      │                 │                  │
    │                      │  sign + submit ─────────────────────►
    │                      │                 │                  │
    │                      │                 │  Escrow on-chain  │
    │                      │                 │  48h dispute      │
    │                      │                 │  window begins    │
    │                      │                 │                  │
    │  [48 hours pass]     │                 │                  │
    │                      │                 │                  │
    │  EscrowFinish        │                 │                  │
    │  + fulfillment_hex ──────────────────────────────────────►│
    │  + NFTokenBurn ──────────────────────────────────────────►│
    │                      │                 │                  │
    │◄─────────────────────────────────────────────────────────│
    │  Payout received     │                 │  NFT burned       │
    │  Replay impossible   │                 │  (ledger confirms)│
```

**Key security properties:**
- Ward never sees the preimage — claimant generates it offline
- EscrowFinish is permissionless after FinishAfter — anyone can submit
- NFT burn is atomic with settlement — replay attacks impossible
- If Ward's servers go offline: escrow still settles automatically at FinishAfter

---

## State Summary

| Phase | XRPL State Change | Transaction |
|-------|-------------------|-------------|
| Vault registered | AccountSet domain field set | `AccountSet` |
| Credential issued | Credential NFT in depositor account | `NFTokenMint` |
| Policy active | Policy NFT in institution account | `NFTokenMint` |
| Default confirmed | No state change — detection only | (WebSocket subscription) |
| Escrow created | EscrowObject created on-chain | `EscrowCreate` |
| Claim settled | EscrowObject removed, XRP transferred | `EscrowFinish` |
| Policy burned | NFT removed from `account_nfts` | `NFTokenBurn` |

All state transitions are XRPL native transactions. No off-chain state is authoritative.
