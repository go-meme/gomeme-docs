# Fee System

This document explains the comprehensive fee system including base fees, dynamic fees, fee distribution, and anti-sniper protection.

## Overview

The fee system combines three components:
1. **Base Fee**: Time-based fee that decreases over time
2. **Dynamic Fee**: Volatility-based fee that increases during high trading activity
3. **Rate Limiter**: Anti-sniper protection that penalizes rapid trading

```
Total Trading Fee = Base Fee + Dynamic Fee
```

The total fee is then split between protocol, partner, creator, and optionally referral.

---

## 1. Base Fee

Base fees decrease over time according to a schedule, making it cheaper to trade as the pool matures.

### Fee Modes

#### 1.1 Linear Fee Scheduler

Fees decrease linearly over time:

```
fee(t) = cliff_fee - (passed_periods × reduction_factor)
```

**Configuration**:
```rust
BaseFeeConfig {
    cliff_fee_numerator: 10_000_000,    // 1% initial fee
    first_factor: 100,                  // number_of_periods
    second_factor: 60,                  // period_frequency (seconds)
    third_factor: 100_000,              // reduction_factor per period
    base_fee_mode: 0,                   // FeeSchedulerLinear
}
```

**Example**:
```
t = 0s:    fee = 10,000,000 (1.0%)
t = 60s:   fee = 10,000,000 - 1 × 100,000 = 9,900,000 (0.99%)
t = 120s:  fee = 10,000,000 - 2 × 100,000 = 9,800,000 (0.98%)
...
t = 6000s: fee = 10,000,000 - 100 × 100,000 = 0 (0%)
```

**Location**: `programs/dynamic-bonding-curve/src/base_fee/fee_scheduler.rs`

---

#### 1.2 Exponential Fee Scheduler

Fees decrease exponentially over time:

```
fee(t) = cliff_fee × (1 - reduction_factor/10000)^passed_periods
```

**Configuration**:
```rust
BaseFeeConfig {
    cliff_fee_numerator: 10_000_000,    // 1% initial fee
    first_factor: 100,                  // number_of_periods
    second_factor: 60,                  // period_frequency (seconds)
    third_factor: 500,                  // reduction_factor (5% per period)
    base_fee_mode: 1,                   // FeeSchedulerExponential
}
```

**Example**:
```
t = 0s:    fee = 10,000,000 × (1)^0 = 10,000,000 (1.0%)
t = 60s:   fee = 10,000,000 × (0.95)^1 = 9,500,000 (0.95%)
t = 120s:  fee = 10,000,000 × (0.95)^2 = 9,025,000 (0.9025%)
...
t = 6000s: fee = 10,000,000 × (0.95)^100 ≈ 59,000 (0.0059%)
```

**Location**: `programs/dynamic-bonding-curve/src/base_fee/fee_scheduler.rs`

---

#### 1.3 Rate Limiter (Anti-Sniper)

Increases fees for traders who execute multiple swaps within a short time period:

```
Effective fee = cliff_fee + (traded_amount / reference_amount) × fee_increment × periods_active
```

**Configuration**:
```rust
BaseFeeConfig {
    cliff_fee_numerator: 1_000_000,     // 0.1% base fee
    first_factor: 100,                  // fee_increment_bps (1% per reference amount)
    second_factor: 600,                 // max_limiter_duration (10 minutes)
    third_factor: 1000_000_000,         // reference_amount (1000 tokens w/ 6 decimals)
    base_fee_mode: 2,                   // RateLimiter
}
```

**How it works**:
1. Track cumulative trade volume since activation
2. Calculate penalty: `(volume / reference_amount) × fee_increment_bps`
3. Apply penalty only during first `max_limiter_duration` seconds
4. After duration expires, fee returns to `cliff_fee_numerator`

**Example**:
```
User buys 5000 tokens within first 10 minutes:
penalty_bps = (5000 / 1000) × 100 = 500 bps = 5%
effective_fee = 0.1% + 5% = 5.1%

Regular user buys 100 tokens:
penalty_bps = (100 / 1000) × 100 = 10 bps = 0.1%
effective_fee = 0.1% + 0.1% = 0.2%
```

**Anti-Sniper Protection**: This prevents bots from buying large amounts immediately after pool creation.

**Location**: `programs/dynamic-bonding-curve/src/base_fee/fee_rate_limiter.rs`

---

## 2. Dynamic Fee

Dynamic fees increase during periods of high volatility to protect liquidity providers and discourage front-running.

### Volatility Tracking

The program tracks price volatility using a volatility accumulator:

```rust
pub struct VolatilityTracker {
    pub last_update_timestamp: u64,
    pub sqrt_price_reference: u128,     // Reference price
    pub volatility_accumulator: u128,   // Current volatility
    pub volatility_reference: u128,     // Decayed baseline
}
```

### Update Logic

**On each swap**:

1. **Before Swap** (update references):
```rust
elapsed = current_timestamp - last_update_timestamp

if elapsed >= filter_period {
    // Update reference price
    sqrt_price_reference = current_price

    if elapsed < decay_period {
        // Decay time window
        volatility_reference = volatility_accumulator × reduction_factor / 10000
    } else {
        // Outside decay window, reset
        volatility_reference = 0
    }
}
```

2. **After Swap** (update accumulator):
```rust
// Calculate price change in bins
delta_bin = (upper_price / lower_price - 1) / bin_step × 2

// Update accumulator
volatility_accumulator = volatility_reference + delta_bin × 10000

// Cap at maximum
volatility_accumulator = min(volatility_accumulator, max_volatility_accumulator)
```

**Location**: `programs/dynamic-bonding-curve/src/state/fee.rs:58-108`

### Fee Calculation

```rust
variable_fee = (volatility_accumulator × bin_step)^2 × variable_fee_control / 10^11
```

**Configuration**:
```rust
DynamicFeeConfig {
    initialized: 1,                     // 1 = enabled
    max_volatility_accumulator: 14_460_000,
    variable_fee_control: 100_000,
    bin_step: 1,                        // 0.01% price bins
    filter_period: 10,                  // 10 seconds
    decay_period: 120,                  // 2 minutes
    reduction_factor: 5000,             // 50% decay
    bin_step_u128: 1_844_674_407_370_955, // bin_step << 64 / 10000
}
```

**Example**:
```
volatility_accumulator = 1,000,000 (moderate activity)
bin_step = 1

variable_fee = (1,000,000 × 1)^2 × 100,000 / 10^11
             = 10^12 × 100,000 / 10^11
             = 10,000,000
             = 1%
```

**Location**: `programs/dynamic-bonding-curve/src/state/config.rs:299-322`

---

## 3. Fee Collection Modes

Fees can be collected in different tokens depending on the pool configuration:

### CollectFeeMode

```rust
pub enum CollectFeeMode {
    QuoteToken = 0,   // Always collect fees in quote token
    OutputToken = 1,  // Collect fees in output token of swap
}
```

### Fee Application

The mode affects whether fees are taken from input or output:

```rust
pub struct FeeMode {
    pub fees_on_input: bool,       // Take fees from input amount?
    pub fees_on_base_token: bool,  // Are fees in base token?
    pub has_referral: bool,         // Is there a referral?
}
```

**Logic**:
```rust
match (collect_fee_mode, trade_direction) {
    (QuoteToken, BaseToQuote) => {
        fees_on_input = false     // Take from output (quote)
        fees_on_base_token = false
    }
    (QuoteToken, QuoteToBase) => {
        fees_on_input = true      // Take from input (quote)
        fees_on_base_token = false
    }
    (OutputToken, BaseToQuote) => {
        fees_on_input = false     // Take from output (quote)
        fees_on_base_token = false
    }
    (OutputToken, QuoteToBase) => {
        fees_on_input = false     // Take from output (base)
        fees_on_base_token = true
    }
}
```

**Location**: `programs/dynamic-bonding-curve/src/state/fee.rs:117-142`

---

## 4. Fee Distribution

Trading fees are split between multiple parties:

### Primary Split

```
Total Trading Fee
    │
    ├─► Protocol Fee (e.g., 20% of total)
    │       │
    │       ├─► Protocol (80%)
    │       └─► Referral (20%, optional)
    │
    └─► Trading Fee (e.g., 80% of total)
            │
            ├─► Partner Fee (based on creator_trading_fee_percentage)
            └─► Creator Fee (remaining)
```

### Configuration

```rust
pub struct PoolFeesConfig {
    pub protocol_fee_percent: u8,      // e.g., 20
    pub referral_fee_percent: u8,      // e.g., 20
    // ...
}

pub struct PoolConfig {
    pub creator_trading_fee_percentage: u8,  // e.g., 50
    // ...
}
```

### Calculation

**Step 1**: Split total fee into protocol and trading:
```rust
protocol_fee = total_fee × protocol_fee_percent / 100
trading_fee = total_fee - protocol_fee
```

**Step 2**: Split protocol fee with referral (if present):
```rust
if has_referral {
    referral_fee = protocol_fee × referral_fee_percent / 100
    protocol_fee = protocol_fee - referral_fee
}
```

**Step 3**: Split trading fee between partner and creator:
```rust
creator_fee = trading_fee × creator_trading_fee_percentage / 100
partner_fee = trading_fee - creator_fee
```

**Location**: `programs/dynamic-bonding-curve/src/state/config.rs:196-221`

### Example

```
Total trading fee: 1000 tokens
protocol_fee_percent: 20
referral_fee_percent: 20
creator_trading_fee_percentage: 50
has_referral: true

Calculation:
protocol_fee = 1000 × 20 / 100 = 200
referral_fee = 200 × 20 / 100 = 40
protocol_fee = 200 - 40 = 160
trading_fee = 1000 - 200 = 800
creator_fee = 800 × 50 / 100 = 400
partner_fee = 800 - 400 = 400

Result:
- Protocol: 160 tokens (16%)
- Referral: 40 tokens (4%)
- Creator:  400 tokens (40%)
- Partner:  400 tokens (40%)
```

---

## 5. Fee Accumulation

Fees accumulate in the pool and can be claimed separately:

### Pool State

```rust
pub struct VirtualPool {
    // Accumulated fees
    pub protocol_base_fee: u64,
    pub protocol_quote_fee: u64,
    pub partner_base_fee: u64,
    pub partner_quote_fee: u64,
    pub creator_base_fee: u64,
    pub creator_quote_fee: u64,
    // ...
}
```

### Fee Updates (on swap)

```rust
fn apply_swap_result(...) {
    let PartnerAndCreatorSplitFee { partner_fee, creator_fee } =
        config.split_partner_and_creator_fee(trading_fee)?;

    if fee_mode.fees_on_base_token {
        self.partner_base_fee += partner_fee;
        self.protocol_base_fee += protocol_fee;
        self.creator_base_fee += creator_fee;
    } else {
        self.partner_quote_fee += partner_fee;
        self.protocol_quote_fee += protocol_fee;
        self.creator_quote_fee += creator_fee;
    }
}
```

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool.rs:820-878`

---

## 6. Fee Constants

```rust
// Fee denominator (9 decimals of precision)
pub const FEE_DENOMINATOR: u64 = 1_000_000_000;

// Fee bounds
pub const MAX_FEE_BPS: u64 = 9900;        // 99%
pub const MAX_FEE_NUMERATOR: u64 = 990_000_000;

pub const MIN_FEE_BPS: u64 = 1;           // 0.01%
pub const MIN_FEE_NUMERATOR: u64 = 100_000;

// Default percentages
pub const PROTOCOL_FEE_PERCENT: u8 = 20;  // 20% of total
pub const HOST_FEE_PERCENT: u8 = 20;      // 20% of protocol fee
```

**Fee Numerator Format**:
```
Actual fee = fee_numerator / FEE_DENOMINATOR

Examples:
1_000_000 = 0.1%
10_000_000 = 1%
100_000_000 = 10%
990_000_000 = 99%
```

**Location**: `programs/dynamic-bonding-curve/src/constants.rs:58-86`

---

## 7. Fee Claiming

### Protocol Fees

**Instruction**: `claim_protocol_fee`

**Access**: Claim operator only

```rust
let (base_amount, quote_amount) = pool.claim_protocol_fee();
pool.protocol_base_fee = 0;
pool.protocol_quote_fee = 0;
```

### Partner Fees

**Instruction**: `claim_trading_fee`

**Access**: Partner (config fee_claimer)

```rust
let (base_amount, quote_amount) = pool.claim_partner_trading_fee(
    max_base_amount,
    max_quote_amount,
)?;
pool.partner_base_fee -= base_amount;
pool.partner_quote_fee -= quote_amount;
```

### Creator Fees

**Instruction**: `claim_creator_trading_fee`

**Access**: Creator only

```rust
let (base_amount, quote_amount) = pool.claim_creator_trading_fee(
    max_base_amount,
    max_quote_amount,
)?;
pool.creator_base_fee -= base_amount;
pool.creator_quote_fee -= quote_amount;
```

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool.rs:915-945`

---

## 8. Migration Fees

When the curve completes and migrates to a DEX, an additional fee can be charged:

### Configuration

```rust
pub struct PoolConfig {
    pub migration_fee_percentage: u8,         // Total migration fee
    pub creator_migration_fee_percentage: u8, // Creator's share
    // ...
}
```

### Calculation

```rust
// From total migration amount
migration_fee = migration_amount × migration_fee_percentage / 100
migration_amount_after_fee = migration_amount - migration_fee

// Split between partner and creator
creator_migration_fee = migration_fee × creator_migration_fee_percentage / 100
partner_migration_fee = migration_fee - creator_migration_fee
```

### Withdrawal

**Instruction**: `withdraw_migration_fee`

**Flags**:
```rust
pub const PARTNER_MASK: u8 = 0b100;
pub const CREATOR_MASK: u8 = 0b010;
```

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/ix_withdraw_migration_fee.rs`

---

## 9. Surplus Distribution

After migration, any excess quote tokens are distributed:

### Calculation

```rust
total_surplus = pool.quote_reserve - migration_threshold

partner_and_creator_surplus = total_surplus × 80 / 100  // 80% share
protocol_surplus = total_surplus × 20 / 100             // 20% share

creator_surplus = partner_and_creator_surplus × creator_share / 100
partner_surplus = partner_and_creator_surplus - creator_surplus
```

### Constants

```rust
pub const PARTNER_AND_CREATOR_SURPLUS_SHARE: u8 = 80;  // 80%
```

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool.rs:962-998`

---

## Fee Flow Diagram

```
                         ┌─────────────────┐
                         │   Swap Occurs   │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │  Calculate Total Fee    │
                    │  = Base Fee + Dynamic   │
                    └─────────┬───────────────┘
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
    ┌────────────────────┐      ┌────────────────────┐
    │   Protocol Fee     │      │   Trading Fee      │
    │       (20%)        │      │       (80%)        │
    └─────┬──────────────┘      └────────┬───────────┘
          │                              │
    ┌─────┴──────┐           ┌───────────┴──────────┐
    ▼            ▼           ▼                      ▼
┌────────┐  ┌─────────┐  ┌────────┐          ┌─────────┐
│Protocol│  │Referral │  │Partner │          │ Creator │
│  (80%) │  │  (20%)  │  │  (50%) │          │  (50%)  │
└────────┘  └─────────┘  └────────┘          └─────────┘
    │            │           │                     │
    │            │           │                     │
    ▼            ▼           ▼                     ▼
 Claimable   Sent to    Claimable            Claimable
 by admin   referral    by partner           by creator
```
