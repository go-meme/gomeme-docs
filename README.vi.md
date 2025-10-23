# Tài liệu Dynamic Bonding Curve

Tài liệu toàn diện cho chương trình Solana Dynamic Bonding Curve.

## Tổng quan chương trình

Dynamic Bonding Curve là một chương trình Solana triển khai automated market maker (AMM) với:
- **Đường cong thanh khoản tập trung từng phần**
- **Hệ thống phí động** với điều chỉnh dựa trên biến động
- **Bảo vệ chống sniper** thông qua rate limiters
- **Tự động di chuyển thanh khoản** đến Meteora DEX
- **Phân phối phí đa bên** (giao thức, đối tác, người tạo, giới thiệu)

**Program ID**: `dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN`

---

## Cấu trúc tài liệu

### Bắt đầu nhanh

**Mới với bonding curves?** Bắt đầu tại đây:
1. [00-overview.md](./00-overview.md) - Giới thiệu tổng quan
2. [05-workflows.md](./05-workflows.md) - Các thao tác phổ biến và ví dụ
3. [01-instructions.md](./01-instructions.md) - Tham chiếu lệnh

### Tài liệu cốt lõi

| Tệp | Mô tả | Dành cho |
|------|-------|----------|
| [**00-overview.md**](./00-overview.md) | Tổng quan, kiến trúc và khái niệm chính | Tất cả mọi người |
| [**01-instructions.md**](./01-instructions.md) | Tham chiếu lệnh đầy đủ (25 lệnh) | Lập trình viên |
| [**02-state-accounts.md**](./02-state-accounts.md) | Cấu trúc tài khoản và PDAs | Lập trình viên |
| [**03-bonding-curve-math.md**](./03-bonding-curve-math.md) | Công thức toán học | Lập trình viên nâng cao |
| [**04-fee-system.md**](./04-fee-system.md) | Cơ chế phí và phân phối | Tích hợp, trader |
| [**05-workflows.md**](./05-workflows.md) | Hướng dẫn từng bước với ví dụ code | Lập trình viên |
| [**06-migration.md**](./06-migration.md) | Chi tiết hệ thống di chuyển | Người dùng nâng cao |

---

## Tính năng chính

### 1. Bonding Curve

- Lên đến **20 điểm giá** cho thiết kế đường cong linh hoạt
- Mô hình **thanh khoản tập trung** (kiểu Uniswap V3)
- Toán học **fixed-point Q64.64** cho độ chính xác
- Hỗ trợ mọi dải giá ($0.000001 đến $1,000,000)

**Tìm hiểu thêm**: [03-bonding-curve-math.md](./03-bonding-curve-math.md)

### 2. Phí động

- **Phí cơ bản**: Giảm theo thời gian (tuyến tính hoặc mũ)
- **Phí động**: Tăng theo biến động
- **Rate limiter**: Bảo vệ chống sniper
- **Tổng phí**: Giới hạn tối đa 99%

**Tìm hiểu thêm**: [04-fee-system.md](./04-fee-system.md)

### 3. Phân phối đa bên

```
Tổng phí (100%)
├─ Giao thức (20%)
│  ├─ Giao thức (80%)
│  └─ Giới thiệu (20%)
└─ Giao dịch (80%)
   ├─ Đối tác (50%)
   └─ Người tạo (50%)
```

**Tìm hiểu thêm**: [04-fee-system.md#4-fee-distribution](./04-fee-system.md#4-fee-distribution)

### 4. Hệ thống di chuyển

- Tự động di chuyển đến **Meteora DAMM** hoặc **DAMM V2**
- **Vesting LP token** với lịch trình tùy chỉnh
- **Phân phối dư thừa** (80% cho đối tác/người tạo, 20% cho giao thức)
- **Phí di chuyển** có thể cấu hình (0.25% đến 6%)

**Tìm hiểu thêm**: [06-migration.md](./06-migration.md)

---

## Danh mục lệnh

### Admin (4 lệnh)
- `create_claim_fee_operator` - Ủy quyền thu phí
- `close_claim_fee_operator` - Thu hồi ủy quyền
- `claim_protocol_fee` - Thu phí giao thức
- `protocol_withdraw_surplus` - Rút dư thừa sau di chuyển

### Đối tác (4 lệnh)
- `create_partner_metadata` - Tạo thông tin đối tác
- `create_config` - Tạo cấu hình pool
- `claim_trading_fee` - Thu phí giao dịch đối tác
- `partner_withdraw_surplus` - Rút dư thừa

### Người tạo (5 lệnh)
- `initialize_virtual_pool_with_spl_token` - Tạo pool (SPL)
- `initialize_virtual_pool_with_token2022` - Tạo pool (Token-2022)
- `create_virtual_pool_metadata` - Thêm metadata pool
- `claim_creator_trading_fee` - Thu phí người tạo
- `creator_withdraw_surplus` - Rút dư thừa
- `transfer_pool_creator` - Chuyển quyền sở hữu

### Giao dịch (2 lệnh)
- `swap` - Swap exact-in cũ
- `swap2` - Swap nâng cao (exact-in, exact-out, partial-fill)

### Di chuyển (8 lệnh)
- `create_locker` - Tạo locker LP token
- `withdraw_leftover` - Rút token cơ sở dư thừa
- `withdraw_migration_fee` - Thu phí di chuyển
- Di chuyển Meteora DAMM (4 lệnh)
- Di chuyển DAMM V2 (2 lệnh)

**Tham chiếu đầy đủ**: [01-instructions.md](./01-instructions.md)

---

## Tài khoản trạng thái

| Tài khoản | Kích thước | Mục đích |
|-----------|------------|----------|
| **PoolConfig** | 1040 bytes | Cấu hình pool bất biến |
| **VirtualPool** | 416 bytes | Trạng thái pool có thể thay đổi |
| **PartnerMetadata** | Biến đổi | Thông tin đối tác |
| **VirtualPoolMetadata** | Biến đổi | Metadata pool |
| **ClaimFeeOperator** | 40 bytes | Ủy quyền thu phí |

**Tham chiếu đầy đủ**: [02-state-accounts.md](./02-state-accounts.md)

---

## Quy trình phổ biến

### Tạo Pool

```typescript
// 1. Đối tác: Tạo cấu hình
await program.methods.createConfig(params)...

// 2. Người tạo: Khởi tạo pool
await program.methods.initializeVirtualPoolWithSplToken(metadata)...
```

**Hướng dẫn**: [05-workflows.md#1-pool-creation-workflow](./05-workflows.md#1-pool-creation-workflow)

### Giao dịch

```typescript
// Mua token bằng USDC
await program.methods.swap2({
    amount0: inputAmount,      // USDC vào
    amount1: minOutputAmount,  // Token tối thiểu ra
    swapMode: 0,               // ExactIn
})...
```

**Hướng dẫn**: [05-workflows.md#2-trading-workflow](./05-workflows.md#2-trading-workflow)

### Di chuyển

```typescript
// 1. Tạo locker
await program.methods.createLocker()...

// 2. Tạo metadata
await program.methods.migrationDammV2CreateMetadata()...

// 3. Thực hiện di chuyển
await program.methods.migrationDammV2()...
```

**Hướng dẫn**: [05-workflows.md#4-migration-workflow](./05-workflows.md#4-migration-workflow)

---

## Chi tiết toán học

### Công thức Constant Product

Cho mỗi đoạn thanh khoản:
```
x * y = L²
```

Trong đó:
- `x` = số lượng token cơ sở
- `y` = số lượng token quote
- `L` = hằng số thanh khoản

### Tính toán giá

Giá được lưu dưới dạng **sqrt(price)** ở định dạng **Q64.64**:
```
sqrt_price = √(price_quote_per_base) * 2^64
```

### Công thức số lượng token

**Token cơ sở**:
```
Δx = L * (1/√P_lower - 1/√P_upper)
```

**Token quote**:
```
Δy = L * (√P_upper - √P_lower)
```

**Chi tiết đầy đủ**: [03-bonding-curve-math.md](./03-bonding-curve-math.md)

---

## Tính toán phí

### Phí cơ bản (Giảm theo thời gian)

**Tuyến tính**:
```
fee(t) = cliff_fee - (periods * reduction_factor)
```

**Mũ**:
```
fee(t) = cliff_fee * (1 - reduction_factor/10000)^periods
```

### Phí động (Biến động)

```
variable_fee = (volatility * bin_step)^2 * control / 10^11
total_fee = base_fee + variable_fee
```

### Rate Limiter (Chống Sniper)

```
penalty = (volume / reference) * increment_bps
effective_fee = base_fee + penalty
```

**Chi tiết đầy đủ**: [04-fee-system.md](./04-fee-system.md)

---

## Kiến trúc chương trình

```
programs/
└── dynamic-bonding-curve/
    ├── src/
    │   ├── lib.rs                    # Điểm vào chương trình (25 lệnh)
    │   ├── instructions/              # Xử lý lệnh
    │   │   ├── admin/                # Thao tác admin
    │   │   ├── partner/              # Thao tác đối tác
    │   │   ├── creator/              # Thao tác người tạo
    │   │   ├── swap/                 # Logic giao dịch
    │   │   ├── initialize_pool/      # Tạo pool
    │   │   └── migration/            # Hệ thống di chuyển
    │   ├── state/                    # Cấu trúc tài khoản
    │   ├── math/                     # Phép toán
    │   ├── curve.rs                  # Logic bonding curve
    │   ├── base_fee/                 # Fee schedulers
    │   └── ...
```

---

## Hằng số

### Giới hạn giá
```rust
MIN_SQRT_PRICE: u128 = 4_295_048_016              // ~$0.000001
MAX_SQRT_PRICE: u128 = 79_226_673_521_066_979_... // ~$1,000,000
```

### Giới hạn phí
```rust
FEE_DENOMINATOR: u64 = 1_000_000_000   // Độ chính xác 9 chữ số
MAX_FEE_BPS: u64 = 9900                // 99%
MIN_FEE_BPS: u64 = 1                   // 0.01%
PROTOCOL_FEE_PERCENT: u8 = 20          // 20%
```

---

## Tài nguyên

### Tài liệu bên ngoài
- [Tài liệu Meteora DAMM](https://docs.meteora.ag/dlmm/overview)
- [Whitepaper Uniswap V3](https://uniswap.org/whitepaper-v3.pdf)
- [Tài liệu Anchor](https://www.anchor-lang.com/)
- [Tài liệu Solana](https://docs.solana.com/)

---

## Hỗ trợ

Đối với câu hỏi và vấn đề:
- GitHub Issues: [Repository issues](https://github.com/go-meme/gomeme-docs/issues)
- Tài liệu: Thư mục này
- Ví dụ code: Thư mục `tests/`

---

## Thẻ tham chiếu nhanh

### PDAs cần thiết
```
Pool:     ["pool", base_mint, config]
Config:   ["config", config_index]
Vault:    ["token_vault", pool, mint]
Authority: ["pool_authority"]
```

### Thao tác phổ biến
```typescript
// Tạo pool
createConfig() → initializePool()

// Giao dịch
swap2({ amount0, amount1, swapMode: 0 })

// Thu phí
claimTradingFee(maxBase, maxQuote)

// Di chuyển
createLocker() → createMetadata() → migrate()
```

### Giới hạn quan trọng
```
Điểm đường cong tối đa: 20
Phí tối đa: 99%
Phí tối thiểu: 0.01%
Dải giá: $0.000001 đến $1,000,000
```

---

**Cập nhật lần cuối**: 2025-10-23
