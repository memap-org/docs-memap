# 4. Xem Lộ Trình Học

[← Bảng điều khiển](03-bang-dieu-khien.md) | [Tiếp theo: Học tập & Tương tác →](05-hoc-tap.md)

---

## Mở Lộ Trình

Từ Bảng điều khiển, nhấn vào thẻ lộ trình bất kỳ để mở trang xem lộ trình.

---

## Giao Diện Lộ Trình

Lộ trình được hiển thị dưới dạng **sơ đồ luồng tương tác** (flowchart). Bạn có thể:

- **Kéo** để di chuyển xem toàn bộ sơ đồ
- **Cuộn chuột** hoặc dùng nút **+/-** để phóng to / thu nhỏ
- **Nhấn vào nút học** (node) để xem nội dung chi tiết

```
                    ┌─────────────┐
                    │  Chủ đề A   │ ← Nút học (Node)
                    │  [✓ Xong]   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Chủ đề B │ │ Chủ đề C │ │ Chủ đề D │
        │[Đang học]│ │[Chưa học]│ │ [Bỏ qua] │
        └──────────┘ └──────────┘ └──────────┘
```

---

## Trạng Thái Của Nút Học

Mỗi nút học (node) trong lộ trình có một trong bốn trạng thái:

| Trạng thái | Màu sắc | Ý nghĩa |
|------------|---------|---------|
| **Chưa bắt đầu** (PENDING) | Xám | Chưa học đến nút này |
| **Đang học** (IN_PROGRESS) | Xanh dương | Đang trong quá trình học |
| **Hoàn thành** (DONE) | Xanh lá ✓ | Đã học xong |
| **Bỏ qua** (SKIPPED) | Gạch ngang | Chọn bỏ qua nút này |

> Trạng thái của bạn được lưu riêng — người dùng khác không thấy tiến độ của bạn.

---

## Xem Nội Dung Chi Tiết Của Một Nút

Nhấn vào bất kỳ nút học nào để mở **thanh thông tin bên cạnh** (sidebar). Thanh này hiển thị:

### Thông tin chính
- **Tiêu đề** nút học
- **Mô tả** nội dung cần học
- **Tài liệu đính kèm**: link, file PDF, video YouTube

### Cập nhật trạng thái học

Ở cuối sidebar, bạn thấy các nút hành động:

| Nút | Hành động |
|-----|-----------|
| **Đánh dấu Hoàn thành** | Chuyển nút sang trạng thái DONE |
| **Đang học** | Chuyển sang IN_PROGRESS |
| **Bỏ qua** | Chuyển sang SKIPPED |
| **Đặt lại** | Về trạng thái PENDING ban đầu |

### Hỏi đáp AI

Bên trong sidebar có khung **Hỏi & Đáp AI**:
- Nhập câu hỏi về nội dung nút học
- AI phân tích tài liệu của nút và trả lời
- Mỗi câu hỏi tạo ra một cuộc trò chuyện ngắn

> Xem thêm tính năng này tại [Học tập & Tương tác](05-hoc-tap.md).

---

## Thanh Công Cụ Lộ Trình

Phía trên hoặc cạnh sơ đồ có các nút:

| Nút | Chức năng |
|-----|-----------|
| 📋 **Bài kiểm tra** | Xem danh sách quiz của lộ trình này |
| 📝 **Bài tập** | Xem và nộp bài tập |
| 💬 **Bình luận** | Mở khung bình luận của lộ trình |
| 📎 **Tài nguyên** | Xem tất cả tài liệu đính kèm |
| 🎯 **Chế độ tập trung** | Bật Pomodoro timer |
| 📤 **Xuất PDF** | Xuất lộ trình sang file PDF |

---

## Quyền Truy Cập Lộ Trình

Lộ trình có ba mức chia sẻ:

| Mức | Ai có thể xem |
|-----|---------------|
| **Riêng tư** (PRIVATE) | Chỉ chủ sở hữu |
| **Nhóm** (GROUP) | Những người được mời cụ thể |
| **Công khai** (PUBLIC) | Tất cả người dùng có link |

Nếu lộ trình ở chế độ **Nhóm**, bạn phải được chủ sở hữu mời mới xem được.

---

## Thanh Tiến Trình Tổng Quan

Ở đầu trang lộ trình, bạn thấy thanh tiến trình tổng quan hiển thị:
- Số nút đã **Hoàn thành** / Tổng số nút
- Phần trăm hoàn thành

---

[← Bảng điều khiển](03-bang-dieu-khien.md) | [Tiếp theo: Học tập & Tương tác →](05-hoc-tap.md)
