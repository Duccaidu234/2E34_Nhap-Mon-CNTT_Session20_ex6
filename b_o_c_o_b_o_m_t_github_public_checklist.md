# QUY TRÌNH BẢO MẬT & CHECKLIST CHO GITHUB PUBLIC REPOSITORY
**Dự án:** Thương mại Điện tử (E-Commerce Platform)  
**Tác giả:** Security & DevOps Team  

---

## 1. Pre-push Checklist & Quy trình Quét API Key

Để đảm bảo không lộ bất kỳ thông tin nhạy cảm nào (VNPay, Momo, Stripe, Database Credentials, AWS Keys) trước khi đẩy mã nguồn lên GitHub Public, toàn bộ các thành viên trong dự án bắt buộc thực hiện các bước sau.

### Bước 1: Cấu hình Git Hook tự động (Pre-push Hook)
Tạo file `.git/hooks/pre-push` hoặc sử dụng **Husky** với nội dung script sau để ngăn chặn việc push mã nguồn chứa secret key:

```bash
#!/bin/sh
# Kiểm tra các từ khóa nhạy cảm trong commit chuẩn bị push
SECRET_REGEX="(api_key|secret|password|passwd|token|aws_access_key|vnp_HashSecret)"

if git diff --cached | grep -iE "$SECRET_REGEX"; then
  echo "⚠️  [BẢO MẬT] Phát hiện thông tin nhạy cảm có khả năng bị hardcode trong code!"
  echo "Vui lòng kiểm tra lại và loại bỏ trước khi push lên public repository."
  exit 1
fi
exit 0
```

* **Cách xác minh (Verification):** Đ thêm một dòng code thử nghiệm như `const secret = "vnp_HashSecret_123456";` và thực hiện `git commit` rồi `git push`. Lệnh push sẽ bị hủy bỏ và terminal hiển thị cảnh báo bảo mật.

### Bước 2: Rà soát lịch sử commit với công cụ quét tự động
Chạy công cụ **TruffleHog** hoặc **GitGuardian** cục bộ để đảm bảo mã nguồn và toàn bộ lịch sử Git không chứa thông tin nhạy cảm:

```bash
# Quét thư mục hiện tại bằng TruffleHog
trufflehog filesystem --no-verification .
```

* **Cách xác minh (Verification):** Terminal trả về kết quả `No leaks found`.

### Bước 3: Kiểm tra tệp `.gitignore`
Xác nhận các file môi trường thực tế không bị theo dõi bởi Git:

```bash
git rm --cached .env .env.local .env.production
git status
```

---

## 2. Cấu trúc tệp `.env.example` Chuẩn Mẫu

Tệp `.env.example` đóng vai trò làm tài liệu hướng dẫn cấu hình cho các môi trường **Development**, **Testing**, và **Production** mà không tiết lộ bất kỳ giá trị thực nào.

```env
# ==========================================
# ỨNG DỤNG THƯƠNG MẠI ĐIỆN TỬ - BIẾN MÔI TRƯỜNG
# ==========================================

# [General Configuration]
NODE_ENV=development # Các giá trị: development | test | production
PORT=3000
APP_URL=http://localhost:3000

# [Database Configuration]
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce_dev_db
DB_USER=postgres
DB_PASSWORD=change_me_to_secure_password

# [Third-Party Payment Gateways]
# VNPay Config (Sandbox / Production)
VNP_TMN_CODE=YOUR_VNPAY_TMN_CODE_HERE
VNP_HASH_SECRET=YOUR_VNPAY_HASH_SECRET_HERE
VNP_URL=https://sandbox.vnpayment.vn/paymentv2/vpcpay.html

# Momo Config
MOMO_PARTNER_CODE=YOUR_MOMO_PARTNER_CODE
MOMO_ACCESS_KEY=YOUR_MOMO_ACCESS_KEY
MOMO_SECRET_KEY=YOUR_MOMO_SECRET_KEY

# Stripe Config
STRIPE_PUBLIC_KEY=pk_test_sample_key
STRIPE_SECRET_KEY=sk_test_sample_key

# [Authentication & JWT Security]
JWT_SECRET=generate_a_very_long_random_secret_string_here
JWT_EXPIRES_IN=7d

# [Cloud Storage & Email Services]
AWS_S3_BUCKET_NAME=your-s3-bucket-name
AWS_ACCESS_KEY_ID=your_aws_access_key
AWS_SECRET_ACCESS_KEY=your_aws_secret_key
AWS_REGION=ap-southeast-1

MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USER=your_mail_username
MAIL_PASS=your_mail_password
```

---

## 3. Báo cáo Đánh giá Rủi ro (Risk Assessment Report) khi thiếu `.gitignore`

### Bối cảnh
Tệp `.gitignore` đóng vai trò là "lá chắn" đầu tiên ngăn chặn Git theo dõi và lưu trữ các tệp chứa thông tin cấu hình nhạy cảm. Việc thiếu tệp này khi public repository sẽ dẫn đến những tổn thất nghiêm trọng.

### Tác động và Rủi ro Cụ thể

| Tiêu chí | Nguy cơ | Tác động kinh doanh & Kỹ thuật |
| :--- | :--- | :--- |
| **Cổng thanh toán** | Rò rỉ `VNP_HASH_SECRET`, `MOMO_SECRET_KEY`, `STRIPE_SECRET_KEY`. | Hacker có thể giả mạo chữ ký số (Checksum) để thanh toán đơn hàng 0 VNĐ, hủy đơn hàng, hoặc rút tiền trực tiếp về tài khoản khác. |
| **Cơ sở dữ liệu** | Lộ `DB_HOST`, `DB_USER`, `DB_PASSWORD`. | Hacker truy cập trực tiếp vào DB, đánh cắp thông tin khách hàng (PII), tống tiền (Ransomware), hoặc xóa dữ liệu sản phẩm. |
| **Hạ tầng Cloud** | Lộ `AWS_ACCESS_KEY_ID` & `AWS_SECRET_ACCESS_KEY`. | Kẻ xấu sử dụng quyền truy cập AWS để đào tiền điện tử (Cryptomining), khiến hóa đơn cloud tăng vọt hàng ngàn USD trong thời gian ngắn. |
| **Xác thực hệ thống** | Lộ `JWT_SECRET`. | Kẻ tấn công có thể tự tạo token xác thực với quyền Admin mà không cần thông qua bước đăng nhập. |

---

## 4. Hướng dẫn Kích hoạt 2FA Bắt buộc cho Repository

Để ngăn ngừa nguy cơ tài khoản cá nhân của lập trình viên bị chiếm đoạt và tải mã độc lên repository, quản trị viên tổ chức cần áp dụng chính sách xác thực 2 lớp (2FA).

1. Truy cập vào **Organization Settings** trên GitHub.
2. Chọn mục **Security** $\rightarrow$ **Authentication security**.
3. Đánh dấu vào mục **Require two-factor authentication for everyone in your organization**.
4. Chọn **Save**. Hệ thống sẽ cấp khoảng thời gian ân hạn (grace period) để các thành viên chưa bật 2FA tiến hành cài đặt (Google Authenticator, Authy hoặc GitHub Mobile) trước khi bị gỡ quyền truy cập.

---

## 5. Tổng kết Checklist trước khi đẩy Code

- [ ] Kiểm tra file `.gitignore` đã liệt kê `.env*`, `node_modules/`, `build/`.
- [ ] Chạy TruffleHog để rà soát toàn bộ commit history.
- [ ] Kiểm tra file `.env.example` đã được cập nhật đầy đủ biến mới.
- [ ] Kích hoạt Git pre-push hook trên máy cá nhân.
- [ ] Xác nhận toàn bộ thành viên dự án đã bật 2FA trên GitHub.