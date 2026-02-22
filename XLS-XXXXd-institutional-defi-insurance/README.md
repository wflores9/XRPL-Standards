<pre>
  xls: XXXX
  title: Institutional DeFi Insurance Protocol
  description: Ecosystem-level insurance layer for XLS-66 Lending Protocol vault depositors, protecting against borrower defaults exceeding First-Loss Capital coverage.
  author: Will Flores <wflores@wardprotocol.org>
  proposal-from: https://github.com/XRPLF/XRPL-Standards/discussions/474
  category: Ecosystem
  requires: XLS-65, XLS-66
  status: Draft
  created: 2026-02-15
  updated: 2026-02-21
</pre>
# Institutional DeFi Insurance Protocol

## 1. Abstract

Ward Protocol provides institutional-grade insurance coverage for vault depositors participating in the XLS-66 Lending Protocol on the XRP Ledger. The protocol enables vault depositors to purchase insurance policies that protect against losses exceeding First-Loss Capital protection when borrowers default.

Ward Protocol operates as an ecosystem-level application using existing XRPL primitives (XLS-30 AMM pools, XLS-20 NFTs, Payment transactions, and Escrow) without requiring protocol amendments. Insurance pools aggregate institutional capital, collect premiums from protected depositors, and automatically process claims triggered by XLS-66 default events that result in vault asset losses.

## 2. Motivation

The XLS-66 Lending Protocol enables fixed-term, fixed-rate loans on the XRP Ledger using pooled liquidity from XLS-65 Single Asset Vaults. While XLS-66 includes First-Loss Capital protection managed by LoanBrokers, vault depositors face significant uninsured risks when defaults exceed this protection.

### 2.1 The Uninsured Loss Gap

When a borrower defaults on an XLS-66 loan, the loss calculation follows:
```
DefaultAmount = Loan.PrincipalOutstanding + Loan.InterestOutstanding
DefaultCovered = min(FirstLossCapital × LiquidationRate, DefaultAmount)
VaultLoss = DefaultAmount - DefaultCovered
```

Example: A $55,000 default with $1,000 first-loss coverage leaves vault depositors with $54,000 in uninsured losses, distributed proportionally across all vault share holders via reduced `Vault.AssetsTotal`.

### 2.2 Institutional Barriers

This uninsured gap creates barriers for institutional participation:

1. Regulatory capital requirements prevent allocation to uninsured positions
2. Risk exposure exceeds acceptable limits for fiduciary mandates
3. No mechanism exists to transfer tail risk to willing counterparties
4. Vault share devaluation from `Vault.LossUnrealized` creates mark-to-market risk

## 3. Specification

### 3.1 Design Rationale: Ecosystem Layer

Ward Protocol operates entirely at the ecosystem layer rather than proposing protocol amendments:

- **Rapid deployment**: Production-ready in weeks, not the months/years required for amendments
- **Market validation**: Prove institutional demand before requesting protocol changes
- **Iteration speed**: Bug fixes and feature additions without consensus requirements
- **Future upgrade path**: Successful adoption informs a potential native amendment (XLS-104)

### 3.2 XRPL Primitives Used

| Primitive | Usage |
|-----------|-------|
| XLS-20 NFTs | Policy certificates (transferable, with metadata) |
| XLS-30 AMM | Insurance pool capital aggregation |
| XLS-65 Vaults | Monitor depositor positions and share values |
| XLS-66 Loans | Detect defaults, calculate losses |
| XLS-80 Permissioned Domains | Institutional compliance and access control |
| XLS-70 Credentials | Credential verification for domain membership |
| Escrow | Time-locked claim settlements (48-hour dispute window) |

### 3.3 Core Components

#### 3.3.1 Insurance Pool Management

Insurance pools aggregate capital using XLS-30 AMM pools:

- **Pool Asset**: XRP or RLUSD (matches the XLS-66 vault asset)
- **Liquidity Providers**: Institutional capital allocators seeking premium yield
- **Pool Shares**: AMM LP tokens representing proportional ownership
- **Capital Requirements**: Minimum 200% coverage ratio enforced programmatically

#### 3.3.2 Policy Issuance

Policies are minted as XLS-20 NFTs with structured metadata:
```json
{
  "protocol": "ward-v1",
  "vault_id": "2DE64CA4...",
  "coverage_amount": "50000000000",
  "coverage_start": "2026-02-15T00:00:00Z",
  "coverage_end": "2026-05-15T00:00:00Z",
  "pool_id": "3F4A5B6C..."
}
```

Policy NFTs are fully transferable, enabling secondary market trading and portfolio management.

#### 3.3.3 Premium Calculation
```
premium = coverage × base_rate × term_factor × risk_multiplier

Where:
  base_rate: 1-5% annually (risk-tiered)
  risk_multiplier: 0.5x - 2.0x based on:
    - Vault utilization rate
    - First-loss coverage ratio
    - Impairment ratio (LossUnrealized / AssetsTotal)
    - Historical default rate
```

#### 3.3.4 Default Monitoring

Real-time monitoring of XLS-66 loan state:

- WebSocket subscription to `LoanManage` transactions
- Parse `tfLoanDefault` flag (0x00010000)
- Track `Vault.AssetsTotal` changes post-default
- Calculate per-share loss impact

#### 3.3.5 Claim Validation (9 Steps)

1. Verify loan defaulted (`lsfLoanDefault` flag on ledger)
2. Calculate vault loss using XLS-66 formulas
3. Fetch policy from Ward database
4. Verify policy status is active
5. Check claim is within coverage window
6. Verify vault matches policy vault_id
7. Calculate payout: `min(depositor_loss, coverage_amount)`
8. Verify pool capital adequacy for payout
9. Approve/reject with full audit log

#### 3.3.6 Escrow Settlement

Approved claims create XRPL Escrow transactions:

- 48-hour hold period for dispute resolution
- Multi-sig (3-of-5) can cancel fraudulent claims during hold
- Automatic release after hold period expires
- Full on-chain audit trail

### 3.4 Permissioned Domain Integration (XLS-80)

Ward Protocol uses XLS-80 Permissioned Domains for institutional access control:

- **Domain Registration**: Ward operates a permissioned domain requiring XLS-70 credentials
- **Credential Types**: KYC verification, accredited investor status, institutional mandate
- **Access Tiers**: Public (view), Authenticated (purchase policies), Admin (pool management)

### 3.5 API Specification

Production API available at `https://api.wardprotocol.org` with OpenAPI documentation at `/docs`.

**Endpoint Categories:**
- **Public**: Health checks, pool statistics, protocol information
- **Permissioned Domains**: Domain creation, credential verification, membership management
- **Admin**: Pool configuration, claim processing, system monitoring

All authenticated endpoints require API key via `X-API-Key` header with tiered rate limiting.

## 4. Security Considerations

1. **Oracle Risk**: Multiple independent monitoring nodes with 3-of-5 consensus required for claim approval
2. **Capital Adequacy**: 200% minimum coverage ratio enforced; pools cannot issue policies beyond capacity
3. **Key Management**: Multi-sig (3-of-5) institutional signers; cold storage for reserve capital
4. **Claim Validation**: 48-hour escrow provides dispute window before settlement
5. **No Custom Smart Contracts**: Uses only native XRPL transaction types, eliminating smart contract risk
6. **Infrastructure Security**: Production VPS hardened with UFW firewall, Fail2ban intrusion prevention, SSH key-only auth, automated PostgreSQL backups

## 5. Reference Implementation

**Repository**: https://github.com/wflores9/ward-protocol

**Stack**:
- Python 3.12 with xrpl-py SDK
- FastAPI + Uvicorn (API server)
- PostgreSQL 16 (policy and claims database)
- Nginx with SSL/TLS (reverse proxy)

**Current Status**:
- Production API live at api.wardprotocol.org
- XLS-80 Permissioned Domain registered on testnet
- 60 automated tests, 75% code coverage
- OpenAPI documentation at /docs and /redoc
- Hardened production infrastructure with automated backups

## 6. Future Upgrade Path: XLS-104

If Ward Protocol achieves product-market fit, a future amendment proposal could introduce:

- Native `InsurancePolicy` ledger object type
- `PolicyClaim` transaction type
- Automated claim settlement hooks integrated with XLS-66
- Reduced operational costs through native ledger support

This ecosystem-first approach validates demand and refines the specification before proposing protocol-level changes.

## 7. Discussion

Community feedback is requested on:

1. Are there other XRPL lending protocols Ward should design compatibility for?
2. Would native insurance primitives be valuable for use cases beyond lending?
3. Should we coordinate with XLS-66 implementers for joint testnet demonstration?
4. Is the 48-hour escrow period appropriate, or should it be configurable?

---

**Contact:**
- GitHub: [@wflores9](https://github.com/wflores9)
- Repository: https://github.com/wflores9/ward-protocol
- API: https://api.wardprotocol.org
- Email: wflores@wardprotocol.org
