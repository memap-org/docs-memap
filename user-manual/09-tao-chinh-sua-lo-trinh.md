# 9. Tạo & Chỉnh Sửa Lộ Trình (Dành cho Giáo Viên / Chủ Sở Hữu)

[← Quản lý tài khoản](08-quan-ly-tai-khoan.md) | [Mục lục →](README.md)

---

> **Yêu cầu:** Tính năng này chỉ dành cho tài khoản có vai trò **Giáo viên** hoặc người được cấp quyền **EDIT** trên lộ trình.

---

## Tổng Quan Giao Diện Editor

Khi mở lộ trình ở chế độ chỉnh sửa (`/roadmap/edt/:id`), màn hình gồm 3 khu vực chính:

```
┌─────────────────────────────────────────────────────────────┐
│  EditorHeader: [Tên lộ trình] [✏️] [🔊] [📤] [📥] [👁️] [Chia sẻ] [⋯]  │
├─────────────────────────────────────┬───────────────────────┤
│                                     │                       │
│        Canvas (ReactFlow)           │   Right Panel         │
│   - Kéo thả node                    │   (hiện khi chọn      │
│   - Vẽ cạnh kết nối                 │    node / edge)       │
│   - Zoom, pan                       │                       │
│   - Cursors cộng tác                │                       │
│                                     │                       │
│  ┌──────────────────────────────┐   │                       │
│  │ Toolbar: [↩] [↪] | [□] [A] [≡] [◻] │   │                       │
│  └──────────────────────────────┘   │                       │
└─────────────────────────────────────┴───────────────────────┘
```

---

## 1. Tạo Lộ Trình Mới

### Bước 1 — Mở form tạo

Từ **Bảng điều khiển**, nhấn nút **"Tạo mới"** (góc phải phần "Lộ trình của tôi").

### Bước 2 — Điền thông tin

| Trường | Bắt buộc | Ghi chú |
|--------|----------|---------|
| **Tên lộ trình** | Có | Tiêu đề ngắn gọn, rõ ràng |
| **Mô tả** | Có | Phạm vi, đối tượng, mục tiêu lộ trình |
| **Danh mục** | Có | Chọn từ danh sách danh mục sẵn có |

> Form **tự động lưu nháp** vào bộ nhớ trình duyệt — nếu bạn đóng tab, dữ liệu vẫn được giữ khi mở lại.

### Bước 3 — Tạo

Nhấn **"Create Roadmap"**. Hệ thống tạo lộ trình và chuyển thẳng vào **Editor**.

> **Giới hạn gói:** FREE = 3 lộ trình, PRO = 25, ENTERPRISE = không giới hạn. Khi đạt giới hạn, hệ thống hiện gợi ý nâng cấp gói.

---

## 2. Tạo Lộ Trình Bằng AI

Nút ✨ **AI Generate** (trên Dashboard hoặc trong Editor) cho phép AI tự tạo toàn bộ sơ đồ lộ trình.

### Cách dùng

1. Nhấn nút **AI Generate** để mở modal
2. Điền các thông tin:
   - **Topic** (bắt buộc): Chủ đề lộ trình — ví dụ: *"Java Backend Developer from beginner to advanced"*
   - **Description**: Mô tả thêm yêu cầu
   - **Max Nodes** (mặc định 15): Số lượng nút tối đa AI tạo ra
   - **Category**: Danh mục
3. Nhấn **Generate**

### Tiến trình

Hệ thống xử lý bất đồng bộ và hiện thanh tiến trình:

| Trạng thái | Nghĩa |
|------------|-------|
| **PENDING** | Đang chờ trong hàng đợi |
| **PROCESSING** | AI đang tạo lộ trình |
| **COMPLETED** | Hoàn tất — tự chuyển vào Editor |
| **FAILED** | Thất bại — thử lại |

> Sau khi AI tạo xong, bạn có thể chỉnh sửa, thêm, xóa nút bình thường trong Editor.

---

## 3. Giao Diện Editor — Thanh Công Cụ (Toolbar)

Thanh công cụ nằm **ở dưới cùng màn hình canvas**, bao gồm:

### Undo / Redo

| Nút | Phím tắt | Chức năng |
|-----|----------|-----------|
| ↩ Undo | `Ctrl+Z` | Hoàn tác thao tác vừa thực hiện |
| ↪ Redo | `Ctrl+Y` hoặc `Ctrl+Shift+Z` | Làm lại thao tác vừa hoàn tác |

### Các Loại Node

Kéo node từ toolbar thả vào canvas để thêm:

| Biểu tượng | Loại | Nhóm | Mô tả |
|-----------|------|------|-------|
| □ | **Rectangle Topic** | Topic | Node học tập chính — có thể thêm nội dung, tài liệu, AI |
| A | **Label** | Label | Nhãn văn bản đơn giản trên canvas |
| ⊞ | **Labeled Group** | Label | Khung nhóm với tiêu đề — gom nhiều node lại |
| ≡ | **Paragraph** | Paragraph | Khối văn bản tự do |
| ◻ | **Resource** | — | Node liên kết tài nguyên (link, file) |

> Ngoài kéo thả, bạn có thể nhấn vào biểu tượng trong toolbar rồi nhấp vào canvas để thêm node tại vị trí đó.

---

## 4. Làm Việc Trên Canvas

### Di chuyển & Zoom

| Thao tác | Cách thực hiện |
|----------|----------------|
| **Pan (kéo khung nhìn)** | Click giữ vùng trống và kéo |
| **Zoom in / out** | Cuộn chuột hoặc nhấn `+` / `-` |
| **Fit to screen** | Nhấn nút ⊡ trên controls |
| **MiniMap** | Góc dưới phải — nhấp để nhảy đến vùng đó |

### Thêm Node

1. Kéo loại node từ Toolbar thả vào canvas, HOẶC
2. Nhấn đúp vào vùng canvas trống → thêm Rectangle Topic mặc định

### Chọn & Xóa

| Thao tác | Cách thực hiện |
|----------|----------------|
| **Chọn 1 node/edge** | Nhấp vào node hoặc edge |
| **Chọn nhiều** | Kéo vùng chọn trên canvas (click giữ vùng trống rồi kéo) |
| **Xóa đã chọn** | Phím `Delete` hoặc `Backspace` |

> Xóa node sẽ **tự động xóa tất cả các cạnh kết nối** đến node đó.

### Vẽ Kết Nối (Edge)

1. Di chuột đến **viền của node nguồn** — điểm kết nối (handle) xuất hiện
2. Kéo từ điểm đó đến **node đích**
3. Cạnh được tạo với style mặc định (mũi tên đen)

### Đổi Tên Node (Inline)

Nhấp đúp vào node trên canvas để sửa tên trực tiếp.

### Alignment Guides

Khi kéo node gần node khác, **đường dẫn căn chỉnh** tự động hiện để hỗ trợ đặt node thẳng hàng.

---

## 5. Right Panel — Tùy Chỉnh Node

Khi nhấn chọn một node, **Right Panel** mở ra bên phải. Nội dung thay đổi theo loại node.

### 5.1 Topic Node (Rectangle Topic)

#### Quick Actions

| Nút | Phím tắt | Chức năng |
|-----|----------|-----------|
| **Duplicate** | — | Nhân bản node (dịch sang phải 50px, xuống 50px) |
| **Copy** | — | Sao chép node vào clipboard |
| **Delete** | `Delete` | Xóa node và tất cả cạnh liên quan |
| **Lock / Unlock** | — | Khóa node — không di chuyển được khi đang bị khóa |
| **Reset** | — | Đặt lại style về mặc định (nền vàng, viền đen) |

#### Layer & Style

| Nút | Chức năng |
|-----|-----------|
| **Front** | Đưa node lên trên cùng (tăng z-index) |
| **Back** | Đưa node xuống dưới cùng (giảm z-index) |
| **Copy Style** | Sao chép style hiện tại vào clipboard |
| **Paste Style** | Dán style đã sao chép vào node này |

#### Align (Multi-select)

Chọn **ít nhất 2 node** trước khi dùng:

| Nút | Chức năng |
|-----|-----------|
| Align Left | Căn trái theo node ngoài cùng bên trái |
| Align Center | Căn giữa theo trung điểm ngang |
| Align Right | Căn phải theo node ngoài cùng bên phải |
| Align Top | Căn trên theo node cao nhất |
| Align Middle | Căn giữa theo trung điểm dọc |
| Align Bottom | Căn dưới theo node thấp nhất |
| Match Width | Đồng bộ chiều rộng với node đang chọn |
| Match Height | Đồng bộ chiều cao với node đang chọn |

#### Appearance

Tùy chỉnh màu sắc, font chữ, viền, bóng đổ — thay đổi áp dụng ngay lập tức trên canvas.

#### Chỉnh Sửa Nội Dung (Edit Content)

Nhấn nút **"Edit Content"** ở dưới cùng Right Panel để mở **trình soạn thảo nội dung**.

---

### 5.2 Label & Labeled Group

Tương tự Topic Node, gồm:
- Quick Actions (Duplicate, Copy, Delete, Lock, Reset)
- Layer controls (Front, Back, Copy/Paste Style)
- **LabelProperties** — tùy chỉnh màu chữ, cỡ chữ, nền

### 5.3 Paragraph

- Quick Actions, Layer controls
- **ParagraphProperties** — tùy chỉnh màu nền, viền, font

### 5.4 Resource Node

- Quick Actions, Layer controls
- **ResourceProperties** — cấu hình URL, loại tài nguyên (link / video / file)

---

## 6. Trình Soạn Thảo Nội Dung Node (ContentEdit)

Mở bằng cách nhấn **"Edit Content"** trong Right Panel của một **Topic Node**.

Trình soạn thảo dùng **TipTap** với đầy đủ tính năng:

### Định dạng văn bản

| Tính năng | Mô tả |
|-----------|-------|
| **Bold / Italic / Underline / Strikethrough** | Định dạng cơ bản |
| **Tiêu đề H1 – H6** | Phân cấp nội dung |
| **Danh sách** | Bullet list, Ordered list, Task list (checkbox) |
| **Căn chỉnh** | Trái, giữa, phải, đều |
| **Màu chữ / Highlight** | Tô màu nội dung |
| **Subscript / Superscript** | Chỉ số dưới / trên |
| **Font chữ** | Thay đổi kiểu font |

### Nhúng nội dung

| Tính năng | Mô tả |
|-----------|-------|
| **Link** | Chèn liên kết |
| **Hình ảnh** | Upload ảnh có thể kéo thay đổi kích thước |
| **Video (file)** | Nhúng video trực tiếp |
| **YouTube** | Nhúng video YouTube bằng URL |
| **Code block** | Khối code với syntax highlighting (hỗ trợ nhiều ngôn ngữ) |

### Nhập từ file

- Nhấn nút **Import .docx** để chuyển đổi tài liệu Word thành nội dung trong editor

### Hỗ trợ AI

- Nhấn nút ✨ **AI** trong editor để nhờ AI hỗ trợ viết / tóm tắt / bổ sung nội dung cho node đang soạn

---

## 7. Tùy Chỉnh Edge (Cạnh Kết Nối)

Nhấn vào bất kỳ cạnh nào trên canvas để mở **Right Edge Panel**:

- **Connection Info**: Hiển thị node nguồn → node đích
- **EdgeProperties**: Tùy chỉnh màu sắc, độ dày, kiểu mũi tên, style đường
- **Delete Edge**: Nút xóa cạnh (màu đỏ)

---

## 8. Thanh Header Editor

Phía trên màn hình editor có các nút:

| Nút | Chức năng |
|-----|-----------|
| **Tên lộ trình** | Hiển thị tên hiện tại |
| ✏️ **Edit Info** | Chỉnh sửa tên, mô tả, danh mục lộ trình |
| 🔊 **Voice Chat** | Bật micro để nói chuyện với người cùng chỉnh sửa |
| 📤 **Export JSON** | Xuất toàn bộ canvas thành file `.json` |
| 📥 **Import JSON** | Nạp file `.json` để thay thế canvas hiện tại |
| 👁️ **Live View** | Chuyển sang chế độ xem (viewer) của lộ trình |
| **Chia sẻ** | Mở cửa sổ quản lý quyền truy cập |
| ⋯ **More** | Các tùy chọn thêm |

### Chỉnh Sửa Thông Tin Lộ Trình

Nhấn ✏️ → **Edit Info modal** cho phép cập nhật:
- Tên lộ trình
- Mô tả
- Danh mục

> Thay đổi được đồng bộ ngay lập tức qua Yjs — tất cả người cùng mở editor đều thấy tên mới.

### Export / Import JSON

**Export:**
1. Nhấn 📤 **Export JSON**
2. File `.json` tải về máy với tên theo lộ trình
3. File chứa toàn bộ: nodes, edges, thông tin lộ trình

**Import:**
1. Nhấn 📥 **Import JSON**
2. Chọn file `.json` đã export trước đó
3. Nếu canvas đang có dữ liệu → hệ thống hỏi xác nhận trước khi ghi đè
4. Canvas được thay thế hoàn toàn bằng dữ liệu từ file

> Import hữu ích để **sao lưu**, **clone** lộ trình, hoặc **di chuyển** giữa các tài khoản.

---

## 9. Quản Lý Quyền Truy Cập (Chia Sẻ)

Nhấn nút **Chia sẻ** trên header để mở cửa sổ quản lý quyền.

### Hai vai trò

| Tab | Vai trò |
|-----|---------|
| **Students** | Học sinh được thêm vào lộ trình |
| **Teachers** | Giáo viên đồng biên soạn |

### Thêm người dùng thủ công

1. Nhập **email** vào ô tìm kiếm
2. Chọn người dùng từ gợi ý
3. Hộp thoại hiện ra — chọn **mức quyền**:
   - **VIEW** — chỉ xem lộ trình
   - **EDIT** — chỉnh sửa canvas và nội dung
4. Nhấn **Save**

### Import danh sách từ Excel

1. Nhấn nút **Import** (biểu tượng upload)
2. Tải file Excel theo mẫu (có thể tải mẫu từ dialog)
3. Cột bắt buộc: `email`, `permission`, `role`
4. Hệ thống xử lý bất đồng bộ và hiện tiến trình:
   - Số dòng thành công / thất bại
   - Chi tiết lỗi từng dòng (nếu có)

### Export danh sách

- Nhấn **Export** để tải file Excel chứa toàn bộ danh sách quyền hiện tại

### Thay đổi / Xóa quyền

- Nhấp vào badge quyền (VIEW / EDIT) của người dùng → hộp thoại đổi quyền
- Nhấn icon 🗑️ → xóa quyền truy cập

### Sao chép link

- Nhấn **Copy link** ở cuối cửa sổ → link lộ trình được sao chép vào clipboard

---

## 10. Cộng Tác Thời Gian Thực

Nhiều người cùng chỉnh sửa một lộ trình **đồng thời**:

- **Cursor awareness**: Thấy con trỏ và tên của người dùng khác di chuyển trên canvas
- **Đồng bộ tức thì**: Mọi thao tác (thêm node, kéo, xóa, đổi màu) hiện với tất cả người đang mở editor ngay lập tức
- **Conflict-free**: Dùng thuật toán **CRDT (Yjs)** — không xảy ra xung đột khi nhiều người cùng sửa
- **Voice chat**: Bật mic 🔊 để nói chuyện trực tiếp với người cùng chỉnh sửa (WebRTC, không cần phần mềm ngoài)

---

## 11. Phím Tắt Tổng Hợp

| Phím tắt | Chức năng |
|----------|-----------|
| `Ctrl+Z` | Undo |
| `Ctrl+Y` / `Ctrl+Shift+Z` | Redo |
| `Delete` / `Backspace` | Xóa node / edge đang chọn |
| `Ctrl+Click` | Chọn nhiều node |
| Kéo vùng chọn | Multi-select |
| Nhấp đúp vào node | Đổi tên inline |

---

## Các Lỗi Thường Gặp

| Lỗi | Nguyên nhân | Cách xử lý |
|-----|------------|------------|
| Không thể tạo lộ trình | Đạt giới hạn gói | Nâng cấp gói tại [Gói cước](07-goi-cuoc.md) |
| Import JSON không được | File sai định dạng hoặc canvas đang tải | Chờ canvas load xong, dùng file `.json` đã export từ MeMap |
| Không thấy cursor người khác | Kết nối WebSocket bị gián đoạn | Tải lại trang |
| Node bị khóa không kéo được | Trạng thái Lock đang bật | Mở Right Panel → nhấn Unlock |

---

[← Quản lý tài khoản](08-quan-ly-tai-khoan.md) | [Mục lục →](README.md)
