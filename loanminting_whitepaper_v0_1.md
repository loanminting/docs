# Loanminting: Fixed-Duration, Non-Liquidating Crypto Lending

## Overview

Loanminting introduces a new model for decentralized lending that replaces the traditional health factor driven, liquidation-based approach with **fixed-duration, non-liquidating loans**. Borrowers post collateral for a defined period (e.g., 2 weeks, 1 month) and are only required to repay by the end of that period. During the loan term, their collateral cannot be seized due to short-term price fluctuations.

This model removes the need for health factors, liquidation bots, and continuous collateral monitoring—simplifying the borrower experience while introducing new yield opportunities for lenders.

---

## Motivation

### Problems with Traditional DeFi Lending
- **Forced Liquidations:** Borrowers can lose positions in flash crashes or oracle errors.
- **Static Lending Design:** Traditional DeFi lending systems like Aave or Compound don't allow loans themselves to become composable assets. Loanmint's design allows **tokenized loan contracts** that can be traded, fractionalized, or used as collateral in other protocols, unlocking new derivative and structured credit markets.
- **Poor UX:** Health factors and dynamic liquidation thresholds confuse non-technical users.

### The Loanmint Solution
- **Predictability:** Borrowers know exactly when repayment is due and cannot be liquidated mid-term.
- **Fair Risk Transfer:** Lenders take on time-limited exposure in exchange for higher yield.
- **Simplicity:** Borrowing feels like taking a short-term loan or issuing a bond, not managing a volatile position.

> “Aave meets Treasury bills — fixed-term, instant loans with predictable yield and no liquidation risk.”

---

## Core Mechanism

### Borrower Experience
1. Deposit collateral (e.g., ETH, wBTC, FRAX).
2. Choose loan term and LTV (e.g., 14 days at 50%).
3. Receive stablecoins instantly.
4. Repay principal + interest by maturity to reclaim collateral.

If the borrower fails to repay, the collateral is seized and auctioned.

### Lender Experience
1. Deposit stablecoins into a **lending pool**.
2. Receive pool shares representing proportional ownership of pool assets.
3. Earn interest from active loan repayments and baseline yield on idle assets.

Lenders can enter and exit pools via mint/redeem functions, which dynamically adjust share value to reflect active loan performance.

---

## Pooled Liquidity Architecture

Loanminting uses a **vault-of-vaults** model:

- The **primary vault** holds liquid stablecoins and shares of individual **loan sub-vaults**.
- Each loan sub-vault represents one borrower’s loan and holds collateral + repayment terms.
- The primary vault’s Net Asset Value (NAV) = value of stablecoins + time-adjusted value of outstanding loans.

Lenders can select pools by asset type (e.g., ETH, wBTC) or participate in a global meta-pool aggregating multiple asset classes.

---

## Pricing and Risk Modeling

Loan pricing depends on four primary parameters:

| Variable | Description |
|-----------|--------------|
| **LTV** | Loan-to-value ratio. Higher = riskier. |
| **Term Length** | Shorter = less uncertainty. |
| **Volatility (σ)** | Realized or implied volatility of collateral. |
| **Base Yield** | Baseline market lending rate (e.g., Aave deposit yield). |

A simplified model:

\[ r = r_{base} + k \times \sigma \times \sqrt{t} \times (LTV / LTV_{max})^\alpha \]

Where:
- `r_base` = baseline yield from stablecoin deposit protocol
- `t` = term length in years
- `k` and `α` = empirically tuned parameters

This structure mimics option pricing logic — the lender effectively sells a short-term put on the collateral’s value.

---

## aToken Integration: Yield Layer

To optimize capital efficiency, Loanminting can denominate deposits in **Aave aTokens** (e.g., aUSDC):

- Lenders deposit USDC → converted to aUSDC → earn baseline yield automatically.
- Borrowers receive unwrapped USDC but owe back aUSDC plus Loanmint’s premium.

This creates an elegant spread model:
- Borrower pays roughly the same as Aave’s borrow rate.
- Lender earns higher than Aave’s lend rate.
- Platform captures the differential spread.

This design bootstraps liquidity while maintaining composability with major DeFi protocols.

---

## Risk Management

### Collateral Defaults
- If repayment is missed, collateral is seized and either auctioned or sold directly.
- Sale proceeds are distributed to pool depositors pro-rata.

### NAV Adjustment
- Continuous revaluation of outstanding loans based on term remaining and volatility.
- Late entrants buy pool shares at updated NAV, reducing arbitrage risk.

### Entry/Exit Controls
- Redemption batching or time-based epochs can mitigate valuation asymmetry.
- Optional entry/exit fees to discourage short-term speculative inflows.

---

## Extensions & Future Features

1. **Tradable Loan Tokens:** Each loan is represented as a token (ERC-721 or ERC-1155) that can be traded, fractionalized, or used as collateral in other protocols. This enables new layers of composability, including structured credit products, secondary loan markets, and yield derivatives.
2. **Yield Tranching:** Senior/junior pools for different risk appetites.
3. **Synthetic Credit Derivatives:** CDS, interest rate swaps, and futures based on aggregated loan risk.
4. **Secondary Loan Markets:** AMM-style trading for loan tokens, enabling real-time liquidity before maturity.
5. **Volatility-Linked Pricing:** Dynamic APR adjustment based on real-time volatility metrics.
6. **Credit Reputation Layer:** On-chain borrower history and credit scoring based on past repayments.
7. **Institutional Pools:** KYC-gated vaults for compliant lending and regulatory compatibility.

---

## Summary

Loanminting aims to bring predictability and simplicity to DeFi lending by eliminating health-factor liquidations and introducing fixed-duration, risk-priced loans. It bridges the gap between traditional finance’s term lending and DeFi’s permissionless, composable liquidity.

**Key innovations:**
- Fixed-term, non-liquidating loans.
- Nested vault architecture for composable risk exposure.
- Dynamic NAV and volatility-aware pricing.
- Aave-integrated yield structure.
- Tokenized loan contracts as credit building blocks for on-chain derivatives and structured markets.

By aligning incentives between borrowers and lenders around time-based risk, Loanminting introduces a new class of crypto-native credit markets — predictable, flexible, and fair.

