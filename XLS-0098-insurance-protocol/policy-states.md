# XLS-0098 — Policy States

Policy state under this specification is derived entirely from on-chain XRPL ledger state.
There is no off-chain policy database. A policy's state is determined by querying the ledger directly.

---

## State Machine

```
                    ┌─────────────────┐
                    │                 │
          ┌─────────│    NONEXISTENT  │
          │         │                 │
          │         └─────────────────┘
          │                  │
          │  NFTokenMint      │
          │  (institution)    │
          │                  ▼
          │         ┌─────────────────┐
          │         │                 │
          │         │     ACTIVE      │◄──────────────┐
          │         │                 │               │
          │         └─────────────────┘               │
          │           │           │                   │
          │  ledger   │           │  default          │
          │  time >   │           │  detected         │
          │  expiry   │           │  (3 confirmations)│
          │           ▼           ▼                   │
          │  ┌──────────┐  ┌──────────────┐           │
          │  │          │  │              │  dispute   │
          │  │ EXPIRED  │  │  DEFAULTED   │──────────►│
          │  │          │  │              │  resolved  │
          │  └──────────┘  └──────────────┘           │
          │       │               │                   │
          │  NFT  │         claim │                   │
          │  burn │    validated  │                   │
          │       │   (9 steps)   │                   │
          │       ▼               ▼                   │
          │  ┌──────────────────────────┐             │
          └─►│                          │             │
             │         BURNED           │             │
             │   (absent from ledger)   │             │
             │                          │             │
             └──────────────────────────┘             │
                                                      │
             (BURNED is terminal — no further claims) │
```

---

## State Definitions

### NONEXISTENT
**Ledger condition:** No NFT with Ward taxon (281) exists for this vault/beneficiary combination.

**How to verify:**
```
GET account_nfts(institution_address)
→ filter NFTokenTaxon == 281
→ filter URI contains vault_address
→ result: empty
```

---

### ACTIVE
**Ledger condition:** Policy NFT exists in institution's account AND XRPL ledger close time < policy expiry.

**How to verify:**
```
GET account_nfts(institution_address)
→ find NFT where NFTokenTaxon == 281
→ decode URI hex → parse JSON
→ check: ledger_close_time < expiry
→ result: policy found, not expired
```

**On-chain data available:**
- `vault` — the XLS-66 vault address covered
- `coverage_xrp` — maximum payout amount
- `beneficiary` — depositor address entitled to claim
- `expiry` — XRPL ledger close time at which policy lapses
- `institution` — institution address that minted the policy
- `ward_signed` — always `false`

**Transitions out of ACTIVE:**
- Ledger time passes `expiry` → **EXPIRED**
- Vault default confirmed (3 ledger closes below health threshold) → **DEFAULTED**

---

### EXPIRED
**Ledger condition:** Policy NFT still exists but XRPL ledger close time ≥ policy expiry.

**How to verify:**
```
GET server_info → validated_ledger.close_time
GET account_nfts(institution_address) → find policy NFT
decode URI → check: close_time >= expiry
→ result: policy exists but lapsed
```

**No claim is possible in EXPIRED state.**

Expired NFTs should be burned by the institution to reclaim ledger reserves:

```
NFTokenBurn(
  account=institution_address,
  nftoken_id=expired_nft_id
)
```

**Transitions out of EXPIRED:**
- Institution burns NFT → **BURNED**

---

### DEFAULTED
**Ledger condition:** Policy NFT exists, not expired, AND vault health ratio has been below 1.5 for ≥ 3 consecutive ledger closes with `tfLoanDefault` flag confirmed.

**How to verify:**
```
GET account_nfts → policy NFT exists, not expired
GET ledger_entry(vault_address) → XLS-66 Vault object
→ parse health_ratio < 1.5
GET tx(loan_manage_tx_hash) → check Flags & tfLoanDefault
→ result: default confirmed on-chain
```

**This state triggers the 9-step claim validation flow.**

A policy in DEFAULTED state has a 48-hour dispute window before EscrowFinish is executable. During this window:
- Community members or the institution may challenge the default classification
- If challenged successfully: policy returns to ACTIVE (dispute resolved, no default)
- If not challenged: claim proceeds to settlement

**Transitions out of DEFAULTED:**
- Dispute resolved → **ACTIVE**
- 9-step validation passes + EscrowCreate submitted → settlement in progress → **BURNED** (after EscrowFinish + NFTokenBurn)

---

### BURNED
**Ledger condition:** NFT is absent from `account_nfts`. The ledger has no record of this token ID.

**How to verify:**
```
GET account_nfts(institution_address)
→ filter NFTokenTaxon == 281, NFTokenID == target
→ result: not found
```

**BURNED is a terminal state.** A burned policy cannot:
- Be reinstated
- Trigger a second claim (replay protection)
- Be transferred (was already non-transferable)

This state is enforced by the XRPL ledger natively — once an NFT is burned, it cannot reappear.

---

## State Query Reference

All state queries use standard XRPL API methods. No Ward API endpoint is required for state verification — any XRPL node can be used.

| State | Primary Query | Key Field |
|-------|---------------|-----------|
| NONEXISTENT | `account_nfts` | NFT absent |
| ACTIVE | `account_nfts` + `server_info` | NFT present, `close_time < expiry` |
| EXPIRED | `account_nfts` + `server_info` | NFT present, `close_time >= expiry` |
| DEFAULTED | `ledger_entry` (Vault) | `health_ratio < 1.5` + `tfLoanDefault` |
| BURNED | `account_nfts` | NFT absent (post-settlement) |

---

## NFT Metadata Reference

Policy metadata is encoded in the NFT URI field as hex-encoded JSON.

**Decode example (Python):**
```python
from xrpl.utils import hex_to_str
import json

nft_uri_hex = "7B2277617264223A2230303938..."  # from account_nfts response
metadata = json.loads(hex_to_str(nft_uri_hex))

print(metadata["vault"])         # XLS-66 vault address
print(metadata["coverage_xrp"]) # coverage amount
print(metadata["expiry"])        # XRPL ledger close time
print(metadata["ward_signed"])   # always False
```

**Full schema:**
```json
{
  "ward": "0098",
  "vault": "<r-address>",
  "coverage_xrp": 10000,
  "period_days": 90,
  "beneficiary": "<r-address>",
  "institution": "<r-address>",
  "issued_at": 760000000,
  "expiry": 767760000,
  "ward_signed": false
}
```

**URI size constraint:** Total encoded URI must be ≤ 256 bytes (XRPL limit). The reference implementation verifies this before returning the mint transaction.

---

## Invariants

These invariants hold for any conformant implementation:

1. A policy in BURNED state can never transition to any other state
2. `ward_signed` is always `false` in policy metadata — no exceptions
3. `expiry` always uses XRPL ledger close time — never local system time
4. A policy NFT never has `tfTransferable` flag — policies cannot be sold
5. NFT burn and EscrowFinish occur in the same settlement flow — atomically
6. Policy state is always derivable from XRPL ledger state alone — no Ward API required
