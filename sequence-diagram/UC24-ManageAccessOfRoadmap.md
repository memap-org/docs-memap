# UC24 — Quản lý quyền truy cập Roadmap (Manage Access Of Roadmap)

> **Lưu ý:** Nội dung chi tiết của use case này đã được tài liệu hoá đầy đủ trong file
> [`UC26-ManageAccessOfRoadmap.md`](UC26-ManageAccessOfRoadmap.md).
> File đó chứa toàn bộ sequence diagram bao gồm: thêm người dùng, đổi/xóa quyền,
> import hàng loạt, export danh sách, copy link và lọc danh sách.

```mermaid
sequenceDiagram
    autonumber
    actor Teacher as Teacher (Owner)
    participant FE as memap-frontend<br/>(React)
    participant GW as API Gateway<br/>(:8090)
    participant RS as roadmap-service<br/>(:8083)
    participant PS as profile-service<br/>(:8082)
    participant NS as notification-service<br/>(:8091)
    participant DB as MongoDB / MySQL

    %% ==================== MỞ POPUP ====================
    rect rgb(230, 245, 255)
        Note over Teacher,DB: MAIN FLOW 1–2 — Tải danh sách quyền truy cập
        Teacher->>FE: Nhấn "Manage Access" trong phần Actions

        par Tải dữ liệu song song
            FE->>GW: GET /roadmap/api/roadmap/{id}/permissions?role=STUDENT<br/>Bearer <access_token>
            GW->>RS: Forward request
            RS->>DB: Truy vấn danh sách permission theo role STUDENT
            DB-->>FE: ApiResponse<RolePermissionResponse[]>

        and
            FE->>GW: POST /profile/users/bulk<br/>{ userIds: [...] }
            GW->>PS: Forward request
            PS-->>FE: ApiResponse<UserEmailResponse[]>
        end

        FE->>FE: Merge permission + user info
        FE-->>Teacher: Hiển thị popup "Sharing roadmap"<br/>Tab mặc định: Students<br/>Bảng: User / Email / Permission / Role
    end

    %% ==================== MAIN FLOW THÊM USER ====================
    rect rgb(230, 255, 240)
        Note over Teacher,NS: MAIN FLOW 3–7 — Thêm người dùng và lưu
        Teacher->>FE: Nhập email vào "Search email to add..."
        FE->>GW: GET /profile/users?email={email}&role=STUDENT
        GW->>PS: Forward request
        PS-->>FE: Gợi ý người dùng tương ứng
        FE-->>Teacher: Hiển thị danh sách gợi ý

        Teacher->>FE: Click chọn người dùng
        FE->>GW: POST /roadmap/api/roadmap/{id}/students<br/>{ userId, permission: "VIEW" }
        GW->>RS: Forward request
        RS->>DB: Thêm RoadmapAccessPermission
        RS->>NS: Gửi thông báo cho người được thêm
        DB-->>RS: RolePermissionResponse
        RS-->>FE: ApiResponse<RolePermissionResponse>
        FE-->>Teacher: Người dùng xuất hiện ngay trong bảng danh sách

        Teacher->>FE: Nhấn "Done"
        FE-->>Teacher: Đóng popup, lưu thay đổi
    end

    %% ==================== ALT 3.F ĐỔI/XÓA QUYỀN ====================
    rect rgb(255, 235, 235)
        Note over Teacher,DB: ALTERNATE FLOW 3.F — Đổi quyền hoặc xóa người dùng
        alt Đổi quyền
            Teacher->>FE: Chọn quyền mới trong cột "Permission"
            FE->>GW: PATCH /roadmap/api/roadmap/{id}/permissions/{userId}/role<br/>{ permission: "EDIT" }
            GW->>RS: Forward request
            RS->>DB: UPDATE permission
            RS-->>FE: RolePermissionResponse cập nhật
            FE-->>Teacher: Quyền cập nhật ngay trên bảng
        else Xóa quyền
            Teacher->>FE: Nhấn icon xóa trong cột "Actions"
            FE->>GW: DELETE /roadmap/api/roadmap/{id}/students/{userId}
            GW->>RS: Forward request
            RS->>DB: Xóa RoadmapAccessPermission
            RS-->>FE: OK
            FE-->>Teacher: Người dùng bị xóa khỏi danh sách
        end
    end
```
