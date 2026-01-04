# Analysis of Token Swap Handlers

## Overview

The Drift protocol implements token swaps through a two-phase mechanism using `handle_begin_swap` and `handle_end_swap` instructions. This design enables atomic swaps with external liquidity sources (like Jupiter) while maintaining strict risk controls.

**Location**: `programs/drift/src/instructions/user.rs:3489` (begin_swap) and `:3770` (end_swap)

## Swap Execution Flow

### Phase 1: `handle_begin_swap` (Lines 3489-3759)

This instruction initiates the swap by creating a flash loan-like state:

1. **Validation Checks**:
   - User is not bankrupt (`user.is_bankrupt()`)
   - User is not being liquidated
   - Both markets have fills enabled
   - Flash loan state is clean (no existing flash loans)
   - Markets cannot both have transfer hooks (one maximum)
   - Markets must be different (`in_market_index != out_market_index`)
   - Amount must be non-zero

2. **State Setup**:
   - Records initial token account balances:
     - `in_spot_market.flash_loan_amount = amount_in`
     - `in_spot_market.flash_loan_initial_token_amount = in_token_account.amount`
     - `out_spot_market.flash_loan_initial_token_amount = out_token_account.amount`

3. **Token Transfer**:
   - Transfers `amount_in` tokens from the protocol vault to user's `in_token_account`
   - This gives the user temporary custody of tokens to execute external swap

4. **Instruction Introspection** (Lines 3622-3758):
   - Validates that BeginSwap is a top-level instruction (not a CPI)
   - Scans upcoming instructions to find matching EndSwap
   - Enforces strict account matching between BeginSwap and EndSwap:
     - Same user account
     - Same authority
     - Same vault accounts
     - Same token accounts
   - Ensures EndSwap is the last Drift instruction in the transaction
   - Prevents additional Drift instructions after EndSwap

### Phase 2: `handle_end_swap` (Lines 3770-4159)

This instruction completes the swap and enforces all risk controls:

1. **Exchange Status Check**:
   - Validates deposits and withdrawals are not globally paused

2. **Market Operation Checks**:
   - In-market: Withdraw operation not paused
   - Out-market: Deposit operation not paused
   - Flash loan state must be set (validates BeginSwap occurred)

3. **Token Reconciliation**:
   - **In-market side**:
     - Calculates residual if user returned more than borrowed
     - Receives residual tokens back to vault
     - Updates user's borrow position for `amount_in`

   - **Out-market side**:
     - Calculates how many tokens were received
     - Transfers received tokens to protocol vault
     - Updates user's deposit position for `amount_out`

4. **Risk Control Validations** (detailed below)

5. **Cleanup**:
   - Resets flash loan state to zero
   - Emits SwapRecord event
   - Updates user's last active slot

## Risk Controls

### 1. Limit Price Protection (Lines 3967-3981)

```rust
if let Some(limit_price) = limit_price {
    let swap_price = calculate_swap_price(
        amount_out, amount_in,
        out_spot_market.decimals,
        in_spot_market.decimals
    )?;
    validate!(swap_price >= limit_price)?;
}
```

- User can specify minimum acceptable price
- Prevents slippage beyond user's tolerance
- Price calculated as: `(amount_out * PRICE_PRECISION / 10^out_decimals) * 10^in_decimals / amount_in`

### 2. Price Band Validation (Lines 4147-4157)

```rust
validate_price_bands_for_swap(
    &in_spot_market, &out_spot_market,
    amount_in, amount_out,
    in_oracle_price, out_oracle_price,
    state.oracle_guard_rails.max_oracle_twap_5min_percent_divergence()
)?;
```

**Purpose**: Prevents manipulation through oracle price divergence

**Checks** (from `math/spot_swap.rs:95`):
- Compares swap execution price against oracle prices
- Validates against 5-minute TWAP (Time-Weighted Average Price)
- Uses tighter margin ratios for markets with higher risk
- Prevents trades during oracle manipulation or extreme volatility

### 3. Reduce-Only Mode Enforcement (Lines 3885-3917, 4035-4061)

**SwapReduceOnly Enum**: `In` | `Out`

**In-market checks**:
```rust
if !in_position_is_reduced {
    validate!(!in_spot_market.is_reduce_only())?;
    validate!(reduce_only != Some(SwapReduceOnly::In))?;
    validate!(user.is_margin_trading_enabled)?;
    validate!(!user.is_reduce_only())?;
}
```

**Out-market checks**:
```rust
if !out_position_is_reduced {
    validate!(!out_spot_market.is_reduce_only())?;
    validate!(reduce_only != Some(SwapReduceOnly::Out))?;
    validate!(!user.is_reduce_only())?;
}
```

**Purpose**:
- Allows users to close positions without increasing liabilities
- Market-level reduce-only prevents new borrows/deposits
- User-level reduce-only prevents risky behavior during liquidation risk

### 4. Margin Requirement Validation (Lines 4086-4112)

```rust
let (margin_type, _) = spot_swap::select_margin_type_for_swap(
    &in_spot_market, &out_spot_market,
    &in_strict_price, &out_strict_price,
    in_token_amount_before, out_token_amount_before,
    in_token_amount_after, out_token_amount_after,
    MarginRequirementType::Initial
)?;

user.meets_withdraw_margin_requirement_and_increment_fuel_bonus_swap(
    &perp_market_map, &spot_market_map, &mut oracle_map,
    margin_type, in_market_index, out_market_index, ...
)?;
```

**Purpose**: Ensures user maintains adequate collateral after swap

**Logic**:
- Selects appropriate margin calculation type based on position changes
- Uses `StrictOraclePrice` (oracle + TWAP) for conservative valuation
- Validates user meets initial margin requirements
- Prevents swaps that would make user unhealthy

### 5. Deposit/Borrow Limits (Lines 3874-3879, 4019-4026)

```rust
update_spot_balances_and_cumulative_deposits_with_limits(
    amount_in, &SpotBalanceType::Borrow,
    &mut in_spot_market, &mut user
)?;

update_spot_balances_and_cumulative_deposits(
    amount_out_after_fee, &SpotBalanceType::Deposit,
    &mut out_spot_market, user.force_get_spot_position_mut(out_market_index)?
)?;
```

**Purpose**: Enforces protocol-level and market-level limits

**Checks**:
- Maximum borrow amounts per market
- Maximum deposit amounts per market
- Prevents excessive concentration risk

### 6. Vault Amount Validation (Lines 3920, 4063)

```rust
math::spot_withdraw::validate_spot_market_vault_amount(
    &in_spot_market, in_vault.amount
)?;

math::spot_withdraw::validate_spot_market_vault_amount(
    &out_spot_market, out_vault.amount
)?;
```

**Purpose**: Ensures vault integrity and prevents accounting errors

### 7. User Not Being Liquidated (Lines 3520-3526)

```rust
math::liquidation::validate_user_not_being_liquidated(
    &mut user, &perp_market_map, &spot_market_map,
    &mut oracle_map,
    ctx.accounts.state.liquidation_margin_buffer_ratio
)?;
```

**Purpose**: Prevents users from swapping while under liquidation

### 8. Margin Trading Permission (Lines 3905-3910)

```rust
validate!(
    user.is_margin_trading_enabled,
    ErrorCode::MarginTradingDisabled,
    "swap lead to increase in liability for in market"
)?;
```

**Purpose**: Users must explicitly enable margin trading to incur new liabilities

### 9. Transfer Hook Restrictions (Lines 3556-3560)

```rust
validate!(
    !(in_spot_has_transfer_hook && out_spot_has_transfer_hook),
    ErrorCode::InvalidSwap,
    "both in and out spot markets cannot both have transfer hooks"
)?;
```

**Purpose**: Limits complexity when handling SPL token transfer hooks (Token-2022)

### 10. State Cleanup Validation (Lines 4131-4145)

```rust
validate!(
    out_spot_market.flash_loan_initial_token_amount == 0
        && out_spot_market.flash_loan_amount == 0,
    ErrorCode::InvalidSwap,
    "end_swap ended in invalid state"
)?;
```

**Purpose**: Ensures flash loan state is properly cleaned up

## Security Features

### 1. Atomic Execution
- BeginSwap and EndSwap must occur in the same transaction
- Instruction introspection validates correct pairing
- Prevents incomplete swaps or state corruption

### 2. Reentrancy Protection
- Flash loan amounts stored in market state
- Validates clean state before beginning
- Prevents nested or concurrent swaps on same markets

### 3. CPI Restrictions
- BeginSwap must be top-level instruction (not CPI)
- Prevents malicious programs from initiating swaps
- Protects against complex attack vectors

### 4. Account Validation
- Strict account matching between Begin and End
- Validates vault-to-token account mint relationships
- Ensures authority consistency

### 5. Oracle Guard Rails
- Price band validation against oracle + TWAP
- Configurable divergence thresholds
- Prevents trades during oracle manipulation

## Fee Structure

Currently **zero fees** (Line 3984):
```rust
let fee = 0_u64; // no fee
```

Fee infrastructure exists for future implementation:
- Fee tracking in `out_spot_market.total_swap_fee`
- User stats volume tracking
- Revenue pool allocation ready

## Event Emission

```rust
pub struct SwapRecord {
    pub ts: i64,
    pub user: Pubkey,
    pub amount_out: u64,    // out market mint precision
    pub amount_in: u64,     // in market mint precision
    pub out_market_index: u16,
    pub in_market_index: u16,
    pub out_oracle_price: i64,  // PRICE_PRECISION
    pub in_oracle_price: i64,   // PRICE_PRECISION
    pub fee: u64,
}
```

## Integration Pattern

The swap mechanism integrates with external DEXs (e.g., Jupiter):

1. User calls `begin_swap(in_market, out_market, amount_in)`
2. Protocol sends `amount_in` to user's token account
3. User/integrator executes Jupiter swap instruction(s)
4. Jupiter consumes in-tokens, produces out-tokens
5. User calls `end_swap(in_market, out_market, limit_price, reduce_only)`
6. Protocol validates all risk controls and reconciles balances

## Key Design Decisions

1. **Flash Loan Pattern**: Enables composability with external DEXs
2. **Instruction Introspection**: Ensures atomic execution without complex state management
3. **Dual-Phase Design**: Separates token movement from risk validation
4. **Conservative Pricing**: Uses strict oracle prices (oracle + TWAP) for margin checks
5. **Multi-Layer Validation**: Redundant checks at user, market, and protocol levels

## Potential Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Oracle manipulation | Price band validation, TWAP checks |
| User insolvency | Margin requirement validation, liquidation checks |
| Excessive leverage | Reduce-only modes, margin trading flag |
| Protocol insolvency | Deposit/borrow limits, vault validation |
| Transaction reordering | Atomic begin/end requirement, instruction introspection |
| Reentrancy | Flash loan state checks, clean state validation |
| Unauthorized access | Authority validation, CPI restrictions |

## Summary

The swap handlers implement a robust, composable token swap mechanism with comprehensive risk controls. The design prioritizes safety through:

- **Multi-layered validation**: User, market, and protocol-level checks
- **Conservative pricing**: Strict oracle prices with TWAP divergence limits
- **Atomicity**: Enforced begin/end pairing via instruction introspection
- **Margin safety**: Initial margin requirements prevent undercollateralized positions
- **Flexibility**: Optional reduce-only and limit-price parameters for user control
- **Composability**: Enables integration with external DEXs while maintaining protocol safety

This architecture successfully balances the competing goals of capital efficiency, user flexibility, and protocol security.
