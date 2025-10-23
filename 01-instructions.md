# Instructions Reference

This document provides a comprehensive list of all program instructions organized by category.

## Instruction Summary

The program contains **25 instructions** organized into 6 categories:

| Category | Instructions |
|----------|-------------|
| Admin | 4 instructions |
| Partner | 4 instructions |
| Creator | 5 instructions |
| Pool Initialization | 2 instructions |
| Trading | 2 instructions |
| Migration | 8 instructions |

---

## 1. Admin Instructions

### 1.1 `create_claim_fee_operator`

Creates an operator account that can claim protocol fees on behalf of the protocol.

**Parameters**: None

**Accounts**:
- `claim_fee_operator` - New claim operator PDA (mut)
- `operator` - Operator wallet (signer)
- `admin` - Protocol admin (signer)
- `system_program`

**Access**: Admin only

**Event**: `EvtCreateClaimFeeOperator`

---

### 1.2 `close_claim_fee_operator`

Closes a claim fee operator account.

**Parameters**: None

**Accounts**:
- `claim_fee_operator` - Existing operator PDA (mut)
- `operator` - Operator wallet
- `admin` - Protocol admin (signer)

**Access**: Admin only

**Event**: `EvtCloseClaimFeeOperator`

---

### 1.3 `claim_protocol_fee`

Claims accumulated protocol fees from a pool.

**Parameters**:
- `max_amount_a: u64` - Maximum base token to claim
- `max_amount_b: u64` - Maximum quote token to claim

**Accounts**:
- `claim_fee_operator` - Claim operator PDA
- `pool` - Virtual pool (mut)
- `base_vault` - Pool's base token vault (mut)
- `quote_vault` - Pool's quote token vault (mut)
- `fee_claimer_base_account` - Destination for base fees (mut)
- `fee_claimer_quote_account` - Destination for quote fees (mut)
- Token programs and authority

**Access**: Claim operator only

**Event**: `EvtClaimProtocolFee`

**Location**: `programs/dynamic-bonding-curve/src/instructions/admin/ix_claim_protocol_fee.rs`

---

### 1.4 `protocol_withdraw_surplus`

Withdraws surplus quote tokens after migration is complete.

**Parameters**: None

**Accounts**:
- `config` - Pool config
- `pool` - Virtual pool (mut)
- `quote_vault` - Pool's quote vault (mut)
- Various fee claimer accounts

**Access**: Protocol only

**Event**: `EvtProtocolWithdrawSurplus`

**Conditions**:
- Pool must be migrated
- Surplus must not have been withdrawn yet

**Location**: `programs/dynamic-bonding-curve/src/instructions/admin/ix_withdraw_protocol_surplus.rs`

---

## 2. Partner Instructions

### 2.1 `create_partner_metadata`

Creates partner metadata for tracking partner information.

**Parameters**:
- `metadata: CreatePartnerMetadataParameters`
  - `name: String`
  - `uri: String`

**Accounts**:
- `partner_metadata` - New metadata PDA (mut)
- `partner` - Partner wallet (signer)
- `system_program`

**Event**: `EvtPartnerMetadata`

**Location**: `programs/dynamic-bonding-curve/src/instructions/partner/ix_create_partner_metadata.rs`

---

### 2.2 `create_config`

Creates a new pool configuration. This is the first step before initializing a pool.

**Parameters**:
- `config_parameters: ConfigParameters` - Complex configuration object

**Config Parameters Include**:
- `pool_fees: PoolFeeParameters` - Fee configuration
- `creator_trading_fee_percentage: u8` - Creator's share of trading fees
- `token_update_authority: u8` - Token authority option
- `migration_fee: MigrationFee` - Migration fee settings
- `collect_fee_mode: u8` - QuoteToken or OutputToken
- `migration_option: u8` - MeteoraDamm or DammV2
- `activation_type: u8` - Slot or timestamp based
- `token_decimal: u8`
- `token_type: u8` - SplToken or Token2022
- Pool parameters (swap amounts, thresholds, etc.)
- `curve: Vec<LiquidityDistributionParameters>` - Bonding curve definition

**Accounts**:
- `config` - New config PDA (mut)
- `quote_mint` - Quote token mint
- `partner` - Partner wallet (signer, pays rent)
- `fee_claimer` - Fee recipient
- `leftover_receiver` - Leftover token recipient
- `system_program`

**Event**: `EvtCreateConfigV2`

**Location**: `programs/dynamic-bonding-curve/src/instructions/partner/ix_create_config.rs:48`

---

### 2.3 `claim_trading_fee`

Claims partner's share of accumulated trading fees.

**Parameters**:
- `max_amount_a: u64` - Max base tokens to claim
- `max_amount_b: u64` - Max quote tokens to claim

**Accounts**:
- `config` - Pool config
- `pool` - Virtual pool (mut)
- `partner` - Partner wallet (signer)
- Vault and token accounts for transfer

**Event**: `EvtClaimTradingFee`

**Location**: `programs/dynamic-bonding-curve/src/instructions/partner/ix_claim_partner_trading_fee.rs`

---

### 2.4 `partner_withdraw_surplus`

Withdraws partner's share of surplus after migration.

**Parameters**: None

**Accounts**:
- `config` - Pool config
- `pool` - Virtual pool (mut)
- `partner` - Partner wallet (signer)
- Token accounts for withdrawal

**Access**: Partner only

**Event**: `EvtPartnerWithdrawSurplus`

**Location**: `programs/dynamic-bonding-curve/src/instructions/partner/ix_withdraw_partner_surplus.rs`

---

## 3. Creator Instructions

### 3.1 `initialize_virtual_pool_with_spl_token`

Initializes a new virtual pool with an SPL token mint.

**Parameters**:
- `params: InitializePoolParameters`
  - `token_metadata: TokenMetadataInput` - Name, symbol, URI
  - `initial_base_amount: u64` - Initial base token amount

**Accounts** (19 accounts):
- `pool` - New virtual pool (mut)
- `config` - Pool configuration
- `creator` - Pool creator (signer, pays)
- `base_mint` - Base token mint (mut, created)
- `base_vault` - Base token vault (mut, created)
- `quote_vault` - Quote token vault (mut, created)
- Metaplex metadata accounts
- Token program and system program

**Event**: `EvtInitializePool`

**Location**: `programs/dynamic-bonding-curve/src/instructions/initialize_pool/ix_initialize_virtual_pool_with_spl_token.rs:44`

---

### 3.2 `initialize_virtual_pool_with_token2022`

Initializes a new virtual pool with a Token-2022 mint.

**Parameters**: Same as SPL token version

**Accounts**: Similar to SPL version but uses Token-2022 program

**Event**: `EvtInitializePool`

**Location**: `programs/dynamic-bonding-curve/src/instructions/initialize_pool/ix_initialize_virtual_pool_with_token2022.rs`

---

### 3.3 `create_virtual_pool_metadata`

Creates metadata for an existing virtual pool.

**Parameters**:
- `metadata: CreateVirtualPoolMetadataParameters`
  - `name: String`
  - `uri: String`

**Accounts**:
- `virtual_pool_metadata` - New metadata PDA (mut)
- `pool` - Virtual pool
- `creator` - Creator wallet (signer)
- `system_program`

**Event**: `EvtVirtualPoolMetadata`

**Location**: `programs/dynamic-bonding-curve/src/instructions/creator/ix_create_virtual_pool_metadata.rs`

---

### 3.4 `claim_creator_trading_fee`

Claims creator's share of accumulated trading fees.

**Parameters**:
- `max_base_amount: u64`
- `max_quote_amount: u64`

**Accounts**:
- `config` - Pool config
- `pool` - Virtual pool (mut)
- `creator` - Creator wallet (signer)
- Token accounts for transfer

**Event**: `EvtClaimCreatorTradingFee`

**Location**: `programs/dynamic-bonding-curve/src/instructions/creator/ix_claim_creator_trading_fee.rs`

---

### 3.5 `creator_withdraw_surplus`

Withdraws creator's share of surplus after migration.

**Parameters**: None

**Accounts**:
- `config` - Pool config
- `pool` - Virtual pool (mut)
- `creator` - Creator wallet (signer)
- Token accounts for withdrawal

**Event**: `EvtCreatorWithdrawSurplus`

**Location**: `programs/dynamic-bonding-curve/src/instructions/creator/ix_withdraw_creator_surplus.rs`

---

### 3.6 `transfer_pool_creator`

Transfers pool creator ownership to a new address.

**Parameters**: None

**Accounts**:
- `pool` - Virtual pool (mut)
- `creator` - Current creator (signer)
- `new_creator` - New creator wallet

**Access**: Current creator only

**Event**: `EvtUpdatePoolCreator`

**Location**: `programs/dynamic-bonding-curve/src/instructions/creator/ix_transfer_pool_creator.rs`

---

## 4. Trading Instructions

### 4.1 `swap`

Legacy swap instruction (exact input only).

**Parameters**:
- `params: SwapParameters`
  - `amount_in: u64` - Input token amount
  - `minimum_amount_out: u64` - Minimum output (slippage protection)

**Accounts**:
- `pool_authority` - PDA authority
- `config` - Pool config
- `pool` - Virtual pool (mut)
- `input_token_account` - User's input tokens (mut)
- `output_token_account` - User's output tokens (mut)
- `base_vault` - Pool's base vault (mut)
- `quote_vault` - Pool's quote vault (mut)
- `base_mint` - Base token mint
- `quote_mint` - Quote token mint
- `payer` - User wallet (signer)
- Token programs
- `referral_token_account` - Optional referral account (mut)

**Events**: `EvtSwap`, `EvtSwap2`, `EvtCurveComplete` (if curve completes)

**Location**: `programs/dynamic-bonding-curve/src/lib.rs:123`

---

### 4.2 `swap2`

Advanced swap instruction supporting multiple swap modes.

**Parameters**:
- `params: SwapParameters2`
  - `amount_0: u64` - Amount (meaning depends on swap_mode)
  - `amount_1: u64` - Limit amount (meaning depends on swap_mode)
  - `swap_mode: u8` - SwapMode enum value

**Swap Modes**:
- `ExactIn (0)`: Exact input amount, minimum output
- `PartialFill (1)`: Partial fill up to migration threshold
- `ExactOut (2)`: Exact output amount, maximum input

**Accounts**: Same as `swap`

**Events**: `EvtSwap`, `EvtSwap2`, `EvtCurveComplete`

**Special Features**:
- Rate limiter validation for anti-sniping
- Single swap instruction validation
- Automatic curve completion detection

**Location**: `programs/dynamic-bonding-curve/src/lib.rs:135`

**Handler**: `programs/dynamic-bonding-curve/src/instructions/swap/ix_swap.rs:119`

---

## 5. Migration Instructions

### 5.1 `create_locker`

Creates a locker account for locked LP tokens.

**Parameters**: None

**Accounts** (permissionless):
- `locker` - New locker PDA (mut)
- `pool` - Virtual pool
- `payer` - Transaction payer (signer)
- `system_program`

**Access**: Permissionless

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/create_locker.rs`

---

### 5.2 `withdraw_leftover`

Withdraws leftover base tokens after pool initialization.

**Parameters**: None

**Accounts**:
- `config` - Pool config
- `pool` - Virtual pool (mut)
- `leftover_receiver` - Receiver account (mut)
- `base_vault` - Pool's base vault (mut)
- Token accounts and programs

**Conditions**:
- Pool must be migrated
- Leftover not yet withdrawn

**Event**: `EvtWithdrawLeftover`

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/withdraw_leftover.rs`

---

### 5.3 `withdraw_migration_fee`

Withdraws migration fees for partner or creator.

**Parameters**:
- `flag: u8` - Bitflag (0b100 = partner, 0b010 = creator)

**Accounts**:
- `config` - Pool config
- `pool` - Virtual pool (mut)
- Various token accounts

**Event**: `EvtWithdrawMigrationFee`

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/ix_withdraw_migration_fee.rs`

---

### 5.4 Meteora DAMM Migration (4 instructions)

#### `migration_meteora_damm_create_metadata`
Creates migration metadata for Meteora DAMM.

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/meteora_damm/migration_meteora_damm_create_metadata.rs`

#### `migrate_meteora_damm`
Initializes the Meteora DAMM pool with migrated liquidity.

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/meteora_damm/migrate_meteora_damm_initialize_pool.rs`

#### `migrate_meteora_damm_lock_lp_token`
Locks LP tokens in the locker contract.

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/meteora_damm/meteora_damm_lock_lp_token.rs`

#### `migrate_meteora_damm_claim_lp_token`
Claims unlocked LP tokens from the locker.

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/meteora_damm/meteora_damm_claim_lp_token.rs`

---

### 5.5 DAMM V2 Migration (2 instructions)

#### `migration_damm_v2_create_metadata`
Creates migration metadata for DAMM V2.

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/dynamic_amm_v2/migration_damm_v2_create_metadata.rs`

#### `migration_damm_v2`
Initializes the DAMM V2 pool with migrated liquidity.

**Location**: `programs/dynamic-bonding-curve/src/instructions/migration/dynamic_amm_v2/migrate_damm_v2_initialize_pool.rs`

---

## Instruction Flow Chart

```
┌─────────────────────────────────────────────────────────────┐
│                    Instruction Lifecycle                     │
└─────────────────────────────────────────────────────────────┘

1. SETUP PHASE
   ┌──────────────────────┐
   │ create_config        │ (Partner)
   └──────────────────────┘
            │
            ▼
   ┌──────────────────────┐
   │ initialize_pool_*    │ (Creator)
   └──────────────────────┘

2. TRADING PHASE
            │
            ▼
   ┌──────────────────────┐
   │ swap / swap2         │ (Traders)
   │ (repeated)           │
   └──────────────────────┘
            │
            ▼
   ┌──────────────────────┐
   │ claim_trading_fee    │ (Partner/Creator - optional)
   └──────────────────────┘

3. COMPLETION PHASE
            │
            ▼
   ┌──────────────────────┐
   │ Curve Complete       │ (Automatic on last swap)
   └──────────────────────┘

4. MIGRATION PHASE
            │
            ▼
   ┌──────────────────────┐
   │ create_locker        │ (Permissionless)
   └──────────────────────┘
            │
            ▼
   ┌──────────────────────┐
   │ migration_*_create_  │ (Permissionless)
   │ metadata             │
   └──────────────────────┘
            │
            ▼
   ┌──────────────────────┐
   │ migrate_*            │ (Permissionless)
   └──────────────────────┘
            │
            ▼
   ┌──────────────────────┐
   │ migrate_*_lock_lp    │ (Permissionless)
   └──────────────────────┘

5. POST-MIGRATION
            │
            ▼
   ┌──────────────────────┐
   │ withdraw_surplus     │ (Partner/Creator/Protocol)
   │ withdraw_migration_  │
   │ fee                  │
   │ withdraw_leftover    │
   └──────────────────────┘
```

## Access Control Summary

| Instruction | Access |
|-------------|--------|
| create_claim_fee_operator | Admin only |
| close_claim_fee_operator | Admin only |
| claim_protocol_fee | Claim operator only |
| protocol_withdraw_surplus | Protocol only |
| create_partner_metadata | Anyone |
| create_config | Partner (pays rent) |
| claim_trading_fee | Partner only |
| partner_withdraw_surplus | Partner only |
| initialize_pool_* | Creator (pays rent) |
| create_virtual_pool_metadata | Creator only |
| claim_creator_trading_fee | Creator only |
| creator_withdraw_surplus | Creator only |
| transfer_pool_creator | Current creator only |
| swap / swap2 | Anyone |
| create_locker | Permissionless |
| withdraw_leftover | Anyone (sends to designated receiver) |
| withdraw_migration_fee | Partner or Creator (based on flag) |
| Migration instructions | Permissionless (but validated) |
