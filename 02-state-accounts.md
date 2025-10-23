# State Accounts Reference

This document describes all the on-chain account structures used by the Dynamic Bonding Curve program.

## Account Overview

| Account | Size (bytes) | Purpose |
|---------|--------------|---------|
| PoolConfig | 1040 | Pool configuration and curve definition |
| VirtualPool | 416 | Active pool state and reserves |
| PartnerMetadata | Variable | Partner information |
| VirtualPoolMetadata | Variable | Pool metadata |
| ClaimFeeOperator | Variable | Authorized fee claimer |

---

## 1. PoolConfig

**Size**: 1040 bytes (zero-copy)

**PDA Seeds**: `["config", config_index]`

The PoolConfig contains all immutable configuration for a bonding curve pool.

### Structure

```rust
pub struct PoolConfig {
    // Core identifiers
    pub quote_mint: Pubkey,           // Quote token mint (e.g., USDC)
    pub fee_claimer: Pubkey,          // Address to receive fees
    pub leftover_receiver: Pubkey,    // Address for leftover base tokens

    // Fee configuration (128 bytes)
    pub pool_fees: PoolFeesConfig,

    // Configuration flags
    pub collect_fee_mode: u8,         // QuoteToken(0) or OutputToken(1)
    pub migration_option: u8,         // MeteoraDamm(0) or DammV2(1)
    pub activation_type: u8,          // Slot(0) or Timestamp(1)
    pub token_decimal: u8,            // Base token decimals
    pub version: u8,                  // Config version
    pub token_type: u8,               // SplToken(0) or Token2022(1)
    pub quote_token_flag: u8,         // Quote token identifier
    pub partner_locked_lp_percentage: u8,  // % of LP locked for partner
    pub partner_lp_percentage: u8,         // % of LP for partner
    pub creator_locked_lp_percentage: u8,  // % of LP locked for creator
    pub creator_lp_percentage: u8,         // % of LP for creator
    pub migration_fee_option: u8,     // Migration fee tier
    pub fixed_token_supply_flag: u8,  // 0=dynamic, 1=fixed supply
    pub creator_trading_fee_percentage: u8, // Creator's share of trading fees
    pub token_update_authority: u8,   // Token authority option
    pub migration_fee_percentage: u8,      // Migration fee %
    pub creator_migration_fee_percentage: u8, // Creator's migration fee %

    // Pool parameters
    pub swap_base_amount: u64,        // Base tokens available for swaps
    pub migration_quote_threshold: u64, // Quote amount to complete curve
    pub migration_base_threshold: u64,  // Base tokens for migration
    pub migration_sqrt_price: u128,   // Target price for partial fills

    // Vesting configuration (48 bytes)
    pub locked_vesting_config: LockedVestingConfig,

    // Token supply (for fixed supply tokens)
    pub pre_migration_token_supply: u64,
    pub post_migration_token_supply: u64,

    // Migrated pool configuration
    pub migrated_collect_fee_mode: u8,
    pub migrated_dynamic_fee: u8,
    pub migrated_pool_fee_bps: u16,

    // Price bounds
    pub sqrt_start_price: u128,       // Minimum price

    // Bonding curve definition (20 points)
    pub curve: [LiquidityDistributionConfig; 20],
}
```

### Sub-structures

#### PoolFeesConfig (128 bytes)

```rust
pub struct PoolFeesConfig {
    pub base_fee: BaseFeeConfig,      // 32 bytes
    pub dynamic_fee: DynamicFeeConfig, // 48 bytes
    pub protocol_fee_percent: u8,      // Protocol's share of fees (e.g., 20%)
    pub referral_fee_percent: u8,      // Referral's share of protocol fee
    // padding: 46 bytes
}
```

#### BaseFeeConfig (32 bytes)

```rust
pub struct BaseFeeConfig {
    pub cliff_fee_numerator: u64,     // Initial fee (e.g., 10_000_000 = 1%)
    pub second_factor: u64,            // Period frequency or max duration
    pub third_factor: u64,             // Reduction factor or reference amount
    pub first_factor: u16,             // Number of periods or increment BPS
    pub base_fee_mode: u8,             // FeeSchedulerLinear(0), Exponential(1), RateLimiter(2)
    // padding: 5 bytes
}
```

**Fee Modes**:
- **FeeSchedulerLinear**: `fee = cliff_fee - (passed_periods * reduction_factor)`
- **FeeSchedulerExponential**: `fee = cliff_fee * (1 - reduction_factor/10000)^passed_periods`
- **RateLimiter**: Anti-sniper protection, increases fee for rapid trading

#### DynamicFeeConfig (48 bytes)

```rust
pub struct DynamicFeeConfig {
    pub initialized: u8,               // 0=disabled, 1=enabled
    pub max_volatility_accumulator: u32, // Cap for volatility (e.g., 14460000)
    pub variable_fee_control: u32,     // Fee multiplier
    pub bin_step: u16,                 // Price bin size (e.g., 1 = 0.01%)
    pub filter_period: u16,            // Min time between updates (seconds)
    pub decay_period: u16,             // Volatility decay time (seconds)
    pub reduction_factor: u16,         // Decay rate (e.g., 5000 = 50%)
    pub bin_step_u128: u128,           // bin_step << 64 / 10000
    // padding: 15 bytes
}
```

**Dynamic Fee Calculation**:
```
variable_fee = (volatility_accumulator * bin_step)^2 * variable_fee_control / 10^11
total_fee = base_fee + variable_fee
```

#### LockedVestingConfig (48 bytes)

```rust
pub struct LockedVestingConfig {
    pub amount_per_period: u64,       // Tokens vested each period
    pub cliff_duration_from_migration_time: u64, // Initial lock time
    pub frequency: u64,                // Vesting period length
    pub number_of_period: u64,         // Total vesting periods
    pub cliff_unlock_amount: u64,      // Tokens unlocked at cliff
    // padding: 8 bytes
}
```

#### LiquidityDistributionConfig (32 bytes)

```rust
pub struct LiquidityDistributionConfig {
    pub sqrt_price: u128,              // Price point (Q64.64 format)
    pub liquidity: u128,               // Virtual liquidity at this point
}
```

**Curve Definition**:
The bonding curve is defined by up to 20 price points. Each segment between points has:
- Start price: `curve[i].sqrt_price` (or `sqrt_start_price` for first segment)
- End price: `curve[i+1].sqrt_price`
- Liquidity: `curve[i+1].liquidity`

**Location**: `programs/dynamic-bonding-curve/src/state/config.rs`

---

## 2. VirtualPool

**Size**: 416 bytes (zero-copy)

**PDA Seeds**: `["pool", base_mint, config]`

The VirtualPool represents the active state of a bonding curve pool.

### Structure

```rust
pub struct VirtualPool {
    // Volatility tracking (64 bytes)
    pub volatility_tracker: VolatilityTracker,

    // References
    pub config: Pubkey,               // Associated config
    pub creator: Pubkey,              // Pool creator
    pub base_mint: Pubkey,            // Base token mint
    pub base_vault: Pubkey,           // Base token vault
    pub quote_vault: Pubkey,          // Quote token vault

    // Reserves
    pub base_reserve: u64,            // Current base token reserve
    pub quote_reserve: u64,           // Current quote token reserve

    // Accumulated fees
    pub protocol_base_fee: u64,       // Protocol fees in base token
    pub protocol_quote_fee: u64,      // Protocol fees in quote token
    pub partner_base_fee: u64,        // Partner fees in base token
    pub partner_quote_fee: u64,       // Partner fees in quote token
    pub creator_base_fee: u64,        // Creator fees in base token
    pub creator_quote_fee: u64,       // Creator fees in quote token

    // Current state
    pub sqrt_price: u128,             // Current price (Q64.64)
    pub activation_point: u64,        // Slot or timestamp of activation

    // Flags
    pub pool_type: u8,                // SplToken(0) or Token2022(1)
    pub is_migrated: u8,              // 0=active, 1=migrated
    pub is_partner_withdraw_surplus: u8,
    pub is_protocol_withdraw_surplus: u8,
    pub migration_progress: u8,       // MigrationProgress enum
    pub is_withdraw_leftover: u8,
    pub is_creator_withdraw_surplus: u8,
    pub migration_fee_withdraw_status: u8, // Bitflags

    // Metrics (32 bytes)
    pub metrics: PoolMetrics,

    // Timestamps
    pub finish_curve_timestamp: u64,  // When curve completed

    // Padding: 56 bytes
}
```

### Sub-structures

#### VolatilityTracker (64 bytes)

```rust
pub struct VolatilityTracker {
    pub last_update_timestamp: u64,
    pub sqrt_price_reference: u128,   // Reference price for volatility calc
    pub volatility_accumulator: u128, // Current volatility
    pub volatility_reference: u128,   // Decayed volatility baseline
    // padding: 8 bytes
}
```

**Update Logic**:
1. On each swap, calculate price change: `delta_bin = (upper_price / lower_price - 1) / bin_step`
2. Update accumulator: `volatility_accumulator = volatility_reference + delta_bin * 10000`
3. After filter_period: update reference price and decay volatility

#### PoolMetrics (32 bytes)

```rust
pub struct PoolMetrics {
    pub total_protocol_base_fee: u64,
    pub total_protocol_quote_fee: u64,
    pub total_trading_base_fee: u64,   // Partner + Creator fees combined
    pub total_trading_quote_fee: u64,
}
```

#### MigrationProgress (enum)

```rust
pub enum MigrationProgress {
    PreBondingCurve = 0,    // Initial state
    PostBondingCurve = 1,   // Curve complete, vesting active
    LockedVesting = 2,      // Ready for migration
    CreatedPool = 3,        // Migrated to DEX
}
```

**State Transitions**:
```
PreBondingCurve
    ↓ (curve completes with vesting)
PostBondingCurve
    ↓ (vesting complete or no vesting)
LockedVesting
    ↓ (migration executed)
CreatedPool
```

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool.rs`

---

## 3. PartnerMetadata

**Size**: Variable (8 + name.len() + uri.len())

**PDA Seeds**: `["partner_metadata", partner_pubkey]`

### Structure

```rust
pub struct PartnerMetadata {
    pub partner: Pubkey,
    pub name: String,
    pub uri: String,
}
```

**Purpose**: Stores partner information for UI display and tracking.

**Location**: `programs/dynamic-bonding-curve/src/state/partner_metadata.rs`

---

## 4. VirtualPoolMetadata

**Size**: Variable (8 + name.len() + uri.len())

**PDA Seeds**: `["virtual_pool_metadata", pool_pubkey]`

### Structure

```rust
pub struct VirtualPoolMetadata {
    pub virtual_pool: Pubkey,
    pub name: String,
    pub uri: String,
}
```

**Purpose**: Additional metadata for pool display.

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool_metadata.rs`

---

## 5. ClaimFeeOperator

**Size**: Variable (8 + 32)

**PDA Seeds**: `["cf_operator", operator_pubkey]`

### Structure

```rust
pub struct ClaimFeeOperator {
    pub operator: Pubkey,  // Authorized operator
}
```

**Purpose**: Authorizes specific addresses to claim protocol fees.

**Location**: `programs/dynamic-bonding-curve/src/state/claim_fee_operator.rs`

---

## Account Relationships

```
┌─────────────────┐
│   PoolConfig    │◄────┐
│                 │     │
│ • Curve         │     │
│ • Fees          │     │
│ • Migration     │     │
└─────────────────┘     │
                        │ has_one
                        │
                  ┌─────────────────┐
                  │  VirtualPool    │
                  │                 │
                  │ • Reserves      │
                  │ • Price         │
                  │ • Fees          │
                  └─────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        │               │               │
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Base Vault  │ │ Quote Vault  │ │   Creator    │
│  (TokenAcct) │ │ (TokenAcct)  │ │   (Wallet)   │
└──────────────┘ └──────────────┘ └──────────────┘


┌─────────────────┐       ┌─────────────────┐
│ Partner         │       │ VirtualPool     │
│ Metadata        │       │ Metadata        │
└─────────────────┘       └─────────────────┘
```

## Storage Optimization

The program uses **zero-copy** deserialization for large accounts (PoolConfig, VirtualPool) to minimize compute units:

```rust
#[account(zero_copy)]
pub struct VirtualPool { ... }

// Load as reference (no copying)
let pool = ctx.accounts.pool.load()?;      // Read-only
let mut pool = ctx.accounts.pool.load_mut()?; // Mutable
```

## Constants

### Price Bounds
- `MIN_SQRT_PRICE`: 4295048016 (~0.000001 price)
- `MAX_SQRT_PRICE`: 79226673521066979257578248091 (~1,000,000 price)

### Fee Constants
- `FEE_DENOMINATOR`: 1,000,000,000 (9 decimals precision)
- `MAX_FEE_BPS`: 9900 (99%)
- `MIN_FEE_BPS`: 1 (0.01%)
- `PROTOCOL_FEE_PERCENT`: 20% (of total trading fee)

### Other
- `MAX_CURVE_POINT_CONFIG`: 20 price points
- `PARTNER_AND_CREATOR_SURPLUS_SHARE`: 80% (of surplus)
- `SWAP_BUFFER_PERCENTAGE`: 25% (extra tokens for buffer)

**Location**: `programs/dynamic-bonding-curve/src/constants.rs`

## Account Size Examples

### Typical PoolConfig
```
8 (discriminator)
+ 1040 (struct)
--------------
= 1048 bytes
≈ 0.0072 SOL rent
```

### Typical VirtualPool
```
8 (discriminator)
+ 416 (struct)
--------------
= 424 bytes
≈ 0.0029 SOL rent
```

### PartnerMetadata
```
8 (discriminator)
+ 32 (pubkey)
+ 4 + name.len()
+ 4 + uri.len()
--------------
= Variable (typically ~200 bytes)
```
