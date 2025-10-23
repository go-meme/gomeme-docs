# Migration System

This document provides in-depth details about the migration system that transfers liquidity from the bonding curve to established DEXs.

## Overview

When a bonding curve completes (reaches its target), the accumulated liquidity is migrated to a permanent DEX pool. The program supports two migration targets:

1. **Meteora Dynamic AMM (DAMM)** - First-generation Meteora pools
2. **Meteora Dynamic AMM V2** - Second-generation with improved features

## Migration States

### MigrationProgress Enum

```rust
pub enum MigrationProgress {
    PreBondingCurve = 0,    // Active trading phase
    PostBondingCurve = 1,   // Curve complete, vesting active
    LockedVesting = 2,      // Ready for migration
    CreatedPool = 3,        // Migrated to DEX
}
```

### State Transitions

```
┌─────────────────────┐
│  PreBondingCurve    │  Active trading
│  (0)                │  quote_reserve < threshold
└──────────┬──────────┘
           │
           │ Last swap reaches threshold
           │ Event: EvtCurveComplete
           │
           ▼
    ┌──────────────┐
    │ Has vesting? │
    └──────┬───────┘
           │
     ┌─────┴─────┐
     │           │
    Yes          No
     │           │
     ▼           ▼
┌──────────┐  ┌─────────────┐
│PostBonding│  │LockedVesting│
│Curve (1)  │  │    (2)      │
└─────┬─────┘  └──────┬──────┘
      │               │
      │ Vesting       │
      │ complete      │
      └───────┬───────┘
              │
              ▼
      ┌──────────────┐
      │ CreatedPool  │  Migration executed
      │    (3)       │  is_migrated = 1
      └──────────────┘
```

---

## Migration Configuration

### Pool Config Parameters

```rust
pub struct PoolConfig {
    // Migration settings
    pub migration_option: u8,          // 0=MeteoraDamm, 1=DammV2
    pub migration_quote_threshold: u64, // Quote amount to trigger migration
    pub migration_base_threshold: u64,  // Base tokens for migration pool
    pub migration_sqrt_price: u128,     // Target price for migration

    // Migration fees
    pub migration_fee_percentage: u8,         // Total fee (e.g., 1%)
    pub creator_migration_fee_percentage: u8, // Creator's share (e.g., 50%)
    pub migration_fee_option: u8,             // Fee tier option

    // LP distribution
    pub partner_locked_lp_percentage: u8,  // Locked LP for partner
    pub partner_lp_percentage: u8,         // Unlocked LP for partner
    pub creator_locked_lp_percentage: u8,  // Locked LP for creator
    pub creator_lp_percentage: u8,         // Unlocked LP for creator

    // Migrated pool configuration
    pub migrated_pool_fee_bps: u16,        // Fee for migrated pool
    pub migrated_collect_fee_mode: u8,     // Fee collection mode
    pub migrated_dynamic_fee: u8,          // Dynamic fee option

    // Vesting for locked LP
    pub locked_vesting_config: LockedVestingConfig,
}
```

### Migration Fee Options

```rust
pub enum MigrationFeeOption {
    FixedBps25 = 0,    // 0.25%
    FixedBps30 = 1,    // 0.3%
    FixedBps100 = 2,   // 1%
    FixedBps200 = 3,   // 2%
    FixedBps400 = 4,   // 4%
    FixedBps600 = 5,   // 6%
    Customizable = 6,  // Custom fee
}
```

---

## Curve Completion

### Trigger Conditions

A curve completes when:
```rust
pool.quote_reserve >= config.migration_quote_threshold
```

This check happens on every swap in `handle_swap_wrapper()`.

### Completion Logic

```rust
// Check if curve completed
if pool.is_curve_complete(config.migration_quote_threshold) {
    // 1. Validate base token reserve
    let required_base_balance = config.migration_base_threshold
        + pool.get_protocol_and_trading_base_fee()?
        + locked_vesting_total;

    require!(
        base_vault_balance >= required_base_balance,
        PoolError::InsufficientLiquidityForMigration
    );

    // 2. Record completion time
    pool.finish_curve_timestamp = current_timestamp;

    // 3. Update migration progress
    if locked_vesting_params.has_vesting() {
        pool.set_migration_progress(MigrationProgress::PostBondingCurve);
    } else {
        pool.set_migration_progress(MigrationProgress::LockedVesting);
    }

    // 4. Emit event
    emit!(EvtCurveComplete {
        pool: pool_key,
        config: config_key,
        base_reserve: pool.base_reserve,
        quote_reserve: pool.quote_reserve,
    });
}
```

**Location**: `programs/dynamic-bonding-curve/src/instructions/swap/ix_swap.rs:287-323`

---

## Migration Amounts

### Quote Token Calculation

The migration takes most quote tokens, minus fees:

```rust
pub struct MigrationAmount {
    pub quote_amount: u64,  // Amount for DEX pool
    pub fee: u64,           // Migration fee
}

fn get_migration_quote_amount(
    migration_quote_threshold: u64,
    migration_fee_percentage: u8,
) -> MigrationAmount {
    // Calculate net amount after fee
    let quote_amount = migration_quote_threshold
        * (100 - migration_fee_percentage)
        / 100;

    let fee = migration_quote_threshold - quote_amount;

    MigrationAmount { quote_amount, fee }
}
```

**Example**:
```
migration_quote_threshold = 85_000_000 (85 USDC)
migration_fee_percentage = 1

quote_amount = 85 * 99 / 100 = 84.15 USDC (to DEX)
fee = 85 - 84.15 = 0.85 USDC (split partner/creator)
```

### Base Token Calculation

```rust
migration_base_amount = config.migration_base_threshold
```

This is a fixed amount configured when the pool is created.

### LP Token Distribution

After migration, LP tokens are distributed:

```rust
pub struct LiquidityDistribution {
    pub partner: LiquidityDistributionItem {
        pub unlocked_liquidity: u128,  // Immediately claimable
        pub locked_liquidity: u128,    // Vested over time
    },
    pub creator: LiquidityDistributionItem {
        pub unlocked_liquidity: u128,
        pub locked_liquidity: u128,
    },
}

fn get_lp_distribution(lp_amount: u64) -> LiquidityDistributionU64 {
    let partner_locked_lp = lp_amount
        * partner_locked_lp_percentage / 100;

    let partner_lp = lp_amount
        * partner_lp_percentage / 100;

    let creator_locked_lp = lp_amount
        * creator_locked_lp_percentage / 100;

    let creator_lp = lp_amount
        - partner_locked_lp
        - partner_lp
        - creator_locked_lp;

    LiquidityDistributionU64 {
        partner_locked_lp,
        partner_lp,
        creator_locked_lp,
        creator_lp,
    }
}
```

**Example Configuration**:
```
partner_locked_lp_percentage = 30
partner_lp_percentage = 20
creator_locked_lp_percentage = 30
creator_lp_percentage = 20 (calculated)

Total LP tokens = 1000

Distribution:
- Partner locked: 300 LP
- Partner unlocked: 200 LP
- Creator locked: 300 LP
- Creator unlocked: 200 LP
```

**Location**: `programs/dynamic-bonding-curve/src/state/config.rs:769-835`

---

## Meteora DAMM Migration

### Overview

Migrates to Meteora's first-generation Dynamic AMM.

### Migration Flow

```
1. migration_meteora_damm_create_metadata
   └─► Creates migration metadata account

2. create_locker
   └─► Creates locker for LP token vesting

3. migrate_meteora_damm
   └─► Initializes Meteora pool
   └─► Transfers liquidity
   └─► Mints LP tokens

4. migrate_meteora_damm_lock_lp_token
   └─► Locks portion of LP tokens

5. (After vesting) migrate_meteora_damm_claim_lp_token
   └─► Claims unlocked LP tokens
```

### Metadata Structure

```rust
pub struct MeteoraDammMigrationMetadata {
    pub pool: Pubkey,
    pub damm_pool: Pubkey,
    pub locker: Pubkey,
    pub partner_locked_escrow: Pubkey,
    pub creator_locked_escrow: Pubkey,
}
```

**PDA Seeds**: `["meteora", pool_pubkey]`

### Migration Instruction

The migration creates a new Meteora pool with:
- Initial price from bonding curve
- Liquidity from accumulated reserves
- Fee configuration from pool config

**Key Accounts**:
- Bonding curve pool and vaults
- Meteora pool and vaults
- LP token mint
- Locker accounts
- Partner and creator token accounts

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/meteora_damm/`

---

## DAMM V2 Migration

### Overview

Migrates to Meteora's second-generation Dynamic AMM with improved features.

### Migration Flow

```
1. migration_damm_v2_create_metadata
   └─► Creates V2 migration metadata

2. create_locker
   └─► Creates locker for vesting

3. migration_damm_v2
   └─► Initializes DAMM V2 pool
   └─► Transfers liquidity
   └─► Handles LP distribution
```

### Metadata Structure

```rust
pub struct DammV2MigrationMetadata {
    pub pool: Pubkey,
    pub damm_pool: Pubkey,
    pub locker: Pubkey,
}
```

**PDA Seeds**: `["damm_v2", pool_pubkey]`

### Differences from DAMM V1

1. **Simpler LP locking** - Built-in locker integration
2. **Better fee structure** - More flexible fee options
3. **Improved oracle** - Better price feed integration

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/dynamic_amm_v2/`

---

## Locker System

### Purpose

The locker holds LP tokens and releases them according to a vesting schedule.

### Locker Account

```rust
// From external locker program
pub struct Locker {
    pub base: Pubkey,
    pub beneficiary: Pubkey,
    pub token_mint: Pubkey,
    pub locked_supply: u64,
    pub start_timestamp: i64,
    pub cliff_timestamp: i64,
    pub end_timestamp: i64,
}
```

**PDA Seeds**: `["base_locker", pool_pubkey]`

### Vesting Configuration

```rust
pub struct LockedVestingConfig {
    pub amount_per_period: u64,                    // Per-period release
    pub cliff_duration_from_migration_time: u64,   // Initial lock
    pub frequency: u64,                            // Vesting interval
    pub number_of_period: u64,                     // Total periods
    pub cliff_unlock_amount: u64,                  // Cliff unlock
}
```

### Vesting Schedule

```
Total locked amount = amount_per_period × number_of_period + cliff_unlock_amount

Timeline:
t=0 (migration) ──┬────────┬────────┬────────┬──── ... ──┬────────►
                  │        │        │        │           │
           cliff_duration  │        │        │           │
                  │        │        │        │           │
                  ▼        ▼        ▼        ▼           ▼
             cliff_unlock  period1  period2  period3 ... periodN
             amount        amount   amount   amount      amount
```

**Example**:
```
cliff_unlock_amount = 100 LP
amount_per_period = 50 LP
number_of_period = 10
cliff_duration_from_migration_time = 86400 (1 day)
frequency = 86400 (1 day)

Total locked: 100 + 50*10 = 600 LP

Schedule:
Day 0: Nothing
Day 1: 100 LP unlocked (cliff)
Day 2: 50 LP unlocked (period 1)
Day 3: 50 LP unlocked (period 2)
...
Day 11: 50 LP unlocked (period 10)
```

**Location**: `programs/dynamic-bonding-curve/src/state/config.rs:324-347`

---

## Surplus Distribution

### What is Surplus?

After migration, any quote tokens above the threshold are "surplus":

```rust
total_surplus = pool.quote_reserve - config.migration_quote_threshold
```

### Distribution

```rust
partner_and_creator_surplus = total_surplus × 80 / 100
protocol_surplus = total_surplus × 20 / 100

// Further split between partner and creator
creator_surplus = partner_and_creator_surplus
    × creator_share / 100

partner_surplus = partner_and_creator_surplus
    - creator_surplus
```

**Constants**:
```rust
pub const PARTNER_AND_CREATOR_SURPLUS_SHARE: u8 = 80;  // 80%
```

### Withdrawal

Each party withdraws their surplus independently:

```rust
// Partner
await program.methods.partnerWithdrawSurplus()...

// Creator
await program.methods.creatorWithdrawSurplus()...

// Protocol
await program.methods.protocolWithdrawSurplus()...
```

**Requirements**:
- Pool must be migrated (`is_migrated = 1`)
- Surplus not already withdrawn
- Caller must be authorized party

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool.rs:962-1010`

---

## Migration Fees

### Fee Structure

```rust
migration_fee = migration_quote_threshold × migration_fee_percentage / 100

creator_migration_fee = migration_fee × creator_migration_fee_percentage / 100
partner_migration_fee = migration_fee - creator_migration_fee
```

### Withdrawal

```rust
pub const PARTNER_MASK: u8 = 0b100;
pub const CREATOR_MASK: u8 = 0b010;

// Track withdrawal with bitflags
pub migration_fee_withdraw_status: u8

// Partner withdraws
withdraw_migration_fee(PARTNER_MASK)
→ migration_fee_withdraw_status |= PARTNER_MASK

// Creator withdraws
withdraw_migration_fee(CREATOR_MASK)
→ migration_fee_withdraw_status |= CREATOR_MASK
```

**Eligibility Check**:
```rust
fn eligible_to_withdraw_migration_fee(mask: u8) -> bool {
    migration_fee_withdraw_status & mask == 0
}
```

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/ix_withdraw_migration_fee.rs`

---

## Leftover Tokens

### What are Leftovers?

For **fixed supply tokens**, there may be excess base tokens after migration:

```rust
leftover = base_vault_balance
    - migration_base_threshold
    - protocol_and_trading_fees
    - locked_vesting_amount
```

### Burnable vs Non-burnable

```rust
max_burnable = pre_migration_token_supply - post_migration_token_supply

burnable_amount = min(leftover, max_burnable)
```

**Options**:
1. **Burn** excess tokens (if mintable)
2. **Transfer** to `leftover_receiver` (if fixed supply)

### Withdrawal

```rust
await program.methods.withdrawLeftover()
    .accounts({
        config,
        pool,
        leftoverReceiver,
        baseVault,
        leftoverReceiverAccount,
        // ...
    })
    .rpc();
```

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/withdraw_leftover.rs`

---

## Migration Checklist

Before executing migration:

- [ ] Curve has completed (`quote_reserve >= threshold`)
- [ ] Vesting period has passed (if applicable)
- [ ] Locker account created
- [ ] Migration metadata created
- [ ] Sufficient SOL for rent
- [ ] All necessary accounts prepared
- [ ] Partner/creator ready to receive LP tokens

After migration:

- [ ] Verify DEX pool created
- [ ] Verify LP tokens minted and distributed
- [ ] Claim unlocked LP tokens
- [ ] Withdraw surplus (if any)
- [ ] Withdraw migration fees
- [ ] Withdraw leftover tokens (if applicable)
- [ ] Lock LP tokens for vesting

---

## Error Cases

Common migration errors:

```rust
PoolError::PoolIsNotCompleted
// Pool hasn't reached threshold yet

PoolError::InsufficientLiquidityForMigration
// Not enough base tokens in vault

PoolError::PoolIsNotMigrated
// Trying to withdraw surplus before migration

PoolError::AlreadyWithdrawSurplus
// Surplus already claimed

PoolError::MigrationFeeAlreadyWithdrawn
// Migration fee already claimed
```

---

## Migration Timeline Example

```
Day 0, 00:00 - Pool created
    ↓
[Trading Phase - 7 days]
    ↓
Day 7, 15:30 - Last swap reaches threshold
              - EvtCurveComplete emitted
              - migration_progress = PostBondingCurve
    ↓
[Vesting Phase - 1 day cliff]
    ↓
Day 8, 15:30 - Cliff period expires
              - migration_progress = LockedVesting
              - Ready for migration
    ↓
Day 8, 16:00 - Someone creates locker
    ↓
Day 8, 16:05 - Someone creates metadata
    ↓
Day 8, 16:10 - Someone executes migration
              - Meteora pool created
              - LP tokens minted
              - is_migrated = 1
              - migration_progress = CreatedPool
    ↓
Day 8, 16:15 - LP tokens locked in vesting
    ↓
Day 8, 16:20 - Partner withdraws surplus
    ↓
Day 8, 16:25 - Creator withdraws surplus
    ↓
Day 8, 16:30 - Protocol withdraws surplus
    ↓
[LP Vesting Phase - 30 days]
    ↓
Day 9, 15:30 - First LP tokens unlock (cliff)
    ↓
Day 10, 15:30 - Period 1 LP tokens unlock
    ↓
...
    ↓
Day 38, 15:30 - Period 30 LP tokens unlock
              - All LP tokens fully vested
```
