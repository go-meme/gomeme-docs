# Dynamic Bonding Curve Documentation

Comprehensive documentation for the Dynamic Bonding Curve Solana program.

## Program Overview

The Dynamic Bonding Curve is a Solana program implementing an automated market maker (AMM) with:
- **Piecewise constant liquidity** bonding curves
- **Dynamic fee system** with volatility-based adjustments
- **Anti-sniper protection** via rate limiters
- **Automatic liquidity migration** to Meteora DEX
- **Multi-party fee distribution** (protocol, partners, creators, referrals)

**Program ID**: `dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN`

---

## Documentation Structure

### Quick Start

**New to bonding curves?** Start here:
1. [00-overview.md](./00-overview.md) - High-level introduction
2. [05-workflows.md](./05-workflows.md) - Common operations and examples
3. [01-instructions.md](./01-instructions.md) - Instruction reference

### Core Documentation

| File | Description | For Who |
|------|-------------|---------|
| [**00-overview.md**](./00-overview.md) | Program overview, architecture, and key concepts | Everyone |
| [**01-instructions.md**](./01-instructions.md) | Complete instruction reference (25 instructions) | Developers |
| [**02-state-accounts.md**](./02-state-accounts.md) | Account structures and PDAs | Developers |
| [**03-bonding-curve-math.md**](./03-bonding-curve-math.md) | Mathematical formulas and calculations | Advanced developers |
| [**04-fee-system.md**](./04-fee-system.md) | Fee mechanisms and distribution | Integrators, traders |
| [**05-workflows.md**](./05-workflows.md) | Step-by-step guides and code examples | Developers |
| [**06-migration.md**](./06-migration.md) | Migration system and liquidity locking | Advanced users |

---

## Key Features

### 1. Bonding Curve

- Up to **20 price points** for flexible curve design
- **Concentrated liquidity** model (Uniswap V3-style)
- **Q64.64 fixed-point** math for precision
- Support for any price range ($0.000001 to $1,000,000)

**Learn more**: [03-bonding-curve-math.md](./03-bonding-curve-math.md)

### 2. Dynamic Fees

- **Base fees**: Decrease over time (linear or exponential)
- **Dynamic fees**: Increase with volatility
- **Rate limiter**: Anti-sniper protection
- **Total fee**: Capped at 99%

**Learn more**: [04-fee-system.md](./04-fee-system.md)

### 3. Multi-Party Distribution

```
Total Fee (100%)
├─ Protocol (20%)
│  ├─ Protocol (80%)
│  └─ Referral (20%)
└─ Trading (80%)
   ├─ Partner (50%)
   └─ Creator (50%)
```

**Learn more**: [04-fee-system.md#4-fee-distribution](./04-fee-system.md#4-fee-distribution)

### 4. Migration System

- Automatic migration to **Meteora DAMM** or **DAMM V2**
- **LP token vesting** with customizable schedules
- **Surplus distribution** (80% to partners/creators, 20% to protocol)
- **Migration fees** configurable (0.25% to 6%)

**Learn more**: [06-migration.md](./06-migration.md)

---

## Instruction Categories

### Admin (4 instructions)
- `create_claim_fee_operator` - Authorize fee claimer
- `close_claim_fee_operator` - Revoke authorization
- `claim_protocol_fee` - Claim protocol fees
- `protocol_withdraw_surplus` - Withdraw post-migration surplus

### Partner (4 instructions)
- `create_partner_metadata` - Create partner info
- `create_config` - Create pool configuration
- `claim_trading_fee` - Claim partner trading fees
- `partner_withdraw_surplus` - Withdraw surplus

### Creator (5 instructions)
- `initialize_virtual_pool_with_spl_token` - Create pool (SPL)
- `initialize_virtual_pool_with_token2022` - Create pool (Token-2022)
- `create_virtual_pool_metadata` - Add pool metadata
- `claim_creator_trading_fee` - Claim creator fees
- `creator_withdraw_surplus` - Withdraw surplus
- `transfer_pool_creator` - Transfer ownership

### Trading (2 instructions)
- `swap` - Legacy exact-in swap
- `swap2` - Advanced swap (exact-in, exact-out, partial-fill)

### Migration (8 instructions)
- `create_locker` - Create LP token locker
- `withdraw_leftover` - Withdraw excess base tokens
- `withdraw_migration_fee` - Claim migration fees
- Meteora DAMM migration (4 instructions)
- DAMM V2 migration (2 instructions)

**Full reference**: [01-instructions.md](./01-instructions.md)

---

## State Accounts

| Account | Size | Purpose |
|---------|------|---------|
| **PoolConfig** | 1040 bytes | Immutable pool configuration |
| **VirtualPool** | 416 bytes | Mutable pool state |
| **PartnerMetadata** | Variable | Partner information |
| **VirtualPoolMetadata** | Variable | Pool metadata |
| **ClaimFeeOperator** | 40 bytes | Fee claim authorization |

**Full reference**: [02-state-accounts.md](./02-state-accounts.md)

---

## Common Workflows

### Creating a Pool

```typescript
// 1. Partner: Create configuration
await program.methods.createConfig(params)...

// 2. Creator: Initialize pool
await program.methods.initializeVirtualPoolWithSplToken(metadata)...
```

**Guide**: [05-workflows.md#1-pool-creation-workflow](./05-workflows.md#1-pool-creation-workflow)

### Trading

```typescript
// Buy tokens with USDC
await program.methods.swap2({
    amount0: inputAmount,      // USDC in
    amount1: minOutputAmount,  // Min tokens out
    swapMode: 0,               // ExactIn
})...
```

**Guide**: [05-workflows.md#2-trading-workflow](./05-workflows.md#2-trading-workflow)

### Migration

```typescript
// 1. Create locker
await program.methods.createLocker()...

// 2. Create metadata
await program.methods.migrationDammV2CreateMetadata()...

// 3. Execute migration
await program.methods.migrationDammV2()...
```

**Guide**: [05-workflows.md#4-migration-workflow](./05-workflows.md#4-migration-workflow)

---

## Mathematical Details

### Constant Product Formula

For each liquidity segment:
```
x * y = L²
```

Where:
- `x` = base token amount
- `y` = quote token amount
- `L` = liquidity constant

### Price Calculation

Prices stored as **sqrt(price)** in **Q64.64 format**:
```
sqrt_price = √(price_quote_per_base) * 2^64
```

### Token Amount Formulas

**Base tokens**:
```
Δx = L * (1/√P_lower - 1/√P_upper)
```

**Quote tokens**:
```
Δy = L * (√P_upper - √P_lower)
```

**Full details**: [03-bonding-curve-math.md](./03-bonding-curve-math.md)

---

## Fee Calculations

### Base Fee (Time-Decay)

**Linear**:
```
fee(t) = cliff_fee - (periods * reduction_factor)
```

**Exponential**:
```
fee(t) = cliff_fee * (1 - reduction_factor/10000)^periods
```

### Dynamic Fee (Volatility)

```
variable_fee = (volatility * bin_step)^2 * control / 10^11
total_fee = base_fee + variable_fee
```

### Rate Limiter (Anti-Sniper)

```
penalty = (volume / reference) * increment_bps
effective_fee = base_fee + penalty
```

**Full details**: [04-fee-system.md](./04-fee-system.md)

---

## Program Architecture

```
programs/
└── dynamic-bonding-curve/
    ├── src/
    │   ├── lib.rs                    # Program entry point (25 instructions)
    │   ├── instructions/              # Instruction handlers
    │   │   ├── admin/                # Admin operations
    │   │   ├── partner/              # Partner operations
    │   │   ├── creator/              # Creator operations
    │   │   ├── swap/                 # Trading logic
    │   │   ├── initialize_pool/      # Pool creation
    │   │   └── migration/            # Migration system
    │   ├── state/                    # Account structures
    │   │   ├── config.rs             # PoolConfig (1040 bytes)
    │   │   ├── virtual_pool.rs       # VirtualPool (416 bytes)
    │   │   ├── fee.rs                # Fee tracking
    │   │   └── ...
    │   ├── math/                     # Mathematical operations
    │   │   ├── safe_math.rs          # Overflow protection
    │   │   ├── u128x128_math.rs      # Fixed-point math
    │   │   └── fee_math.rs           # Fee calculations
    │   ├── curve.rs                  # Bonding curve logic
    │   ├── base_fee/                 # Fee schedulers
    │   │   ├── fee_scheduler.rs      # Time-based fees
    │   │   └── fee_rate_limiter.rs   # Anti-sniper
    │   ├── error.rs                  # Error definitions
    │   ├── event.rs                  # Event definitions
    │   └── constants.rs              # Program constants
    └── Cargo.toml

libs/
├── damm-v2/                          # Meteora DAMM V2 integration
├── dynamic-amm/                      # Meteora DAMM integration
└── locker/                           # LP token vesting
```

---

## Constants

### Price Bounds
```rust
MIN_SQRT_PRICE: u128 = 4_295_048_016              // ~$0.000001
MAX_SQRT_PRICE: u128 = 79_226_673_521_066_979_... // ~$1,000,000
```

### Fee Limits
```rust
FEE_DENOMINATOR: u64 = 1_000_000_000   // 9 decimals
MAX_FEE_BPS: u64 = 9900                // 99%
MIN_FEE_BPS: u64 = 1                   // 0.01%
PROTOCOL_FEE_PERCENT: u8 = 20          // 20%
```

### Curve Configuration
```rust
MAX_CURVE_POINT_CONFIG: usize = 20     // Max price points
SWAP_BUFFER_PERCENTAGE: u8 = 25        // 25% buffer
```

### Distribution
```rust
PARTNER_AND_CREATOR_SURPLUS_SHARE: u8 = 80  // 80%
```

---

## Events

The program emits events for all major operations:

| Event | Emitted When |
|-------|--------------|
| `EvtCreateConfig` | Pool config created |
| `EvtInitializePool` | Pool initialized |
| `EvtSwap`, `EvtSwap2` | Swap executed |
| `EvtCurveComplete` | Bonding curve completes |
| `EvtClaimProtocolFee` | Protocol fees claimed |
| `EvtClaimTradingFee` | Partner fees claimed |
| `EvtClaimCreatorTradingFee` | Creator fees claimed |
| `EvtProtocolWithdrawSurplus` | Protocol surplus withdrawn |
| `EvtPartnerWithdrawSurplus` | Partner surplus withdrawn |
| `EvtCreatorWithdrawSurplus` | Creator surplus withdrawn |
| `EvtWithdrawMigrationFee` | Migration fee withdrawn |
| `EvtWithdrawLeftover` | Leftover tokens withdrawn |
| `EvtUpdatePoolCreator` | Creator transferred |

---

## Error Codes

Common errors:

| Error | Description |
|-------|-------------|
| `PoolIsCompleted` | Cannot swap, curve complete |
| `InsufficientLiquidityForMigration` | Not enough base tokens |
| `PoolIsNotCompleted` | Migration not allowed yet |
| `AmountIsZero` | Invalid zero amount |
| `AmountLeftIsNotZero` | Exact output not achievable |
| `MathOverflow` | Calculation overflow |
| `InvalidCollectFeeMode` | Invalid fee mode |
| `InvalidBaseFeeMode` | Invalid fee scheduler |
| `FailToValidateSingleSwapInstruction` | Multiple swaps detected |

**Full list**: See `programs/dynamic-bonding-curve/src/error.rs`

---

## SDK and Integration

### TypeScript SDK

Located in `dynamic-bonding-curve-sdk/`:

```typescript
import {
    quoteExactIn,
    quoteExactOut,
    quotePartialFill,
} from 'dynamic-bonding-curve-sdk';

// Simulate swap
const quote = quoteExactIn({
    pool,
    config,
    amountIn: 1000000,
    tradeDirection: 'QuoteToBase',
});

console.log(quote.outputAmount);
console.log(quote.priceImpact);
```

### PDA Helpers

```typescript
import { PublicKey } from '@solana/web3.js';

const PROGRAM_ID = new PublicKey('dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN');

// Pool PDA
const [poolPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from('pool'), baseMint.toBuffer(), config.toBuffer()],
    PROGRAM_ID
);

// Config PDA
const [configPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from('config'), configIndexBuffer],
    PROGRAM_ID
);

// Vault PDA
const [vaultPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from('token_vault'), pool.toBuffer(), mint.toBuffer()],
    PROGRAM_ID
);
```

---

## Testing

Run program tests:

```bash
# Build program
anchor build

# Run tests
anchor test

# Run specific test
anchor test -- --test test_swap
```

Test files located in `programs/dynamic-bonding-curve/src/tests/`:
- `test_swap.rs` - Swap functionality
- `test_create_config.rs` - Configuration
- `test_dynamic_fee_params.rs` - Dynamic fees
- `test_rate_limiter.rs` - Anti-sniper
- `price_math.rs` - Price calculations

---

## Security Considerations

### Rounding
- All input amounts: **round UP** (pool receives more)
- All output amounts: **round DOWN** (user receives less)
- All fees: **round UP** (pool receives more)

### Overflow Protection
- Uses `SafeMath` trait for all arithmetic
- `U256` for intermediate calculations
- Checked conversions everywhere

### Access Control
- Admin operations: `admin` signer required
- Fee claims: Authorized parties only
- Pool creation: Anyone can create
- Migration: Permissionless after completion

### Slippage Protection
- `minimum_amount_out` for exact-in swaps
- `maximum_amount_in` for exact-out swaps
- Price impact warnings in SDK

---

## Resources

### External Documentation
- [Meteora DAMM Docs](https://docs.meteora.ag/dlmm/overview)
- [Uniswap V3 Whitepaper](https://uniswap.org/whitepaper-v3.pdf)
- [Anchor Documentation](https://www.anchor-lang.com/)
- [Solana Documentation](https://docs.solana.com/)

### Related Programs
- Meteora DAMM: Dynamic AMM for migrated liquidity
- Locker: LP token vesting
- Token Metadata: Metaplex token metadata

---

## Version History

- **v0.1.6** - Latest release
- **v0.1.5** - Fee improvements
- **v0.1.4** - Migration updates
- **v0.1.3** - Creator features
- Earlier versions - See git history

---

## License

See repository root for license information.

---

## Support

For questions and issues:
- GitHub Issues: [Repository issues](https://github.com/solpump/dynamic-bonding-curve/issues)
- Documentation: This directory
- Code examples: `tests/` directory

---

## Quick Reference Card

### Essential PDAs
```
Pool:     ["pool", base_mint, config]
Config:   ["config", config_index]
Vault:    ["token_vault", pool, mint]
Authority: ["pool_authority"]
```

### Common Operations
```typescript
// Create pool
createConfig() → initializePool()

// Trade
swap2({ amount0, amount1, swapMode: 0 })

// Claim fees
claimTradingFee(maxBase, maxQuote)

// Migrate
createLocker() → createMetadata() → migrate()
```

### Important Limits
```
Max curve points: 20
Max fee: 99%
Min fee: 0.01%
Price range: $0.000001 to $1,000,000
```

---

**Last Updated**: 2025-10-23
