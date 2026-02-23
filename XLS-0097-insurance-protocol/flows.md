# XLS-0097 — Institutional DeFi Insurance Protocol  
## Functional Flow Diagrams

This document defines the complete operational flows under **XLS-0097** for institutional-grade DeFi insurance on XRPL, integrated with:

- XLS-0065 Vaults  
- XLS-0066 Loans  
- XLS-0020 Policy NFT  
- AMM Pool reserves  
- lsfLoanDefault flag  
- 48-hour Escrow dispute window  

---

## 1. Policy Issuance Flow

```mermaid
sequenceDiagram
    participant D as Depositor (Institutional Client)
    participant O as Operator (Underwriting Node)
    participant V as Vault (XLS-0065)
    participant A as AMM Pool (Insurance Liquidity Pool)
    participant N as Policy NFT (XLS-0020)

    Note over D,O: Step 1 — Policy Application Submitted  
    D->>O: Submit policy parameters (coverage amount, duration, asset class)
    
    Note over O,V: Step 2 — Collateralization  
    O->>V: Lock collateral using VaultCreate (XLS-0065)
    V-->>O: VaultID and lock confirmation

    Note over O,A: Step 3 — Liquidity Allocation  
    O->>A: Allocate LP tokens for reserved coverage in AMM Pool  
    A-->>O: Return confirmation and reserve ratio

    Note over O,N: Step 4 — NFT Policy Issuance  
    O->>N: Mint Policy NFT (XLS-0020) with metadata:  
    - PolicyID  
    - Vault reference  
    - Reserve ratio  
    - Expiry and terms  
    N-->>O: TokenID confirmation

    Note over O,D: Step 5 — NFT Delivery  
    O->>D: Transfer Policy NFT (via NFTokenOfferAccept)
    D-->>O: Confirm acceptance (policy active)
    
    Note right of A: AMM reserves updated, liability recorded.

## 2. Claim Validation Flow

```mermaid
sequenceDiagram
    participant D as Depositor (Policy Holder)
    participant O as Operator (Claims Assessor)
    participant V as Vault (XLS-0065)
    participant L as Loan Contract (XLS-0066)
    participant N as Policy NFT (XLS-0020)
    participant A as AMM Pool (Reserves Source)
    participant E as Escrow (Payout Channel)
    participant R as Risk Committee (Off-chain Validator)
    participant X as XRPL Ledger State

    Note over D,O: Step 1 — Claim Submission  
    D->>O: Submit claim request referencing Policy NFT (TokenID)

    Note over O,N: Step 2 — Policy Metadata Fetch  
    O->>N: Query metadata (coverage amount, validity, expiry)  
    N-->>O: Return policy data

    Note over O,L: Step 3 — Loan Default Check  
    O->>L: Verify insured loan state  
    L-->>O: Return lsfLoanDefault = true/false

    Note over O,V: Step 4 — Vault Collateral Validation  
    O->>V: Inspect Vault lock integrity and asset balance  
    V-->>O: Return confirmation of sufficient reserves

    Note over O,R: Step 5 — Risk Committee Attestation  
    O->>R: Submit off-ledger evidence and claim metadata  
    R-->>O: Return attested signature packet

    Note over O,X: Step 6 — Ledger State Audit  
    O->>X: Validate escrow conditions, no duplicate claims  
    X-->>O: Return OK

    Note over O,A: Step 7 — Reserve Availability Check  
    O->>A: Query AMM reserves for payout capacity  
    A-->>O: Yield on-ledger liquidity summary

    Note over O: Step 8 — Claim Approval Flag  
    O->>O: Mark claim state = VALIDATED (internal ledger memo/tag)

    Note over O,E: Step 9 — EscrowCreate Transaction  
    O->>E: Create EscrowCreate for payout  
    E-->>O: 48-hour dispute countdown initiated

## 3. Claim Settlement Flow

```mermaid
sequenceDiagram
    participant O as Operator (Insurer)
    participant E as Escrow (XRPL Payout Channel)
    participant D as Depositor (Claimant)
    participant R as Risk Committee / Dispute Agent
    participant X as XRPL Ledger

    Note over O,E: Step 1 — EscrowCreate
    O->>E: Submit EscrowCreate (amount, destination=D, condition=R attestation)
    E-->>O: Escrow sequence & condition hash recorded

    Note over E: Step 2 — 48-hour Dispute Window
    E-->>R: Notify dispute agents (window open)
    R-->>E: Optionally file EscrowCancel if dispute facts arise
    E-->>O: Timer countdown (T+48h)

    Note over E,X: Step 3 — Ledger Monitoring
    X-->>E: Track Escrow expiration time and dispute state

    alt No Dispute Filed
        Note over E,D: Step 4a — Payout Finalization  
        E->>X: Execute EscrowFinish (conditional release)  
        X-->>D: Funds settled to claimant wallet  
        O-->>X: Mark policy status = “Settled”
    else Dispute Filed
        Note over R,O: Step 4b — EscrowCancel Procedure  
        R->>E: Issue EscrowCancel  
        E-->>O: Funds revert to originating AMM reserve wallet
        O-->>X: Policy marked “Under Review / Disputed”
    end
