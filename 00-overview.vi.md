# Dynamic Bonding Curve - Tổng quan

## Nó là gì?

Dynamic Bonding Curve là một chương trình Solana triển khai automated market maker (AMM) với cơ chế bonding curve. Nó được thiết kế để ra mắt token mới với cơ chế khám phá giá công bằng và tự động di chuyển thanh khoản đến các DEX đã được thiết lập.

## Khái niệm chính

### 1. Bonding Curve
Bonding curve là mối quan hệ toán học giữa giá token và nguồn cung. Khi mua nhiều token hơn, giá tăng theo một đường cong được xác định trước. Chương trình này triển khai bonding curve **thanh khoản tập trung từng phần**, tương tự như Uniswap V3.

### 2. Virtual Pool
Cơ chế giao dịch cốt lõi sử dụng "virtual pools" - các tài khoản on-chain duy trì:
- Dự trữ token cơ sở (token mới được ra mắt)
- Dự trữ token quote (thường là USDC, SOL, hoặc token đã được thiết lập khác)
- Giá hiện tại (được lưu dưới dạng sqrt_price cho độ chính xác toán học)
- Tích lũy và phân phối phí

### 3. Phí động
Chương trình triển khai hệ thống phí tinh vi kết hợp:
- **Phí cơ bản**: Phí dựa trên thời gian có thể giảm theo thời gian (giảm tuyến tính hoặc mũ)
- **Phí động**: Phí dựa trên biến động tăng trong giao dịch hoạt động cao
- **Rate limiter**: Bảo vệ chống sniper làm tăng phí cho giao dịch nhanh

### 4. Hệ thống di chuyển
Khi bonding curve hoàn thành (đạt mục tiêu), thanh khoản tích lũy có thể được di chuyển đến:
- Meteora Dynamic AMM (DAMM V1)
- Meteora Dynamic AMM V2 (DAMM V2)

Việc di chuyển bao gồm cơ chế khóa LP token để đảm bảo thanh khoản dài hạn.

## Kiến trúc chương trình

```
┌─────────────────────────────────────────────────────────────┐
│              Chương trình Dynamic Bonding Curve              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Config     │  │ Virtual Pool │  │   Metadata   │     │
│  │              │  │              │  │              │     │
│  │ • Tham số phí│  │ • Dự trữ     │  │ • Đối tác    │     │
│  │ • Đường cong │  │ • Giá        │  │ • Người tạo  │     │
│  │ • Di chuyển  │  │ • Phí        │  │ • Thông tin  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           Xử lý lệnh (Instructions)                  │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │ • Tạo & Khởi tạo Pool                                │  │
│  │ • Thao tác Swap (Exact In/Out, Partial Fill)        │  │
│  │ • Quản lý & Thu phí                                  │  │
│  │ • Di chuyển & Khóa thanh khoản                       │  │
│  │ • Thao tác Admin                                     │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Logic Toán học & Đường cong              │  │
│  ├──────────────────────────────────────────────────────┤  │
│  │ • Tính toán sqrt price (x64.64 fixed-point)          │  │
│  │ • Phân phối thanh khoản theo điểm giá                │  │
│  │ • Tính toán phí (cơ bản + động + rate limiter)       │  │
│  │ • Bảo vệ slippage                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Tính năng chính

1. **Cơ chế Ra mắt Công bằng**: Bonding curves đảm bảo khám phá giá minh bạch
2. **Bảo vệ Chống Sniper**: Rate limiters ngăn chặn front-running và giao dịch nhanh
3. **Hệ thống Phí Động**: Phí thích ứng với biến động thị trường
4. **Di chuyển Tự động**: Chuyển đổi liền mạch đến các DEX đã được thiết lập
5. **Phân phối Phí Đa bên**: Phân phối công bằng giữa giao thức, người tạo và đối tác
6. **Tính linh hoạt Token**: Hỗ trợ cả SPL Token và Token-2022
7. **Vesting & Khóa**: Cơ chế vesting token và khóa LP token tích hợp

## Các bên liên quan

### 1. Giao thức (Admin)
- Thu phí giao thức
- Có thể rút dư thừa sau khi di chuyển
- Quản lý các claim operators

### 2. Đối tác
- Nền tảng hoặc dịch vụ tích hợp bonding curve
- Nhận phần chia sẻ phí giao dịch và phí di chuyển
- Có thể cấu hình tham số pool
- Nhận LP tokens sau khi di chuyển

### 3. Người tạo
- Người tạo token ra mắt bonding curve
- Nhận phần chia sẻ phí giao dịch
- Có thể thu phí di chuyển
- Nhận LP tokens sau khi di chuyển
- Có thể chuyển quyền sở hữu

### 4. Traders
- Mua và bán token theo bonding curve
- Trả phí giao dịch
- Có thể cung cấp giới thiệu để chia sẻ phí

## Program ID

```
dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN
```

## Bước tiếp theo

Tiếp tục đọc:
- [01-instructions.md](./01-instructions.md) - Tham chiếu lệnh chi tiết
- [02-state-accounts.md](./02-state-accounts.md) - Tài liệu cấu trúc tài khoản
- [03-bonding-curve-math.md](./03-bonding-curve-math.md) - Chi tiết toán học
- [04-fee-system.md](./04-fee-system.md) - Tính toán và phân phối phí
- [05-workflows.md](./05-workflows.md) - Quy trình và thao tác phổ biến
- [06-migration.md](./06-migration.md) - Chi tiết quy trình di chuyển
