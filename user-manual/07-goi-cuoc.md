# 7. Gói Cước & Thanh Toán

[← Bài kiểm tra](06-quiz.md) | [Tiếp theo: Quản lý tài khoản →](08-quan-ly-tai-khoan.md)

---

## Các Gói Dịch Vụ

MeMap cung cấp ba gói dịch vụ:

| | **FREE** | **PRO** | **ENTERPRISE** |
|--|---------|---------|----------------|
| **Giá (tháng)** | Miễn phí | $9.99 | $29.99 |
| **Giá (năm)** | — | $99.99 | $299.99 |
| **Số lộ trình tối đa** | 3 | 25 | Không giới hạn |
| **Lưu trữ / lộ trình** | 1 GB | 2 GB | 3 GB |
| **Tạo lộ trình bằng AI** | ✗ | ✓ | ✓ |
| **Hỗ trợ ưu tiên** | ✗ | ✓ | ✓ |

> **Lưu ý:** Tính năng thanh toán và quản lý gói cước chủ yếu áp dụng cho vai trò **Giáo viên (Teacher)**.  
> Người học (Student) sử dụng tài khoản FREE và không cần nâng cấp để học các lộ trình được chia sẻ.

---

## Gói FREE — Tài Khoản Mặc Định

Khi đăng ký, tài khoản của bạn tự động được cấp gói **FREE**:
- Không cần nhập thông tin thẻ
- Không hết hạn
- Có thể tạo tối đa **3 lộ trình**

---

## Nâng Cấp Lên Gói Trả Phí

### Bước 1 — Truy cập trang thanh toán

Vào menu → **Tài khoản** → **Gói cước** hoặc trực tiếp đến trang `/billing`.

### Bước 2 — Chọn gói và chu kỳ thanh toán

- Chọn tab **Hàng tháng** hoặc **Hàng năm** (tiết kiệm hơn ~17%)
- So sánh các tính năng giữa các gói
- Nhấn **Đăng ký** trên gói muốn chọn

### Bước 3 — Thanh toán qua Stripe

Bạn được chuyển đến trang thanh toán bảo mật của **Stripe**:
1. Nhập thông tin thẻ tín dụng/ghi nợ
2. Nhập địa chỉ thanh toán (nếu yêu cầu)
3. Nhấn **Xác nhận đăng ký**

> MeMap **không lưu** thông tin thẻ của bạn — toàn bộ do Stripe xử lý bảo mật.

### Bước 4 — Xác nhận

Sau khi thanh toán thành công:
- Bạn được chuyển về trang **Thanh toán thành công**
- Gói mới có hiệu lực **ngay lập tức**
- Giới hạn lộ trình và lưu trữ được cập nhật

---

## Xem Thông Tin Gói Hiện Tại

Vào **Tài khoản** → **Gói đăng ký** để xem:

| Thông tin | Ý nghĩa |
|-----------|---------|
| **Tên gói** | FREE / PRO / ENTERPRISE |
| **Trạng thái** | Đang hoạt động / Quá hạn / Đã huỷ |
| **Chu kỳ thanh toán** | Hàng tháng / Hàng năm |
| **Ngày gia hạn** | Khi nào sẽ trừ tiền tiếp theo |
| **Số lộ trình tối đa** | Giới hạn theo gói |
| **Lưu trữ mỗi lộ trình** | Giới hạn theo gói |

### Trạng thái gói

| Trạng thái | Ý nghĩa |
|------------|---------|
| **Đang hoạt động** (ACTIVE) | Bình thường |
| **Đang dùng thử** (TRIALING) | Trong thời gian dùng thử miễn phí |
| **Quá hạn** (PAST_DUE) | Thanh toán không thành công |
| **Đã huỷ** (CANCELLED) | Gói đã bị huỷ, sẽ về FREE |

---

## Đổi Gói Dịch Vụ

1. Vào trang **Gói cước** (`/billing`)
2. Nhấn **Nâng cấp** hoặc **Hạ xuống** trên gói muốn chuyển sang
3. Xác nhận thay đổi
4. Thay đổi có hiệu lực theo chính sách của Stripe (thường ngay lập tức hoặc đầu chu kỳ tiếp theo)

---

## Huỷ Gói Dịch Vụ

### Khi nào nên huỷ?

Nếu bạn không muốn gia hạn thêm, huỷ trước ngày gia hạn để không bị trừ tiền.

### Các bước huỷ

1. Vào **Tài khoản** → **Gói đăng ký**
2. Nhấn nút **Huỷ gói đăng ký**
3. Xác nhận trong hộp thoại xác nhận

### Sau khi huỷ

- Gói **vẫn còn hiệu lực** đến hết ngày gia hạn hiện tại
- Sau đó, tài khoản tự động về gói **FREE**
- Nếu bạn có nhiều hơn 3 lộ trình (giới hạn FREE), các lộ trình dư **không bị xoá** — bạn chỉ không thể tạo thêm lộ trình mới

---

## Câu Hỏi Thường Gặp

**Tôi có thể huỷ bất cứ lúc nào không?**
Có. Không có phí huỷ. Gói vẫn hoạt động đến hết chu kỳ đã trả.

**Thanh toán có an toàn không?**
Có. MeMap dùng **Stripe** — một trong những cổng thanh toán bảo mật nhất thế giới. Thông tin thẻ không lưu trên server của MeMap.

**Tôi có được hoàn tiền nếu huỷ sớm không?**
Chính sách hoàn tiền tùy từng trường hợp. Liên hệ quản trị viên để được hỗ trợ.

**Nếu thanh toán thất bại thì sao?**
Stripe sẽ thử lại tự động. Nếu vẫn thất bại, trạng thái gói chuyển sang **PAST_DUE** — bạn cần cập nhật thông tin thẻ để tiếp tục dùng.

---

[← Bài kiểm tra](06-quiz.md) | [Tiếp theo: Quản lý tài khoản →](08-quan-ly-tai-khoan.md)
