---
name: zerion-lending
description: >
  Borrow tokens against crypto collateral on Aave v3 and Morpho using Zerion CLI for portfolio checks
  and position monitoring. Covers the full lifecycle: research rates, supply collateral, borrow,
  monitor health factor, repay, and withdraw. Use when the user asks to borrow, take a loan,
  leverage a position, get liquidity without selling, or manage an existing debt position on-chain.
license: MIT
allowed-tools: Bash
---

# Zerion — Collateralized Lending

**Purpose:** Enable borrowing against crypto collateral on Aave v3 (multi-chain) and Morpho, with Zerion CLI providing portfolio state and position tracking. Covers rate research → supply collateral → borrow → health monitoring → repay → withdraw.

## When to use

- "Borrow USDC against my ETH"
- "Take a loan on Aave without selling my tokens"
- "How much can I borrow against my WBTC?"
- "What's the current borrow rate for [token] on [protocol]?"
- "My health factor is low — help me add collateral or repay"
- "Repay my Aave loan" / "Withdraw my collateral after repaying"
- Any request to borrow, leverage, take a loan, manage debt, or interact with a lending protocol on-chain

## Requirements

- Zerion CLI: `npm install -g zerion-cli`
- Zerion API key: `export ZERION_API_KEY="zk_..."`
- A funded wallet with assets to use as collateral

## Default protocol

Default to **Aave v3** unless the user specifies otherwise or a rate comparison clearly favors another supported protocol. See [`reference/`](./reference/) for full protocol and network coverage.

## Core commands

**Zerion CLI (read-only state):**
```bash
zerion portfolio <address>                # balances across all chains
zerion positions <address> --defi         # lending positions grouped by protocol
zerion analyze <address>                  # full wallet overview
zerion swap <chain> <amount> <from> <to>  # acquire borrow or repay asset
zerion bridge <from-chain> <token> <amount> <to-chain> <token> --cheapest
```

**Aave v3 Pool — read:**
- `getUserAccountData(address)` → collateral, debt, available borrows, LTV, health factor
- `getReserveData(asset)` → liquidity rate, variable borrow rate, LTV, liquidation threshold, caps
- `getReservesList()` → all listed assets on that deployment

**Aave v3 Pool — write (build unsigned, never broadcast):**
- `supply(asset, amount, onBehalfOf, referralCode)`
- `borrow(asset, amount, interestRateMode, referralCode, onBehalfOf)` — mode `2` = variable
- `repay(asset, amount, rateMode, onBehalfOf)` — pass `MaxUint256` to clear dust
- `withdraw(asset, amount, to)` — pass `MaxUint256` to withdraw everything
- `setUserEMode(categoryId)` — efficiency mode for correlated assets

Full ABI details and Pool addresses per chain: [`reference/aave-v3/`](./reference/aave-v3/).

## Risk parameters

| Term | Definition |
|------|-----------|
| LTV | Max borrow as % of collateral value |
| Liquidation threshold | LTV at which position is liquidated (always > LTV) |
| Health factor | `(Σ collateral × liquidation_threshold) / total_debt`. Must stay > 1.0 |
| Variable borrow rate | Tracks pool utilization; default mode |
| Liquidation bonus | Extra % the liquidator claims on liquidation |

**Target health factor: ≥ 1.5** — leaves ~33% buffer before liquidation. Full definitions in [`docs/risk-parameters.md`](./docs/risk-parameters.md).

## Safe borrow calculation

Given collateral USD value `C`, liquidation threshold `LT`, target health factor `HF`:

```
max_borrow_USD = (C × LT) / HF
```

**Example:** 10 ETH @ $2,000 = $20,000 collateral, LT = 0.825, target HF = 1.5
→ `max_borrow_USD = (20,000 × 0.825) / 1.5 = $11,000`

Always show this calculation explicitly. If the user requests more than the safe maximum, warn and recommend the calculated amount.

## Lifecycle

The full 10-step workflow lives in [`workflows/`](./workflows/). At a glance:

1. **Check portfolio** — `zerion portfolio` + `zerion positions --defi`. Skip ahead if a position already exists.
2. **Identify intent** — collateral asset, borrow asset, chain/protocol, amount (or "safe max").
3. **Fetch rates** — `getReserveData` for both assets; surface a comparison table.
4. **Calculate safe borrow** — apply the formula above; confirm amount with user.
5. **Supply collateral** — approve (if needed) → `supply`. Present decoded calls; do not sign.
6. **Borrow** — `borrow` with mode `2`. Show projected HF, daily borrow cost, net carry.
7. **Verify position** — re-run `zerion positions --defi` + `getUserAccountData`.
8. **Add collateral** (when HF drops or user wants more borrow) — swap/bridge if needed, then `supply`.
9. **Repay debt** — acquire borrow asset if short, approve, `repay(MaxUint256)` to clear dust.
10. **Withdraw collateral** — `withdraw(MaxUint256, user)` after debt is fully cleared.

## Non-negotiables

- **Never sign or broadcast transactions.** Build unsigned calls, decode them clearly, and wait for the user to sign in their own wallet.
- **Always show the safe-borrow math** before presenting a borrow transaction. The user confirms the amount before any tx is built.
- **Verify decimals per token** when constructing amounts in base units (USDC = 6, WETH = 18, WBTC = 8). Wrong decimals = wrong-by-orders-of-magnitude transactions.
- **Check ERC-20 allowance before `supply` and `repay`.** Insert an `approve` call if `allowance < amount`.
- **Use `MaxUint256` for full repay and full withdraw.** Exact amounts leave dust from accrued interest and can leave the position open.
- **Surface health factor before and after any state-changing action** so the user sees the risk delta, not just the outcome.

## Examples

End-to-end walkthroughs in [`examples/`](./examples/):
- `borrow-usdc-against-eth.md`
- `add-collateral-low-hf.md`
- `repay-and-withdraw.md`
- `cross-chain-collateral.md`
