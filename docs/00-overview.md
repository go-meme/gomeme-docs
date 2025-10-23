# Dynamic Bonding Curve - Overview

## What is it?

The Dynamic Bonding Curve is a Solana program that implements an automated market maker (AMM) with a bonding curve mechanism. It's designed for launching new tokens with a fair price discovery mechanism and automatic liquidity migration to established DEXs.

## Key Concepts

### 1. Bonding Curve
A bonding curve is a mathematical relationship between token price and supply. As more tokens are purchased, the price increases along a predetermined curve. This program implements a **piecewise constant liquidity** bonding curve, similar to Uniswap V3.

### 2. Virtual Pool
The core trading mechanism uses "virtual pools" - on-chain accounts that maintain:
- Base token reserves (the new token being launched)
- Quote token reserves (usually USDC, SOL, or another established token)
- Current price (stored as sqrt_price for mathematical precision)
- Fee accumulation and distribution

### 3. Dynamic Fees
The program implements a sophisticated fee system that combines:
- **Base fees**: Time-based fees that can decrease over time (linear or exponential decay)
- **Dynamic fees**: Volatility-based fees that increase during high trading activity
- **Rate limiter**: Anti-sniper protection that increases fees for rapid trading

### 4. Migration System
When a bonding curve completes (reaches its target), the accumulated liquidity can be migrated to:
- Meteora Dynamic AMM (DAMM V1)
- Meteora Dynamic AMM V2 (DAMM V2)

The migration includes LP token locking mechanisms to ensure long-term liquidity.

## Program Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Dynamic Bonding Curve Program               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Config     │  │ Virtual Pool │  │   Metadata   │     │
│  │              │  │              │  │              │     │
│  │ • Fee params │  │ • Reserves   │  │ • Partner    │     │
│  │ • Curve      │  │ • Price      │  │ • Creator    │     │
│  │ • Migration  │  │ • Fees       │  │ • Pool Info  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Instruction Handlers                     │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │ • Pool Creation & Initialization                     │  │
│  │ • Swap Operations (Exact In/Out, Partial Fill)       │  │
│  │ • Fee Management & Claims                            │  │
│  │ • Migration & Liquidity Locking                      │  │
│  │ • Admin Operations                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 Math & Curve Logic                    │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │ • Sqrt price calculations (x64.64 fixed-point)       │  │
│  │ • Liquidity distribution across price points         │  │
│  │ • Fee computation (base + dynamic + rate limiter)    │  │
│  │ • Slippage protection                                │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Key Features

1. **Fair Launch Mechanism**: Bonding curves ensure transparent price discovery
2. **Anti-Sniper Protection**: Rate limiters prevent front-running and rapid trading
3. **Dynamic Fee System**: Fees adapt to market volatility
4. **Automatic Migration**: Seamless transition to established DEXs
5. **Multi-Party Fee Distribution**: Fair distribution between protocol, creators, and partners
6. **Token Flexibility**: Supports both SPL Token and Token-2022
7. **Vesting & Locking**: Built-in token vesting and LP token locking mechanisms

## Stakeholders

### 1. Protocol (Admin)
- Collects protocol fees
- Can withdraw surplus after migration
- Manages claim operators

### 2. Partner
- Platform or service that integrates the bonding curve
- Receives share of trading fees and migration fees
- Can configure pool parameters
- Receives LP tokens after migration

### 3. Creator
- Token creator who launches the bonding curve
- Receives share of trading fees
- Can claim migration fees
- Receives LP tokens after migration
- Can transfer ownership

### 4. Traders
- Buy and sell tokens along the bonding curve
- Pay trading fees
- Can provide referral for fee sharing

## Program ID

```
dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN
```

## Next Steps

Continue reading:
- [01-instructions.md](./01-instructions.md) - Detailed instruction reference
- [02-state-accounts.md](./02-state-accounts.md) - Account structure documentation
- [03-bonding-curve-math.md](./03-bonding-curve-math.md) - Mathematical details
- [04-fee-system.md](./04-fee-system.md) - Fee calculation and distribution
- [05-workflows.md](./05-workflows.md) - Common workflows and operations
- [06-migration.md](./06-migration.md) - Migration process details
