# Loanminting Protocol – Technical Architecture Overview

**Version:** ARCHITECTURE_SPEC_V0.1
*(Living document – this specification is intended to evolve as the system design matures)*

---

## 1. Introduction

The Loanminting Protocol enables **fixed-duration, non-liquidating crypto loans** backed by tokenized collateral and pooled lender liquidity.  
Unlike health-factor–based lending protocols, Loanminting defines **immutable loan terms** at creation (duration, LTV, APR) and manages them through isolated on-chain loan vaults.

This document outlines the **smart contract architecture**, roles, and system flows required to implement the protocol securely, upgradeably, and composably.

---

## 2. Design Principles

1. **Immutable terms, dynamic risk:**  
   Each loan’s terms are fixed at origination; risk parameters can evolve for future loans via configurable pricing contracts.

2. **Composable, tokenized loans:**  
   Every loan is a self-contained vault represented as an NFT (ERC-721) and fractionally ownable via ERC-20 claim tokens.

3. **Fixed-term, non-liquidating structure:**  
   All loans are **fixed-duration instruments** with **no intra-period liquidations**.  
   The loan lifecycle is purely time-based (`block.timestamp`)—default and collateral seizure can occur only at or after maturity (subject to any grace period).  
   No health factors or price-based triggers affect loans during their active term.

4. **Modular extensibility:**  
   Pricing, liquidation, and rollover logic are implemented as replaceable modules without altering live loans.

5. **Data validation and circuit breakers:**  
   NAV and pricing calculations assume accurate, validated data. If data becomes unreliable or price movements exceed defined thresholds, **circuit breakers** are triggered to pause minting, redemption, and new loan origination until confidence is restored.

6. **Grace and fairness:**  
   Borrowers receive a configurable grace period with incremental late penalties before collateral is seized.

---

## 3. System Overview

| Layer | Contract | Core Function |
|-------|-----------|---------------|
| Loan Layer | **LoanVault** | Custodies collateral and loan state |
| Factory Layer | **LoanFactory** | Originates and configures new loans |
| Pool Layer | **LiquidityPool** | Holds lender deposits and loan shares |
| Pricing Layer | **PricingRouter** | Computes APR and loan NAV based on validated market inputs |
| Oracle Layer | **PriceOracle**, **VolatilityOracle** | Provides validated asset and volatility data |
| Liquidation Layer | **LiquidationModule** | Handles collateral settlement (pluggable; protocol or third-party executors) |
| Rollover Layer (optional) | **RolloverRouter** | Enables refinancing for matured loans |

Each module is **replaceable**, **versioned**, and designed for clean upgrades without breaking live positions.

---

## 4. Contract Architecture

### 4.1 LoanVault (per-loan contract)

**Purpose:**  
Represent a single loan instance with its own collateral, borrower, and lender share distribution.

**Representation:**  
- ERC-721 token = loan identity (unique vault)  
- ERC-20 claim tokens = fractional ownership of repayment streams

**Immutable parameters:**
```solidity
enum RateType { FIXED, FLOATING }

struct LoanTerms {
    address borrower;
    address collateralAsset;
    uint256 collateralAmount;
    address borrowAsset;
    uint256 principal;
    uint256 startTimestamp;
    uint256 maturityTimestamp;
    RateType rateType;
    address rateProvider;   // IRateProvider
    uint256 gracePeriod;
    uint256 maxLatePenalty;
    uint8 liquidationMode;  // 0: Seizure, 1: Partial, 2: Rollover
}
```

**Core functions:**
- `repay(uint256 amount)` – borrower repayment and interest accrual update  
- `reassignBorrower(address newBorrower)` – borrower may reassign the loan prior to default  
- `default()` – callable after grace expiry; triggers liquidation module  
- `claim()` – lenders claim principal and interest  
- `navQuote()` – returns NAV priced via `PricingRouter`  

**Borrower reassignment motivation:**  
Borrowers can transfer their repayment obligation and right to reclaim collateral. This supports **secondary market trading** of loan positions, particularly attractive for **non-callable loans** (where early repayment doesn’t stop interest accrual).  
Only the borrower may trigger reassignment, and only before default.

**Rate query helpers:**  
```solidity
function currentAPR() external view returns (uint256);
function aprAt(uint256 timestamp) external view returns (uint256);
function twaAPR(uint256 start, uint256 end) external view returns (uint256);
```
These delegate to the `IRateProvider` contract.

**Lifecycle states:**
```
Active → (Repaid on time) → Settled
Active → (Past maturity; grace applies) → Repaid → Settled
Active → (Grace expired; unpaid) → Defaulted
```

---

### 4.2 IRateProvider

**Purpose:**  
Defines the interface for rate data sources used by floating or fixed loans.

```solidity
interface IRateProvider {
    function currentAPR(address loanVault) external view returns (uint256 aprWad);
    function aprAt(address loanVault, uint256 timestamp) external view returns (uint256 aprWad);
    function twaAPR(address loanVault, uint256 start, uint256 end) external view returns (uint256 aprWad);
}
```

**Variants:**
- **FixedRateProvider:** returns a constant APR for all queries.  
- **EpochRateProvider:** stores timestamped `RateSegment` intervals with start, end, and APR. Provides rate continuity across epochs.

---

### 4.3 LoanFactory

**Purpose:**  
Originates and validates new loans.

**Responsibilities:**
- Validate parameters (LTV, duration, rate type, collateral, etc.)  
- Compute loan pricing via `PricingRouter`  
- Pull borrower collateral and deploy new `LoanVault` (ERC-1167 minimal proxy)  
- Transfer principal from `LiquidityPool` to borrower  
- Register loan for indexing and tracking  

**createLoan:**
```solidity
createLoan(
    address collateralAsset,
    uint256 collateralAmount,
    address borrowAsset,
    uint256 principal,
    uint256 duration,
    RateType rateType,
    bytes rateParams
)
```
Loans are created only if all parameters are valid and the system is not paused.

---

### 4.4 LiquidityPool

**Purpose:**  
Holds lender deposits and manages exposure to underlying loans.

**Implements:**  
`INavAsset` – standardized NAV interface for composable pooling.  
```solidity
interface INavAsset {
    function nav() external view returns (uint256 valueInBorrowAsset, uint256 lastUpdated);
}
```

**Why INavAsset:**  
- Enables meta-pools that can hold other pools or individual loans seamlessly.  
- Simplifies accounting by using uniform NAV reporting.  
- Encourages extensibility: any future yield-bearing or structured product can integrate by implementing this interface.

**NAV calculation:**  
```
poolNAV = liquidAssets + Σ (loanClaimBalance_i * loanNAV_i)
```

**Key features:**
- Epoch-based mint/redeem to prevent front-running during volatility  
- Circuit breakers pause mint/redeem when volatility or price moves breach limits  
- May hold other pools as composable assets  

**Mint / Redeem:**  
- `deposit(amount)` → mints pool shares at NAV  
- `redeem(shares)` → burns shares and returns proportional liquidity  

---

### 4.5 PricingRouter

**Purpose:**  
Centralized pricing and valuation logic for both loan creation and post-origination NAV updates.

**Behavior:**  
Computes APR based on inputs (LTV, duration, volatility) using a configurable model. Current model is placeholder; final model likely draws from **option-based term structures**.

**Functions:**
```solidity
function quoteAPR(LTV, duration, vol, otherInputs) external view returns (uint256);
function loanNAV(address loanVault, MarketInputs m, LoanState s)
    external view returns (uint256 navInBorrowAsset);
```

### Conceptual NAV model

Loan NAV represents the **present value of the loan instrument**, which can be understood as a **short put option** written by the lender on the collateral.

The **borrower** effectively holds a **European put option** on their collateral, with a **strike price equal to the total debt due at maturity (principal + accrued interest + fees) divided by the amount of collateral posted**.  
If the collateral’s value falls below this strike, the borrower’s optimal move is to default and surrender the collateral—just as a put holder would exercise their option.

For example, suppose a borrower pledges **5 ETH** as collateral, initially worth **$4,000 per ETH** (total $20,000), and borrows **$15,000** for a **6‑month** term at **10% APR**.  
At maturity, the borrower owes:

```
Debt due = 15,000 × (1 + 0.10 × 0.5) = 15,750
```

The corresponding **strike price** is:

```
K = 15,750 ÷ 5 = 3,150 USD per ETH
```

If ETH falls below **$3,150**, the borrower is better off defaulting and letting lenders seize the collateral.  
If ETH stays above that level, the borrower repays and reclaims it.

From the lender’s perspective, the loan’s value at any point is the **present value of the debt owed minus the value of the borrower’s embedded put option**:

```
loanNAV ≈ PV(DebtDue) − BorrowerPutValue(P_market, LTV, σ, τ)
```

Where:
- `PV(DebtDue)` is the discounted value of principal + interest due at maturity (discounted at the risk‑free rate)  
- `BorrowerPutValue` reflects the probability‑weighted loss from default, increasing with higher volatility (σ) or lower collateral prices (P_market)  
- `τ` is the remaining time to maturity  

**Intuition:**  
- If collateral value collapses → the put becomes very valuable → loanNAV falls toward 0.  
- If collateral rallies → the put’s value approaches 0 → loanNAV approaches the full debt value.  

As prices, volatility, and time evolve, the option’s value updates continuously, ensuring NAV always reflects the current market risk of the loan.


**Circuit breakers:**  
Triggered if:  
- Price deviation exceeds threshold over target interval  
- Volatility change exceeds allowed delta  
- Oracle data is stale or inconsistent

When triggered, all minting, redemption, and new loan creation pause until reset by admin or automatic cooldown.

---

### 4.6 Oracles

**Purpose:**  
Provide validated market data for pricing and NAV calculation.

**Design:**  
- Use reliable oracle provider (e.g., Chainlink or RedStone).  
- Validate feed freshness and deviation.  
- Circuit breakers activate on invalid or extreme data events.  
- When circuit breakers are active dependent functions (pricing, minting, origination) are paused. Repay, default(), claim()/settlement remain available.

**Components:**
- **PriceOracle:** Asset spot prices.  
- **VolatilityOracle:** Rolling realized or implied vol.  
- **RateFeeder (optional):** Supplies epoch rate updates to `EpochRateProvider`.

---

### 4.7 LiquidationModule

**Purpose:**  
Handle collateral disposition when a loan defaults.

**Modes:**
1. **Partial Settlement (default):** Sell enough collateral to cover obligation (with penalty); return remainder to borrower.  
2. **Pure Seizure:** Transfer all collateral to lenders (simpler fallback).  
3. **Auto-Rollover:** Refinance under new terms if liquidity allows.

**Flow:**
```
Collateral (or proceeds)
 → LoanVault
 → Claim token holders
 → LiquidityPool
 → Lenders
```

---

### 4.8 RolloverRouter (optional)

**Purpose:**  
Refinance loans automatically at maturity where eligible.

**Flow:**  
1. Check borrower preference, liquidity, and risk parameters.  
2. Call `LoanFactory.createLoan()` for replacement.  
3. Repay old loan atomically.  
4. Transfer collateral to new vault.  

---

## 5. Roles & Permissions

| Role | Capabilities |
|------|---------------|
| `ADMIN` | Configure parameters, upgrade contracts, and manually toggle circuit breakers |
| `RISK_MANAGER` | Update pricing constants and risk parameters |
| `ORACLE_UPDATER` | Post signed oracle data (if applicable) |
| `FACTORY` | Authorized to originate loans |
| `KEEPER` | Trigger default or settlement actions |

Automatic circuit breakers govern most safety events. Manual controls provide final authority.

---

## 6. NAV and Risk Controls

### Loan NAV (via PricingRouter)

Loan NAV represents the **mark-to-market fair value** of each loan, computed by the `PricingRouter` using an option-based valuation model.

```
loanNAV ≈ PV(DebtDue) − BorrowerPutValue(P_market, σ, τ)
```

- **PV(DebtDue)** — present value of principal plus accrued interest due at maturity, discounted at the risk‑free rate.  
- **BorrowerPutValue** — value of the borrower’s embedded put option on their collateral, which rises with higher volatility (σ), lower market prices (P_market), or longer remaining time to maturity (τ).  
- Loan NAV decreases as collateral value falls or volatility increases, and rises as risk subsides.

The `PricingRouter` continuously re‑marks each loan based on validated market inputs and circuit‑breaker conditions to ensure NAV reflects the current market risk of the loan.


### Pool NAV (LiquidityPool)

```
poolNAV = liquidAssets + Σ (poolExposure_i) 
where poolExposure_i = (pool’s pro-rata share in loan i) × loanNAV_i
```

Pool NAV aggregates all active loan valuations and liquid assets held in the pool, providing the total market value of lender exposure.  
When circuit breakers are triggered due to data anomalies or extreme volatility, NAV-dependent operations (minting, redemption, and loan origination) are temporarily paused until stability and data confidence are restored.

### Circuit Breaker Policy
Operations are paused when:  
- Oracle data is stale or missing  
- Price or volatility changes exceed predefined thresholds  
- Manual override triggered by admin  

When circuit breakers are active, minting, redemption, and origination halt until revalidation. Repay, default(), claim()/settlement remain available.
---

## 7. System Flow Summary

### Borrower Flow
1. Request quote via frontend → PricingRouter  
2. Approve collateral → LoanFactory  
3. Factory deploys `LoanVault` and transfers principal  
4. Loan runs to maturity → repay, roll over, or default  

### Lender Flow
1. Deposit stablecoins into LiquidityPool  
2. Receive ERC-4626 pool shares  
3. Accrue yield as loans repay  
4. Shares and claim tokens tradable on AMMs for instant liquidity  
5. Redeem shares for proportional liquidity  

### Default Flow
1. Loan past maturity + grace → any caller triggers `default()`  
2. LiquidationModule resolves collateral  
3. Proceeds distributed to lenders via pool  

---

## 8. Upgradeability & Extensibility

| Component | Upgradeable? | Notes |
|------------|---------------|-------|
| LoanVault | ❌ | Immutable minimal proxy |
| LoanFactory | ✅ | UUPS or Proxy |
| LiquidityPool | ✅ | ERC-4626 compatible upgrades |
| PricingRouter | ✅ | Replaceable pricing logic |
| Oracles | ✅ | Swappable providers |
| LiquidationModule | ✅ | Strategy plug-in |
| RolloverRouter | ✅ | Optional module |

---

## 9. Event Model

- `LoanCreated`, `LoanRepaid`, `Defaulted`, `Settled`  
- `BorrowerReassigned`, `CollateralSeized`, `CollateralReturned`  
- `PoolMinted`, `PoolRedeemed`, `NavUpdated`  
- `SystemPaused`, `SystemUnpaused`  

---

## 10. Security & Audit Focus

- Collateral custody and transfer logic  
- Loan valuation accuracy and circuit breaker enforcement  
- State transitions (Active → Defaulted → Settled)  
- Reentrancy prevention across repayment and liquidation  
- Upgradeability and admin permissions  
- Pool accounting integrity  
- Oracle freshness validation  

---

## 11. Summary

Loanminting’s architecture provides a modular, composable framework for fixed-duration, non-liquidating crypto loans.  
- **LoanVaults** manage borrower logic and loan states.  
- **Factories** handle origination and parameter validation.  
- **LiquidityPools** hold and value lender exposure.  
- **PricingRouter** provides dynamic, data-driven loan and pool valuation.  
- **Circuit breakers** ensure stability under abnormal conditions.  

Together, these components form a robust, upgradeable system designed for transparency, safety, and future extensibility.
