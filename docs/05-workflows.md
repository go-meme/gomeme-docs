# Workflows and Common Operations

This document describes common workflows and step-by-step processes for using the Dynamic Bonding Curve program.

## Table of Contents

1. [Pool Creation Workflow](#1-pool-creation-workflow)
2. [Trading Workflow](#2-trading-workflow)
3. [Fee Management Workflow](#3-fee-management-workflow)
4. [Migration Workflow](#4-migration-workflow)
5. [Administrative Operations](#5-administrative-operations)

---

## 1. Pool Creation Workflow

### Overview

Creating a new bonding curve pool involves two steps:
1. Create a pool configuration
2. Initialize the virtual pool with a token

### 1.1 Create Pool Configuration

**Who**: Partner (platform)

**Instruction**: `create_config`

**Steps**:

```typescript
// 1. Define the bonding curve
const curve = [
    { sqrtPrice: "184467440737095516160", liquidity: "1000000000000000000" },  // Point 1
    { sqrtPrice: "260731679438094071296", liquidity: "2000000000000000000" },  // Point 2
    { sqrtPrice: "368894881152183001088", liquidity: "3000000000000000000" },  // Point 3
    // ... up to 20 points
];

// 2. Configure fees
const poolFees = {
    baseFee: {
        cliffFeeNumerator: 10_000_000,  // 1% initial fee
        baseFeeMode: 0,                  // Linear scheduler
        firstFactor: 100,                // 100 periods
        secondFactor: 60,                // 60 seconds per period
        thirdFactor: 100_000,            // Reduce by 0.01% per period
    },
    dynamicFee: {
        initialized: 1,
        maxVolatilityAccumulator: 14_460_000,
        variableFeeControl: 100_000,
        binStep: 1,
        filterPeriod: 10,
        decayPeriod: 120,
        reductionFactor: 5000,
    },
    protocolFeePercent: 20,  // 20% to protocol
    referralFeePercent: 20,  // 20% of protocol to referral
};

// 3. Configure pool parameters
const configParams = {
    poolFees,
    creatorTradingFeePercentage: 50,  // 50% of trading fees to creator
    tokenUpdateAuthority: 0,           // CreatorUpdateAuthority
    migrationFee: {
        feePercentage: 1,              // 1% migration fee
        creatorFeePercentage: 50,      // 50% to creator
    },
    collectFeeMode: 0,                 // QuoteToken
    migrationOption: 1,                // DammV2
    activationType: 1,                 // Timestamp
    tokenDecimal: 6,
    tokenType: 0,                      // SplToken
    quoteTokenFlag: 0,
    partnerLockedLpPercentage: 30,
    partnerLpPercentage: 20,
    creatorLockedLpPercentage: 30,
    creatorLpPercentage: 20,
    swapBaseAmount: 800_000_000_000,   // 800k tokens
    migrationQuoteThreshold: 85_000_000, // 85 USDC
    migrationBaseThreshold: 200_000_000_000, // 200k tokens
    sqrtStartPrice: "18446744073709551616", // Price = 1
    lockedVesting: {
        amountPerPeriod: 10_000_000_000,
        cliffDurationFromMigrationTime: 86400,  // 1 day
        frequency: 86400,                       // 1 day
        numberOfPeriod: 30,                     // 30 days
        cliffUnlockAmount: 50_000_000_000,
    },
    migrationFeeOption: 6,             // Customizable
    fixedTokenSupplyFlag: 0,           // Dynamic supply
    preMigrationTokenSupply: 0,
    postMigrationTokenSupply: 0,
    curve,
};

// 4. Call instruction
await program.methods
    .createConfig(configParams)
    .accounts({
        config: configPDA,
        quoteMint: USDC_MINT,
        partner: partnerWallet.publicKey,
        feeClaimer: feeClaimerWallet.publicKey,
        leftoverReceiver: leftoverReceiverWallet.publicKey,
        systemProgram: SystemProgram.programId,
    })
    .signers([partnerWallet])
    .rpc();
```

**PDA Derivation**:
```typescript
const [configPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from("config"), configIndexBuffer],
    programId
);
```

---

### 1.2 Initialize Virtual Pool

**Who**: Creator (token launcher)

**Instruction**: `initialize_virtual_pool_with_spl_token` or `initialize_virtual_pool_with_token2022`

**Steps**:

```typescript
// 1. Prepare token metadata
const metadata = {
    name: "My Token",
    symbol: "MTK",
    uri: "https://example.com/metadata.json",
};

// 2. Calculate initial base amount (includes swap buffer)
// This is the amount of tokens to mint and deposit
const initialBaseAmount = calculateInitialSupply(
    configParams.swapBaseAmount,
    configParams.migrationBaseThreshold,
    configParams.lockedVesting
);

// 3. Derive PDAs
const [poolPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from("pool"), baseMint.publicKey.toBuffer(), configPDA.toBuffer()],
    programId
);

const [baseVaultPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from("token_vault"), poolPDA.toBuffer(), baseMint.publicKey.toBuffer()],
    programId
);

const [quoteVaultPDA] = PublicKey.findProgramAddressSync(
    [Buffer.from("token_vault"), poolPDA.toBuffer(), quoteMint.toBuffer()],
    programId
);

// 4. Call instruction
await program.methods
    .initializeVirtualPoolWithSplToken({
        tokenMetadata: metadata,
        initialBaseAmount,
    })
    .accounts({
        pool: poolPDA,
        config: configPDA,
        creator: creatorWallet.publicKey,
        baseMint: baseMint.publicKey,
        baseVault: baseVaultPDA,
        quoteVault: quoteVaultPDA,
        metadataAccount: metadataPDA,
        // ... other Metaplex accounts
        tokenProgram: TOKEN_PROGRAM_ID,
        systemProgram: SystemProgram.programId,
        rent: SYSVAR_RENT_PUBKEY,
    })
    .signers([creatorWallet, baseMint])
    .rpc();
```

**Result**: Pool is live and ready for trading!

---

## 2. Trading Workflow

### 2.1 Buy Tokens (Quote → Base)

**Who**: Any user

**Instruction**: `swap2` with `ExactIn` mode

**Example**: Buy tokens with 10 USDC

```typescript
// 1. Fetch pool state
const pool = await program.account.virtualPool.fetch(poolPDA);
const config = await program.account.poolConfig.fetch(configPDA);

// 2. Calculate expected output (off-chain simulation)
const { outputAmount, priceImpact } = simulateSwap({
    pool,
    config,
    amountIn: 10_000_000,  // 10 USDC
    tradeDirection: "QuoteToBase",
    swapMode: "ExactIn",
});

// 3. Set slippage tolerance
const minOutputAmount = outputAmount * 0.99;  // 1% slippage

// 4. Execute swap
await program.methods
    .swap2({
        amount0: 10_000_000,        // Input: 10 USDC
        amount1: minOutputAmount,   // Min output
        swapMode: 0,                // ExactIn
    })
    .accounts({
        poolAuthority: poolAuthorityPDA,
        config: configPDA,
        pool: poolPDA,
        inputTokenAccount: userQuoteAccount,
        outputTokenAccount: userBaseAccount,
        baseVault: baseVaultPDA,
        quoteVault: quoteVaultPDA,
        baseMint: baseMintPDA,
        quoteMint: quoteMintPDA,
        payer: userWallet.publicKey,
        tokenBaseProgram: TOKEN_PROGRAM_ID,
        tokenQuoteProgram: TOKEN_PROGRAM_ID,
        referralTokenAccount: null,  // Optional
    })
    .signers([userWallet])
    .rpc();
```

---

### 2.2 Sell Tokens (Base → Quote)

**Who**: Any user

**Instruction**: `swap2` with `ExactIn` mode

**Example**: Sell 1000 tokens for USDC

```typescript
await program.methods
    .swap2({
        amount0: 1000_000_000,      // Input: 1000 tokens
        amount1: minOutputUSDC,     // Min USDC output
        swapMode: 0,                // ExactIn
    })
    .accounts({
        poolAuthority: poolAuthorityPDA,
        config: configPDA,
        pool: poolPDA,
        inputTokenAccount: userBaseAccount,   // Note: Base is input
        outputTokenAccount: userQuoteAccount, // Quote is output
        baseVault: baseVaultPDA,
        quoteVault: quoteVaultPDA,
        baseMint: baseMintPDA,
        quoteMint: quoteMintPDA,
        payer: userWallet.publicKey,
        tokenBaseProgram: TOKEN_PROGRAM_ID,
        tokenQuoteProgram: TOKEN_PROGRAM_ID,
    })
    .signers([userWallet])
    .rpc();
```

---

### 2.3 Exact Output Swap

**Who**: Any user

**Instruction**: `swap2` with `ExactOut` mode

**Example**: Get exactly 1000 tokens

```typescript
await program.methods
    .swap2({
        amount0: 1000_000_000,      // Desired output: 1000 tokens
        amount1: maxInputUSDC,      // Max USDC to spend
        swapMode: 2,                // ExactOut
    })
    .accounts({
        // ... same as above
    })
    .signers([userWallet])
    .rpc();
```

---

### 2.4 With Referral

**Who**: Any user with referral

**Steps**:

```typescript
// Create referral token account
const referralTokenAccount = await getAssociatedTokenAddress(
    quoteMintPDA,  // Or baseMintPDA, depending on fee mode
    referralWallet.publicKey
);

await program.methods
    .swap2(params)
    .accounts({
        // ... normal accounts
        referralTokenAccount,  // Add referral account
    })
    .signers([userWallet])
    .rpc();
```

**Result**: Referral receives a share of the protocol fee.

---

## 3. Fee Management Workflow

### 3.1 Claim Partner Trading Fees

**Who**: Partner (fee_claimer from config)

**Instruction**: `claim_trading_fee`

```typescript
await program.methods
    .claimTradingFee(
        new BN(100_000_000),  // Max base tokens
        new BN(100_000_000)   // Max quote tokens
    )
    .accounts({
        config: configPDA,
        pool: poolPDA,
        partner: partnerWallet.publicKey,
        baseVault: baseVaultPDA,
        quoteVault: quoteVaultPDA,
        partnerBaseAccount: partnerBaseTokenAccount,
        partnerQuoteAccount: partnerQuoteTokenAccount,
        poolAuthority: poolAuthorityPDA,
        tokenBaseProgram: TOKEN_PROGRAM_ID,
        tokenQuoteProgram: TOKEN_PROGRAM_ID,
    })
    .signers([partnerWallet])
    .rpc();
```

---

### 3.2 Claim Creator Trading Fees

**Who**: Creator

**Instruction**: `claim_creator_trading_fee`

```typescript
await program.methods
    .claimCreatorTradingFee(
        new BN(100_000_000),  // Max base tokens
        new BN(100_000_000)   // Max quote tokens
    )
    .accounts({
        config: configPDA,
        pool: poolPDA,
        creator: creatorWallet.publicKey,
        baseVault: baseVaultPDA,
        quoteVault: quoteVaultPDA,
        creatorBaseAccount: creatorBaseTokenAccount,
        creatorQuoteAccount: creatorQuoteTokenAccount,
        poolAuthority: poolAuthorityPDA,
        tokenBaseProgram: TOKEN_PROGRAM_ID,
        tokenQuoteProgram: TOKEN_PROGRAM_ID,
    })
    .signers([creatorWallet])
    .rpc();
```

---

### 3.3 Claim Protocol Fees

**Who**: Protocol admin (via claim operator)

**Instruction**: `claim_protocol_fee`

```typescript
// 1. Create claim operator (one-time setup)
await program.methods
    .createClaimFeeOperator()
    .accounts({
        claimFeeOperator: operatorPDA,
        operator: operatorWallet.publicKey,
        admin: adminWallet.publicKey,
        systemProgram: SystemProgram.programId,
    })
    .signers([adminWallet, operatorWallet])
    .rpc();

// 2. Claim fees
await program.methods
    .claimProtocolFee(
        new BN(100_000_000),
        new BN(100_000_000)
    )
    .accounts({
        claimFeeOperator: operatorPDA,
        pool: poolPDA,
        baseVault: baseVaultPDA,
        quoteVault: quoteVaultPDA,
        feeClaimerBaseAccount: feeClaimerBaseAccount,
        feeClaimerQuoteAccount: feeClaimerQuoteAccount,
        poolAuthority: poolAuthorityPDA,
        tokenBaseProgram: TOKEN_PROGRAM_ID,
        tokenQuoteProgram: TOKEN_PROGRAM_ID,
    })
    .signers([operatorWallet])
    .rpc();
```

---

## 4. Migration Workflow

### Overview

Migration happens when the bonding curve completes (reaches `migration_quote_threshold`).

### 4.1 Automatic Curve Completion

**Trigger**: Last swap that reaches threshold

**What happens**:
1. Pool state updated: `migration_progress = PostBondingCurve` (if vesting) or `LockedVesting`
2. `finish_curve_timestamp` recorded
3. `EvtCurveComplete` event emitted

---

### 4.2 Migrate to Meteora DAMM

**Steps**:

#### Step 1: Create Metadata
```typescript
await program.methods
    .migrationMeteoraDammCreateMetadata()
    .accounts({
        // Meteora program accounts
    })
    .rpc();
```

#### Step 2: Create Locker
```typescript
await program.methods
    .createLocker()
    .accounts({
        locker: lockerPDA,
        pool: poolPDA,
        payer: userWallet.publicKey,
        systemProgram: SystemProgram.programId,
    })
    .signers([userWallet])
    .rpc();
```

#### Step 3: Initialize Migration Pool
```typescript
await program.methods
    .migrateMeteoraDamm()
    .accounts({
        config: configPDA,
        pool: poolPDA,
        creator: creatorWallet.publicKey,
        // ... many Meteora accounts
    })
    .remainingAccounts(extraAccounts)
    .rpc();
```

#### Step 4: Lock LP Tokens
```typescript
await program.methods
    .migrateMeteoraDammLockLpToken()
    .accounts({
        pool: poolPDA,
        locker: lockerPDA,
        // ... locker accounts
    })
    .rpc();
```

#### Step 5: Claim LP Tokens (after vesting)
```typescript
// After vesting period expires
await program.methods
    .migrateMeteoraDammClaimLpToken()
    .accounts({
        pool: poolPDA,
        locker: lockerPDA,
        beneficiary: beneficiaryWallet.publicKey,
        // ... locker accounts
    })
    .rpc();
```

---

### 4.3 Withdraw Surplus

**Who**: Partner, Creator, Protocol

**When**: After migration completes

**Surplus Distribution**:
- Total surplus = `quote_reserve - migration_threshold`
- Partner + Creator: 80% of surplus
- Protocol: 20% of surplus

#### Partner Withdraw:
```typescript
await program.methods
    .partnerWithdrawSurplus()
    .accounts({
        config: configPDA,
        pool: poolPDA,
        partner: partnerWallet.publicKey,
        quoteVault: quoteVaultPDA,
        partnerQuoteAccount: partnerQuoteAccount,
        poolAuthority: poolAuthorityPDA,
        tokenQuoteProgram: TOKEN_PROGRAM_ID,
    })
    .signers([partnerWallet])
    .rpc();
```

#### Creator Withdraw:
```typescript
await program.methods
    .creatorWithdrawSurplus()
    .accounts({
        config: configPDA,
        pool: poolPDA,
        creator: creatorWallet.publicKey,
        quoteVault: quoteVaultPDA,
        creatorQuoteAccount: creatorQuoteAccount,
        poolAuthority: poolAuthorityPDA,
        tokenQuoteProgram: TOKEN_PROGRAM_ID,
    })
    .signers([creatorWallet])
    .rpc();
```

---

### 4.4 Withdraw Migration Fees

**Who**: Partner or Creator

**Instruction**: `withdraw_migration_fee`

```typescript
// Partner withdraws
await program.methods
    .withdrawMigrationFee(0b100)  // PARTNER_MASK
    .accounts({
        config: configPDA,
        pool: poolPDA,
        claimer: partnerWallet.publicKey,
        // ... accounts
    })
    .signers([partnerWallet])
    .rpc();

// Creator withdraws
await program.methods
    .withdrawMigrationFee(0b010)  // CREATOR_MASK
    .accounts({
        config: configPDA,
        pool: poolPDA,
        claimer: creatorWallet.publicKey,
        // ... accounts
    })
    .signers([creatorWallet])
    .rpc();
```

---

### 4.5 Withdraw Leftover

**Who**: Anyone (permissionless, sends to `leftover_receiver`)

**When**: After migration

**Purpose**: Return excess base tokens (if fixed supply)

```typescript
await program.methods
    .withdrawLeftover()
    .accounts({
        config: configPDA,
        pool: poolPDA,
        leftoverReceiver: leftoverReceiverPDA,
        baseVault: baseVaultPDA,
        leftoverReceiverAccount: leftoverReceiverTokenAccount,
        poolAuthority: poolAuthorityPDA,
        baseMint: baseMintPDA,
        tokenBaseProgram: TOKEN_PROGRAM_ID,
    })
    .rpc();
```

---

## 5. Administrative Operations

### 5.1 Transfer Pool Creator

**Who**: Current creator

**Instruction**: `transfer_pool_creator`

```typescript
await program.methods
    .transferPoolCreator()
    .accounts({
        pool: poolPDA,
        creator: currentCreatorWallet.publicKey,
        newCreator: newCreatorWallet.publicKey,
    })
    .signers([currentCreatorWallet])
    .rpc();
```

---

### 5.2 Create Pool Metadata

**Who**: Creator or Partner

```typescript
// Creator metadata
await program.methods
    .createVirtualPoolMetadata({
        name: "My Awesome Pool",
        uri: "https://example.com/pool-info.json",
    })
    .accounts({
        virtualPoolMetadata: metadataPDA,
        pool: poolPDA,
        creator: creatorWallet.publicKey,
        systemProgram: SystemProgram.programId,
    })
    .signers([creatorWallet])
    .rpc();

// Partner metadata
await program.methods
    .createPartnerMetadata({
        name: "Platform Name",
        uri: "https://platform.com/info.json",
    })
    .accounts({
        partnerMetadata: partnerMetadataPDA,
        partner: partnerWallet.publicKey,
        systemProgram: SystemProgram.programId,
    })
    .signers([partnerWallet])
    .rpc();
```

---

## Complete Lifecycle Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                     POOL LIFECYCLE                           │
└──────────────────────────────────────────────────────────────┘

1. CREATION PHASE
   ┌─────────────┐
   │ Partner:    │ create_config()
   │ Create      │ ────────► [Config Created]
   │ Config      │
   └─────────────┘
          │
          │
   ┌─────────────┐
   │ Creator:    │ initialize_pool_*()
   │ Initialize  │ ────────► [Pool Live]
   │ Pool        │            activation_point recorded
   └─────────────┘            migration_progress = PreBondingCurve

2. TRADING PHASE
   ┌─────────────┐
   │ Traders:    │ swap/swap2() repeatedly
   │ Buy/Sell    │ ────────► [Reserves Updated]
   │ Tokens      │            [Fees Accumulated]
   └─────────────┘            [Price Changes]
          │
          │ (optional)
          ▼
   ┌─────────────┐
   │ Partner/    │ claim_*_trading_fee()
   │ Creator:    │ ────────► [Fees Claimed]
   │ Claim Fees  │
   └─────────────┘

3. COMPLETION (automatic on last swap)
          │
          │ quote_reserve >= migration_quote_threshold
          ▼
   [Curve Complete Event]
   migration_progress = PostBondingCurve or LockedVesting
   finish_curve_timestamp recorded

4. MIGRATION PHASE
   ┌─────────────┐
   │ Anyone:     │ create_locker()
   │ Create      │ ────────► [Locker Created]
   │ Locker      │
   └─────────────┘
          │
          ▼
   ┌─────────────┐
   │ Anyone:     │ migration_*_create_metadata()
   │ Create      │ ────────► [Migration Metadata]
   │ Metadata    │
   └─────────────┘
          │
          ▼
   ┌─────────────┐
   │ Anyone:     │ migrate_*()
   │ Initialize  │ ────────► [DEX Pool Created]
   │ DEX Pool    │            migration_progress = CreatedPool
   └─────────────┘            is_migrated = 1
          │
          ▼
   ┌─────────────┐
   │ Anyone:     │ migrate_*_lock_lp_token()
   │ Lock LP     │ ────────► [LP Tokens Locked]
   │ Tokens      │
   └─────────────┘

5. POST-MIGRATION
   ┌─────────────┐
   │ Partner/    │ withdraw_surplus()
   │ Creator/    │ ────────► [Surplus Claimed]
   │ Protocol:   │
   └─────────────┘
          │
          ▼
   ┌─────────────┐
   │ Partner/    │ withdraw_migration_fee()
   │ Creator:    │ ────────► [Migration Fees Claimed]
   └─────────────┘
          │
          ▼
   ┌─────────────┐
   │ Anyone:     │ withdraw_leftover()
   │ Withdraw    │ ────────► [Leftover Tokens Returned]
   │ Leftover    │
   └─────────────┘
          │
          ▼
   ┌─────────────┐
   │ After       │ migrate_*_claim_lp_token()
   │ Vesting:    │ ────────► [LP Tokens Unlocked]
   │ Claim LP    │
   └─────────────┘

                    [Pool Fully Retired]
```
