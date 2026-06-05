# 2. Đăng Ký & Đăng Nhập

[← Giới thiệu](01-gioi-thieu.md) | [Tiếp theo: Bảng điều khiển →](03-bang-dieu-khien.md)

---

## Đăng Ký Tài Khoản

MeMap sử dụng hệ thống đăng ký qua **link mời**. Để tạo tài khoản, bạn cần nhận được link mời từ quản trị viên hoặc giáo viên.

### Bước 1 — Mở link mời

Nhấn vào link mời được gửi qua email. Trình duyệt sẽ mở trang đăng ký với địa chỉ dạng:

```
https://<domain>/invite?token=<mã-mời>
```

### Bước 2 — Điền thông tin đăng ký

Điền đầy đủ các trường sau trong form:

| Trường | Mô tả | Ví dụ |
|--------|-------|-------|
| **Họ** | Họ của bạn | Nguyễn |
| **Tên** | Tên của bạn | Văn An |
| **Giới tính** | Nam / Nữ / Khác | Nam |
| **Tên người dùng** | Tên đăng nhập (không dấu, không khoảng trắng) | nguyenvanan |
| **Mật khẩu** | Xem yêu cầu bên dưới | |
| **Xác nhận mật khẩu** | Nhập lại mật khẩu | |

### Yêu Cầu Mật Khẩu

Mật khẩu phải đáp ứng **tất cả** các tiêu chí sau:

- Ít nhất **8 ký tự**
- Có ít nhất **1 chữ hoa** (A–Z)
- Có ít nhất **1 chữ thường** (a–z)
- Có ít nhất **1 chữ số** (0–9)
- Có ít nhất **1 ký tự đặc biệt** (ví dụ: `!@#$%^&*`)

**Ví dụ hợp lệ:** `Hocgioi@2024`

### Bước 3 — Xác nhận đăng ký

Nhấn nút **Đăng ký**. Hệ thống sẽ:
1. Tạo tài khoản trong hệ thống xác thực
2. Tạo hồ sơ cá nhân
3. Cấp gói **FREE** tự động
4. Chuyển bạn đến trang đăng nhập

---

## Đăng Nhập

### Bước 1 — Truy cập trang chủ

Mở trình duyệt và truy cập địa chỉ của nền tảng MeMap. Nhấn nút **Đăng nhập** trên trang chủ.

### Bước 2 — Nhập thông tin đăng nhập

Bạn sẽ được chuyển đến trang đăng nhập bảo mật (Keycloak). Điền:
- **Tên người dùng** hoặc **Email**
- **Mật khẩu**

Nhấn **Đăng nhập**.

### Bước 3 — Vào bảng điều khiển

Sau khi đăng nhập thành công, hệ thống tự động chuyển bạn đến **Bảng điều khiển** (Dashboard).

---

## Đổi Mật Khẩu

Nếu bạn muốn đổi mật khẩu sau khi đăng nhập:

1. Nhấn vào **ảnh đại diện** ở góc trên bên phải
2. Chọn **Hồ sơ & Cài đặt**
3. Tìm mục **Đổi mật khẩu**
4. Điền: Mật khẩu hiện tại → Mật khẩu mới → Xác nhận mật khẩu mới
5. Nhấn **Lưu**

> Xem thêm tại [Quản lý tài khoản](08-quan-ly-tai-khoan.md).

---

## Đăng Xuất

1. Nhấn vào **ảnh đại diện** ở góc trên bên phải
2. Chọn **Đăng xuất**

---

## Các Vấn Đề Thường Gặp

| Vấn đề | Nguyên nhân | Giải pháp |
|--------|-------------|-----------|
| Link mời không hoạt động | Link đã hết hạn hoặc đã dùng | Liên hệ quản trị viên để nhận link mới |
| Mật khẩu không được chấp nhận | Chưa đáp ứng tiêu chí | Kiểm tra lại yêu cầu mật khẩu ở trên |
| Tên người dùng đã tồn tại | Tên đã được đăng ký | Thử tên người dùng khác |
| Không đăng nhập được | Sai tên/mật khẩu | Kiểm tra Caps Lock, thử lại |

---

[← Giới thiệu](01-gioi-thieu.md) | [Tiếp theo: Bảng điều khiển →](03-bang-dieu-khien.md)
