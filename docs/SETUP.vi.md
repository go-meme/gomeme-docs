# Hướng dẫn Cài đặt GitHub Pages

Hướng dẫn này sẽ giúp bạn hoàn tất thiết lập để host tài liệu tại https://docs.go.meme

## ✅ Những gì đã Hoàn thành

- ✅ Các tệp tài liệu đã được tạo và commit
- ✅ Cấu hình MkDocs với Material theme
- ✅ GitHub Actions workflow cho tự động triển khai
- ✅ Tệp CNAME cho custom domain
- ✅ Repository đã được push lên GitHub

## 📋 Các Bước Tiếp theo

### 1. Bật GitHub Pages

1. Truy cập repository của bạn: https://github.com/go-meme/gomeme-docs
2. Nhấp **Settings** → **Pages** (trong thanh bên trái)
3. Dưới "Build and deployment":
   - **Source**: Chọn "Deploy from a branch"
   - **Branch**: Chọn `gh-pages` và `/ (root)`
   - Nhấp **Save**

### 2. Cấu hình Custom Domain trong GitHub

1. Vẫn ở **Settings** → **Pages**
2. Dưới "Custom domain":
   - Nhập: `docs.go.meme`
   - Nhấp **Save**
3. Đợi kiểm tra DNS (có thể mất vài phút)
4. Khi đã xác minh, chọn **"Enforce HTTPS"**

### 3. Cấu hình DNS trong Cloudflare

1. Truy cập Cloudflare dashboard
2. Chọn domain `go.meme` của bạn
3. Nhấp **DNS** → **Records**
4. Thêm/Cập nhật các bản ghi sau:

   **Tùy chọn A: Sử dụng CNAME (Khuyến nghị)**
   ```
   Type: CNAME
   Name: docs
   Target: go-meme.github.io
   Proxy status: Proxied (đám mây màu cam)
   TTL: Auto
   ```

   **Tùy chọn B: Sử dụng A Records (Thay thế)**
   ```
   Type: A
   Name: docs
   IPv4 address: 185.199.108.153
   Proxy status: Proxied
   ```

   Thêm các A records bổ sung cho:
   - 185.199.109.153
   - 185.199.110.153
   - 185.199.111.153

5. Lưu bản ghi DNS

### 4. Đợi Triển khai

GitHub Action sẽ tự động:
1. Build trang MkDocs
2. Deploy lên nhánh `gh-pages`
3. Làm cho nó khả dụng tại custom domain của bạn

**Thời gian**:
- GitHub Actions deploy: ~2-5 phút
- DNS propagation: 5-30 phút
- SSL certificate: 10-60 phút

### 5. Xác minh Triển khai

Sau ~30 phút, truy cập:
- https://docs.go.meme (custom domain của bạn)
- https://go-meme.github.io/gomeme-docs (GitHub Pages mặc định)

Cả hai đều nên hiển thị tài liệu của bạn!

## 🔄 Cập nhật Tài liệu

### Phát triển Local

1. Cài đặt dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. Chạy local server:
   ```bash
   mkdocs serve
   ```

   Truy cập: http://127.0.0.1:8000

3. Thay đổi các tệp `.md`

4. Commit và push:
   ```bash
   git add .
   git commit -m "Cập nhật tài liệu"
   git push
   ```

### Tự động Triển khai

Mỗi lần push lên nhánh `main` sẽ tự động:
1. Kích hoạt GitHub Actions
2. Build trang với MkDocs
3. Deploy lên GitHub Pages
4. Cập nhật https://docs.go.meme

## 🐛 Xử lý Sự cố

### GitHub Pages không hiển thị

1. Kiểm tra GitHub Actions: https://github.com/go-meme/gomeme-docs/actions
2. Đảm bảo workflow hoàn thành thành công
3. Kiểm tra nhánh `gh-pages` tồn tại
4. Xác minh cài đặt Pages trỏ đến nhánh `gh-pages`

### Custom domain không hoạt động

1. Kiểm tra bản ghi DNS trong Cloudflare
2. Xác minh tệp CNAME chứa `docs.go.meme`
3. Đợi 30-60 phút cho DNS propagation
4. Kiểm tra cài đặt GitHub Pages hiển thị domain đã được xác minh

### HTTPS không hoạt động

1. Đợi 10-60 phút để cấp certificate
2. Đảm bảo "Enforce HTTPS" được chọn trong cài đặt GitHub Pages
3. Xóa cache trình duyệt

## ✅ Checklist

- [ ] Bật GitHub Pages trong cài đặt repository
- [ ] Cấu hình nhánh `gh-pages` làm nguồn
- [ ] Đặt custom domain thành `docs.go.meme` trong GitHub
- [ ] Thêm bản ghi CNAME trong Cloudflare DNS
- [ ] Đợi triển khai đầu tiên hoàn thành
- [ ] Xác minh trang có thể truy cập tại https://docs.go.meme
- [ ] Bật HTTPS enforcement
- [ ] Kiểm tra phát triển local với `mkdocs serve`

## 🎉 Hoàn thành!

Khi tất cả các bước hoàn tất, tài liệu của bạn sẽ có mặt tại:

**🌐 https://docs.go.meme**

Mỗi lần push lên nhánh `main` sẽ tự động cập nhật trang trong vài phút!
