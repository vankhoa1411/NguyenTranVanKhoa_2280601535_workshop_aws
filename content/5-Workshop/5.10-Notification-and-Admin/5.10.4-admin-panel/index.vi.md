---

title : "Admin Panel"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.10.4. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng **Admin Panel** cho hệ thống **DocuMind AI**.

Admin Panel là giao diện dành cho quản trị viên để theo dõi hoạt động hệ thống, quản lý người dùng, kiểm tra tài liệu đã upload, xem lỗi xử lý, theo dõi notification, xem audit log và giám sát các thành phần quan trọng như SQS Worker, AI Provider, CloudWatch Logs hoặc WAF nếu đã tích hợp.

Admin Panel giúp hệ thống dễ vận hành hơn, đặc biệt khi có nhiều người dùng upload tài liệu và nhiều job xử lý chạy bất đồng bộ.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Xây dựng giao diện Admin Panel.
* Chỉ cho phép user có role `ADMIN` truy cập.
* Hiển thị tổng quan hệ thống.
* Hiển thị danh sách users.
* Hiển thị danh sách documents.
* Hiển thị admin notifications.
* Hiển thị audit logs.
* Theo dõi lỗi xử lý tài liệu.
* Chuẩn bị dashboard cho monitoring và security.

---

#### Vai trò của Admin Panel trong DocuMind AI

Admin Panel giúp quản trị viên:

* Xem tổng số user.
* Xem tổng số tài liệu đã upload.
* Xem số tài liệu đang xử lý.
* Xem số tài liệu đã hoàn thành.
* Xem số tài liệu lỗi.
* Theo dõi AI provider đang dùng.
* Kiểm tra lỗi OCR, AI và worker.
* Xem admin notifications.
* Xem audit log.
* Quản lý user role nếu cần.
* Theo dõi cảnh báo bảo mật.

---

#### Kiến trúc Admin Panel

Luồng dữ liệu:

```text id="zlifx7"
Admin User
  |
  v
Admin Panel Frontend
  |
  v
Admin API
  |
  v
PostgreSQL + Prisma
  |
  v
System Events / Notifications / Audit Logs
```

![admin-panel](/static/images/5-Workshop/5.10-Notification-and-Admin/5.10.4-admin-panel/admin-panel.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình Admin Panel hiển thị các card tổng quan như Total Users, Total Documents, Processing, Completed, Failed, Admin Notifications và Audit Logs.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.10-Notification-and-Admin/5.10.4-admin-panel/admin-panel.png`

---

#### Các màn hình trong Admin Panel

Admin Panel có thể gồm:

| Màn hình          | Mục đích                          |
| ----------------- | --------------------------------- |
| Overview          | Tổng quan hệ thống                |
| Users             | Quản lý người dùng                |
| Documents         | Xem toàn bộ tài liệu              |
| Processing Errors | Xem lỗi OCR/AI/Worker             |
| Notifications     | Xem admin notifications           |
| Audit Logs        | Xem lịch sử hoạt động             |
| Security          | Xem cảnh báo bảo mật              |
| Monitoring        | Xem trạng thái log/service nếu có |

---

#### Admin Overview Cards

Trang overview nên có các card:

```text id="ps8xdg"
Total Users
Total Documents
Processing Documents
Completed Documents
Failed Documents
Unread Admin Alerts
AI Provider Status
Queue Health
```

Ví dụ response:

```json id="im6ne8"
{
  "success": true,
  "data": {
    "totalUsers": 25,
    "totalDocuments": 120,
    "processingDocuments": 5,
    "completedDocuments": 100,
    "failedDocuments": 3,
    "unreadAdminAlerts": 4,
    "defaultAIProvider": "openai"
  }
}
```

---

#### API Admin đề xuất

Các API chính:

| API                                | Mục đích                |
| ---------------------------------- | ----------------------- |
| `GET /api/admin/overview`          | Lấy thống kê tổng quan  |
| `GET /api/admin/users`             | Lấy danh sách users     |
| `GET /api/admin/documents`         | Lấy toàn bộ documents   |
| `GET /api/admin/documents/:id`     | Xem chi tiết document   |
| `GET /api/admin/processing-errors` | Xem lỗi xử lý           |
| `GET /api/admin/notifications`     | Xem admin notifications |
| `GET /api/admin/audit-logs`        | Xem audit logs          |
| `PUT /api/admin/users/:id/role`    | Đổi role user nếu cần   |

Tất cả API admin cần middleware:

```text id="o41eo1"
requireAuth
requireAdmin
```

---

#### Middleware kiểm tra Admin

Backend cần kiểm tra role:

```ts id="k6z9yv"
export function requireAdmin(req: any, res: any, next: any) {
  if (!req.user || req.user.role !== "ADMIN") {
    return res.status(403).json({
      success: false,
      message: "Admin permission required."
    });
  }

  next();
}
```

Frontend cũng cần protected route:

```text id="817nzt"
Nếu user.role !== ADMIN:
  redirect về Dashboard hoặc hiển thị 403 Forbidden
```

---

#### Quản lý Users

Trang Users nên hiển thị:

| Cột        | Mô tả             |
| ---------- | ----------------- |
| Name       | Tên user          |
| Email      | Email             |
| Role       | USER/ADMIN        |
| Status     | Active/Inactive   |
| Created At | Ngày tạo          |
| Actions    | View, Change Role |

Nếu cho phép đổi role, cần ghi Audit Log:

```text id="5uy0ni"
ADMIN_CHANGED_ROLE
```

---

#### Quản lý Documents

Trang Documents cho Admin xem:

| Cột         | Mô tả            |
| ----------- | ---------------- |
| File Name   | Tên tài liệu     |
| Owner       | User sở hữu      |
| Status      | Trạng thái xử lý |
| Provider    | OpenAI/Gemini    |
| Uploaded At | Thời gian upload |
| Actions     | View Detail      |

Admin có thể lọc theo:

```text id="fxedvj"
status
user
provider
date
failed only
```

---

#### Processing Errors

Trang Processing Errors nên hiển thị các tài liệu lỗi:

```text id="ngvsxt"
FAILED documents
Textract errors
AI provider errors
SQS/DLQ errors
Permission errors
```

Ví dụ item lỗi:

```json id="qjta42"
{
  "documentId": "doc_001",
  "fileName": "contract-demo.pdf",
  "status": "FAILED",
  "error": "UnsupportedDocumentException",
  "createdAt": "2026-06-10T10:00:00.000Z"
}
```

---

#### Admin Notifications trong Panel

Admin Panel cần hiển thị notification theo severity:

```text id="7lk2h7"
info
warning
error
critical
```

Critical alerts nên nổi bật hơn để Admin xử lý nhanh.

Ví dụ:

```text id="j5riay"
Critical: Message detected in Dead Letter Queue.
Warning: AI provider fallback used.
Error: Document processing failed.
```

---

#### Audit Logs trong Panel

Audit Logs nên có:

* Search.
* Filter theo action.
* Filter theo user.
* Filter theo severity.
* Filter theo date range.
* Pagination.
* View metadata detail.
* Export nếu cần.

Các action quan trọng:

```text id="iy33sp"
USER_LOGIN
DOCUMENT_UPLOADED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
SECURITY_FORBIDDEN_ACCESS
ADMIN_CHANGED_ROLE
```

---

#### Security và Monitoring tùy chọn

Nếu đã tích hợp CloudWatch và WAF, Admin Panel có thể hiển thị thêm:

* CloudWatch log summary.
* WAF allowed/blocked requests.
* Recent security alerts.
* AI provider status.
* SQS queue approximate messages.
* DLQ message count.
* Secrets Manager config status.

Nếu chưa triển khai đầy đủ, có thể hiển thị placeholder:

```text id="xq6f8o"
Monitoring integration will be configured in the next workshop section.
```

---

#### Giao diện đề xuất

Admin Panel nên có layout:

```text id="jv7ccx"
Sidebar
  - Overview
  - Users
  - Documents
  - Notifications
  - Audit Logs
  - Security
  - Monitoring

Main Content
  - Statistic cards
  - Tables
  - Filters
  - Detail drawer/modal
```

Phong cách UI nên đồng bộ với DocuMind AI:

* Glassmorphism card.
* Blue/Sky theme.
* Badge màu theo severity.
* Table rõ ràng.
* Filter dễ dùng.
* Responsive tốt.

---

#### Test Admin Panel

Các bước test:

1. Đăng nhập bằng user thường.
2. Truy cập `/admin`.
3. Kiểm tra bị chặn `403 Forbidden`.
4. Đăng nhập bằng Admin.
5. Truy cập Admin Panel thành công.
6. Kiểm tra Overview Cards.
7. Mở Users page.
8. Mở Documents page.
9. Tạo một document lỗi và kiểm tra Processing Errors.
10. Kiểm tra Admin Notifications.
11. Kiểm tra Audit Logs.
12. Test filter và pagination.

---

#### Các lỗi thường gặp

| Lỗi                              | Nguyên nhân                           | Cách xử lý                      |
| -------------------------------- | ------------------------------------- | ------------------------------- |
| User thường vào được Admin Panel | Thiếu protected route hoặc middleware | Kiểm tra frontend và backend    |
| Admin bị chặn sai                | Role chưa lưu đúng                    | Kiểm tra user.role              |
| Overview count sai               | Query thống kê sai                    | Kiểm tra Prisma aggregate/count |
| Documents không hiển thị owner   | Query thiếu include user              | Kiểm tra Prisma include         |
| Audit log rỗng                   | Chưa gọi audit service                | Kiểm tra controller/worker      |
| Notification không lọc được      | API thiếu filter                      | Kiểm tra query params           |
| UI table chậm                    | Dữ liệu nhiều                         | Thêm pagination                 |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Admin Panel chỉ cho phép role `ADMIN`.
* Overview hiển thị thống kê hệ thống.
* Admin xem được danh sách users.
* Admin xem được toàn bộ documents.
* Admin xem được processing errors.
* Admin xem được admin notifications.
* Admin xem được audit logs.
* Filter và pagination hoạt động.
* Các hành động admin được ghi audit log.
* User thường không truy cập được Admin Panel.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có Admin Panel để quản trị viên theo dõi và vận hành hệ thống. Admin có thể kiểm tra người dùng, tài liệu, lỗi xử lý, notification và audit log, giúp hệ thống dễ kiểm soát hơn khi triển khai thực tế.
