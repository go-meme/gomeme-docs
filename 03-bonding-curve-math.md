# Bonding Curve Mathematics

This document explains the mathematical foundations of the dynamic bonding curve implementation.

## Overview

The program implements a **concentrated liquidity AMM** with a piecewise constant product formula, similar to Uniswap V3. The bonding curve is divided into price ranges, each with its own liquidity parameter.

## Core Concepts

### 1. Sqrt Price Representation

Prices are stored as the **square root of the actual price** in a **Q64.64 fixed-point format**:

```
sqrt_price = √(price_quote_per_base) * 2^64
```

**Why sqrt price?**
- More numerically stable for calculations
- Symmetric price movements (1% up = 1% down)
- Avoids precision loss in extreme price ranges

**Example**:
```
Actual price: 1 USDC per TOKEN
sqrt_price = √1 * 2^64 = 18,446,744,073,709,551,616

Actual price: 100 USDC per TOKEN
sqrt_price = √100 * 2^64 = 184,467,440,737,095,516,160
```

**Price Bounds**:
```rust
MIN_SQRT_PRICE: u128 = 4,295,048,016              // ~$0.000001 per token
MAX_SQRT_PRICE: u128 = 79,226,673,521,066,979,... // ~$1,000,000 per token
```

**Location**: `programs/dynamic-bonding-curve/src/constants.rs:3-4`

---

### 2. Liquidity

Liquidity `L` is a constant that determines the relationship between price and reserves:

```
L = √(x * y)
```

Where:
- `x` = base token reserve
- `y` = quote token reserve

For a price range `[P_lower, P_upper]` with liquidity `L`:
```
L = constant for all prices in this range
```

---

## Bonding Curve Definition

### Piecewise Constant Liquidity

The curve is defined by up to **20 price points**, creating up to 20 segments:

```rust
pub struct LiquidityDistributionConfig {
    pub sqrt_price: u128,    // Price point
    pub liquidity: u128,     // Liquidity for segment ending at this price
}

pub curve: [LiquidityDistributionConfig; 20]
```

**Curve Structure**:
```
Segment 0: [sqrt_start_price, curve[0].sqrt_price] → liquidity = curve[0].liquidity
Segment 1: [curve[0].sqrt_price, curve[1].sqrt_price] → liquidity = curve[1].liquidity
...
Segment i: [curve[i-1].sqrt_price, curve[i].sqrt_price] → liquidity = curve[i].liquidity
```

**Visual Example**:
```
Price
  │
  │                    ┌─────────────
  │                   ─┘
  │              ┌───┘               Segment 2: L₂
  │             ─┘
  │        ┌───┘
  │       ─┘                         Segment 1: L₁
  │  ┌───┘
  │ ─┘                               Segment 0: L₀
  │
  └────────────────────────────────► Reserves
    start  P₀    P₁    P₂    P₃
```

**Location**: `programs/dynamic-bonding-curve/src/state/config.rs:559-564`

---

## Core Formulas

### 1. Constant Product Formula (per segment)

Within each liquidity segment:

```
x * √P = L     (for base token)
y / √P = L     (for quote token)
```

Where:
- `x` = base token amount
- `y` = quote token amount
- `P` = price (quote per base)
- `L` = liquidity constant

This gives us:
```
x * y = L²
```

---

### 2. Token Amount Calculations

#### Base Token Amount (Δx)

For a price range `[√P_lower, √P_upper]`:

```
Δx = L * (√P_upper - √P_lower) / (√P_upper * √P_lower)
   = L * (1/√P_lower - 1/√P_upper)
```

**Implementation**:
```rust
pub fn get_delta_amount_base_unsigned(
    lower_sqrt_price: u128,
    upper_sqrt_price: u128,
    liquidity: u128,
    round: Rounding,
) -> Result<u64> {
    let numerator_1 = U256::from(liquidity);
    let numerator_2 = U256::from(upper_sqrt_price - lower_sqrt_price);
    let denominator = U256::from(lower_sqrt_price) * U256::from(upper_sqrt_price);

    mul_div_u256(numerator_1, numerator_2, denominator, round)
}
```

**Location**: `programs/dynamic-bonding-curve/src/curve.rs:62-72`

---

#### Quote Token Amount (Δy)

For a price range `[√P_lower, √P_upper]`:

```
Δy = L * (√P_upper - √P_lower)
```

**Implementation**:
```rust
pub fn get_delta_amount_quote_unsigned(
    lower_sqrt_price: u128,
    upper_sqrt_price: u128,
    liquidity: u128,
    round: Rounding,
) -> Result<u64> {
    let liquidity = U256::from(liquidity);
    let delta_sqrt_price = U256::from(upper_sqrt_price - lower_sqrt_price);
    let prod = liquidity * delta_sqrt_price;

    // Divide by 2^128 (Q64.64 * Q64.64 = Q128.128)
    match round {
        Rounding::Up => prod.div_ceil(2^128),
        Rounding::Down => prod >> 128,
    }
}
```

**Location**: `programs/dynamic-bonding-curve/src/curve.rs:109-158`

---

### 3. Next Price Calculations

#### From Input Amount

When adding tokens, find the new price:

**Base Token Input** (selling base for quote):
```
√P' = √P * L / (L + Δx * √P)
```

**Quote Token Input** (buying base with quote):
```
√P' = √P + Δy / L
```

**Implementation**:
```rust
pub fn get_next_sqrt_price_from_input(
    sqrt_price: u128,
    liquidity: u128,
    amount_in: u64,
    base_for_quote: bool,  // true if selling base
) -> Result<u128> {
    if base_for_quote {
        // √P' = √P * L / (L + Δx * √P)
        get_next_sqrt_price_from_base_amount_in_rounding_up(...)
    } else {
        // √P' = √P + Δy / L
        get_next_sqrt_price_from_quote_amount_in_rounding_down(...)
    }
}
```

**Location**: `programs/dynamic-bonding-curve/src/curve.rs:162-177`

---

#### From Output Amount

When removing tokens, find the new price:

**Base Token Output** (buying base):
```
√P' = √P * L / (L - Δx * √P)
```

**Quote Token Output** (selling base):
```
√P' = √P - Δy / L
```

**Implementation**:
```rust
pub fn get_next_sqrt_price_from_output(
    sqrt_price: u128,
    liquidity: u128,
    amount_out: u64,
    base_for_quote: bool,
) -> Result<u128> {
    if base_for_quote {
        // √P' = √P - Δy / L
        get_next_sqrt_price_from_quote_amount_out_rounding_down(...)
    } else {
        // √P' = √P * L / (L - Δx * √P)
        get_next_sqrt_price_from_base_amount_out_rounding_up(...)
    }
}
```

**Location**: `programs/dynamic-bonding-curve/src/curve.rs:179-193`

---

## Swap Calculations

### Exact Input Swap (Quote → Base)

**Goal**: Given quote token input, calculate base token output

**Algorithm**:
```
1. Start at current price P₀
2. For each liquidity segment (ascending):
   a. Calculate max quote tokens to reach segment end price
   b. If remaining input < max:
      - Calculate new price within segment
      - Calculate base token output
      - Done
   c. Else:
      - Move to segment end price
      - Calculate base token output for this segment
      - Subtract used input
      - Continue to next segment
3. Return total base output and final price
```

**Implementation**:
```rust
fn calculate_quote_to_base_from_amount_in(
    &self,
    config: &PoolConfig,
    amount_in: u64,
    stop_sqrt_price: u128,
) -> Result<SwapAmountFromInput> {
    let mut total_output_amount = 0u64;
    let mut current_sqrt_price = self.sqrt_price;
    let mut amount_left = amount_in;

    for i in 0..config.curve.len() {
        if config.curve[i].sqrt_price == 0 { break; }

        let reference_sqrt_price = stop_sqrt_price.min(config.curve[i].sqrt_price);
        if reference_sqrt_price > current_sqrt_price {
            // Calculate max input to reach reference price
            let max_amount_in = get_delta_amount_quote_unsigned_256(
                current_sqrt_price,
                reference_sqrt_price,
                config.curve[i].liquidity,
                Rounding::Up,
            )?;

            if U256::from(amount_left) < max_amount_in {
                // Can't reach reference price, calculate within segment
                let next_sqrt_price = get_next_sqrt_price_from_input(
                    current_sqrt_price,
                    config.curve[i].liquidity,
                    amount_left,
                    false,  // quote to base
                )?;

                let output_amount = get_delta_amount_base_unsigned(
                    current_sqrt_price,
                    next_sqrt_price,
                    config.curve[i].liquidity,
                    Rounding::Down,
                )?;

                total_output_amount += output_amount;
                current_sqrt_price = next_sqrt_price;
                amount_left = 0;
                break;
            } else {
                // Move to reference price
                let next_sqrt_price = reference_sqrt_price;
                let output_amount = get_delta_amount_base_unsigned(
                    current_sqrt_price,
                    next_sqrt_price,
                    config.curve[i].liquidity,
                    Rounding::Down,
                )?;

                total_output_amount += output_amount;
                current_sqrt_price = next_sqrt_price;
                amount_left -= max_amount_in.try_into()?;

                if next_sqrt_price == stop_sqrt_price { break; }
            }
        }
    }

    Ok(SwapAmountFromInput {
        amount_left,
        output_amount: total_output_amount,
        next_sqrt_price: current_sqrt_price,
    })
}
```

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool.rs:740-818`

---

### Exact Output Swap (Quote → Base)

**Goal**: Given desired base token output, calculate required quote token input

**Algorithm**:
```
1. Start at current price P₀
2. For each liquidity segment (ascending):
   a. Calculate max base tokens available to segment end price
   b. If remaining output < max:
      - Calculate new price within segment
      - Calculate required quote input
      - Done
   c. Else:
      - Move to segment end price
      - Calculate quote input for this segment
      - Subtract provided output
      - Continue to next segment
3. Return total quote input and final price
```

**Implementation**:
```rust
pub fn calculate_quote_to_base_from_amount_out(
    &self,
    config: &PoolConfig,
    amount_out: u64,
) -> Result<SwapAmountFromOutput> {
    let mut total_input_amount = 0u64;
    let mut amount_left = amount_out;
    let mut current_sqrt_price = self.sqrt_price;

    for i in 0..config.curve.len() {
        if config.curve[i].sqrt_price == 0 { break; }

        if config.curve[i].sqrt_price > current_sqrt_price {
            // Max base tokens to this price point
            let max_amount_out = get_delta_amount_base_unsigned_256(
                current_sqrt_price,
                config.curve[i].sqrt_price,
                config.curve[i].liquidity,
                Rounding::Down,
            )?;

            if U256::from(amount_left) < max_amount_out {
                // Find price for exact output
                let next_sqrt_price = get_next_sqrt_price_from_output(
                    current_sqrt_price,
                    config.curve[i].liquidity,
                    amount_left,
                    false,  // false = base output
                )?;

                let input_amount = get_delta_amount_quote_unsigned(
                    current_sqrt_price,
                    next_sqrt_price,
                    config.curve[i].liquidity,
                    Rounding::Up,
                )?;

                total_input_amount += input_amount;
                current_sqrt_price = next_sqrt_price;
                amount_left = 0;
                break;
            } else {
                // Use entire segment
                let next_sqrt_price = config.curve[i].sqrt_price;
                let input_amount = get_delta_amount_quote_unsigned(
                    current_sqrt_price,
                    next_sqrt_price,
                    config.curve[i].liquidity,
                    Rounding::Up,
                )?;

                total_input_amount += input_amount;
                current_sqrt_price = next_sqrt_price;
                amount_left -= max_amount_out.try_into()?;
            }
        }
    }

    require!(amount_left == 0, PoolError::AmountLeftIsNotZero);

    Ok(SwapAmountFromOutput {
        amount_in: total_input_amount,
        next_sqrt_price: current_sqrt_price,
    })
}
```

**Location**: `programs/dynamic-bonding-curve/src/state/virtual_pool.rs:378-442`

---

## Rounding

Proper rounding is critical for preventing value extraction:

### Rounding Rules

```rust
pub enum Rounding {
    Up,    // Round in favor of the pool
    Down,  // Round in favor of the user
}
```

**Principles**:
1. **Input amounts**: Round UP (user pays more)
2. **Output amounts**: Round DOWN (user receives less)
3. **Fees**: Round UP (pool receives more)

**Examples**:
```rust
// When calculating required input (user pays)
get_delta_amount_quote_unsigned(..., Rounding::Up)

// When calculating output (user receives)
get_delta_amount_base_unsigned(..., Rounding::Down)

// When calculating next price from input
get_next_sqrt_price_from_base_amount_in_rounding_up(...)  // Round price UP
```

**Location**: `programs/dynamic-bonding-curve/src/math/u128x128_math.rs`

---

## Example Calculation

### Setup
```
Current price: √P = 2^64 (P = 1 USDC per TOKEN)
Liquidity: L = 10,000 * 2^64
Input: 100 USDC
```

### Step 1: Calculate new price
```
Δy = 100 * 2^128 (convert to Q64.64 * Q64.64)
√P' = √P + Δy / L
    = 2^64 + (100 * 2^128) / (10,000 * 2^64)
    = 2^64 + 100 * 2^64 / 10,000
    = 2^64 * (1 + 0.01)
    = 2^64 * 1.01
```

### Step 2: Calculate base output
```
Δx = L * (1/√P - 1/√P')
   = L * (1/(2^64) - 1/(2^64 * 1.01))
   = L * (1 - 1/1.01) / 2^64
   ≈ 99 TOKEN (rounded down)
```

### Result
```
Input:  100 USDC
Output: 99 TOKEN
Price change: 1.00 → 1.01 USDC per TOKEN (+1%)
```

---

## Numerical Precision

### Fixed-Point Arithmetic

All calculations use **Q64.64 fixed-point** format:
```
value = integer_part * 2^64 + fractional_part
```

**Range**:
- Integer part: 0 to 2^64-1
- Fractional part: 0 to 2^64-1
- Precision: ~18 decimal places

### Overflow Protection

The program uses `U256` for intermediate calculations:

```rust
use ruint::aliases::U256;

// Example: Multiplying two u128 values
let a = U256::from(value1);
let b = U256::from(value2);
let product = a.safe_mul(b)?;  // Can't overflow U256
let result = product.try_into()?;  // Checked conversion back
```

**Location**: `programs/dynamic-bonding-curve/src/math/safe_math.rs`

---

## Constants and Limits

```rust
// Curve configuration
pub const MAX_CURVE_POINT_CONFIG: usize = 20;

// Price bounds (Q64.64 format)
pub const MIN_SQRT_PRICE: u128 = 4_295_048_016;
pub const MAX_SQRT_PRICE: u128 = 79_226_673_521_066_979_257_578_248_091;

// Fixed-point precision
pub const RESOLUTION: u8 = 64;
pub const ONE_Q64: u128 = 1u128 << 64;
```

**Location**: `programs/dynamic-bonding-curve/src/constants.rs`

---

## Further Reading

- [Uniswap V3 Whitepaper](https://uniswap.org/whitepaper-v3.pdf) - Similar concentrated liquidity design
- [Q64.64 Fixed-Point](https://en.wikipedia.org/wiki/Q_(number_format)) - Number format explanation
