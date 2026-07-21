---

title : "Notification và Admin"
date : 2026-06-10
weight : 10
chapter : false
pre : " <b> 5.10. </b> "
------------------------

#### Xây dựng Notification và Admin

Trong phần này, chúng ta sẽ xây dựng hệ thống **Notification** và **Admin Panel** cho dự án **DocuMind AI**.

Sau khi hệ thống đã có đầy đủ pipeline xử lý tài liệu từ upload, lưu trữ S3, gửi message vào SQS, OCR bằng Amazon Textract và phân tích bằng Gemini/OpenAI, người dùng cần được thông báo khi tài liệu của họ được xử lý thành công hoặc thất bại. Đồng thời, Admin cần có khu vực riêng để theo dõi hoạt động hệ thống, kiểm tra lỗi, xem audit log và quản lý các sự kiện quan trọng.

Phần này giúp DocuMind AI trở nên hoàn chỉnh hơn về mặt vận hành, theo dõi và quản trị hệ thống.

---

#### Vai trò của Notification trong DocuMind AI

Trong hệ thống **DocuMind AI**, Notification được sử dụng để thông báo cho người dùng và Admin về các sự kiện quan trọng.

Đối với người dùng, notification giúp họ biết:

* Tài liệu đã upload thành công.
* Tài liệu đã được đưa vào hàng đợi xử lý.
* OCR bằng Amazon Textract đã hoàn tất.
* AI Analysis bằng Gemini/OpenAI đã hoàn tất.
* Tài liệu xử lý thất bại.
* Người dùng có thể mở Dashboard để xem kết quả.

Đối với Admin, notification giúp theo dõi:

* Tài liệu xử lý lỗi.
* Worker gặp lỗi khi xử lý message.
* AI provider gặp lỗi hoặc fallback.
* Message bị chuyển sang Dead Letter Queue.
* Có hoạt động bất thường hoặc sự kiện bảo mật.
* Các cảnh báo cần kiểm tra trong hệ thống.

---

#### Vai trò của Admin Panel

Admin Panel là khu vực dành cho quản trị viên để theo dõi và vận hành hệ thống.

Admin có thể:

* Xem tổng quan số lượng user.
* Xem tổng số tài liệu đã upload.
* Xem tài liệu đang xử lý, đã hoàn thành hoặc bị lỗi.
* Xem danh sách notification dành cho Admin.
* Xem audit log của hệ thống.
* Kiểm tra lỗi OCR, AI provider hoặc worker.
* Theo dõi các sự kiện bảo mật.
* Quản lý user hoặc role nếu có triển khai.

Admin Panel giúp hệ thống dễ kiểm soát hơn khi có nhiều người dùng và nhiều tài liệu được xử lý bất đồng bộ.

---

#### Kiến trúc Notification và Admin

Luồng tổng quan:

```text id="nleq36"
Backend API / Worker
  |
  |-- Create User Notification
  |-- Create Admin Notification
  |-- Create Audit Log
  |
  v
PostgreSQL + Prisma
  |
  |-----------------------------|
  v                             v
User Notification Center       Admin Panel
                                |
                                v
                         Audit Log / System Alerts
```

![notification-admin-flow](/static/images/5-Workshop/5.10-Notification-and-Admin/notification-admin-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa Backend API và Worker tạo User Notification, Admin Notification và Audit Log, sau đó lưu vào PostgreSQL + Prisma để hiển thị trên User Notification Center và Admin Panel.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.10-Notification-and-Admin/notification-admin-flow.png`

---

#### Workflow Notification

Quy trình notification trong DocuMind AI:

1. Người dùng upload tài liệu.
2. Backend upload file lên Amazon S3.
3. Backend lưu metadata vào PostgreSQL.
4. Backend tạo User Notification: upload thành công.
5. Backend gửi message vào Amazon SQS.
6. Worker nhận message và bắt đầu xử lý.
7. Worker gọi Amazon Textract OCR.
8. Khi OCR hoàn tất, worker lưu OCR Result.
9. Worker gọi Gemini/OpenAI để phân tích tài liệu.
10. Khi AI Analysis hoàn tất, worker lưu AIAnalysis.
11. Worker cập nhật trạng thái tài liệu thành `COMPLETED`.
12. Worker tạo User Notification: phân tích hoàn tất.
13. Nếu có lỗi, worker tạo User Notification và Admin Notification.
14. Admin có thể xem lỗi trong Admin Panel.

---

#### Nội dung chi tiết

Trong phần **5.10 - Notification and Admin**, chúng ta sẽ thực hiện các bước sau:

* [User Notifications](5.10.1-user-notifications/)
* [Admin Notifications](5.10.2-admin-notifications/)
* [Audit Log](5.10.3-audit-log/)
* [Admin Panel](5.10.4-admin-panel/)

---

#### Các bảng dữ liệu liên quan

Phần này sử dụng các bảng chính trong PostgreSQL:

| Bảng           | Mục đích                                             |
| -------------- | ---------------------------------------------------- |
| `Notification` | Lưu thông báo cho User/Admin                         |
| `AuditLog`     | Lưu lịch sử hành động quan trọng                     |
| `User`         | Xác định người nhận notification và role             |
| `Document`     | Liên kết notification/audit với tài liệu             |
| `AIAnalysis`   | Cung cấp thông tin provider/model khi phân tích xong |

---

#### Notification Model

Model notification có thể dùng chung cho cả User và Admin.

Ví dụ:

```prisma id="zx48rq"
model Notification {
  id         String   @id @default(uuid())
  userId     String?
  roleTarget String?
  type       String
  severity   String
  title      String
  message    String
  metadata   Json?
  isRead     Boolean  @default(false)
  createdAt  DateTime @default(now())
  readAt     DateTime?

  user User? @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

Cách dùng:

* Nếu notification gửi cho một user cụ thể, dùng `userId`.
* Nếu notification gửi cho Admin, dùng `roleTarget = "ADMIN"`.
* Nếu notification gửi toàn hệ thống, có thể dùng `roleTarget` hoặc broadcast logic riêng.

---

#### Audit Log Model

Audit Log dùng để ghi lại các hành động quan trọng.

Ví dụ:

```prisma id="x54zh5"
model AuditLog {
  id          String   @id @default(uuid())
  userId      String?
  action      String
  target      String?
  description String?
  ipAddress   String?
  userAgent   String?
  metadata    Json?
  severity    String   @default("info")
  createdAt   DateTime @default(now())

  user User? @relation(fields: [userId], references: [id], onDelete: SetNull)
}
```

Audit Log nên ghi lại:

```text id="aud5ew"
USER_LOGIN
USER_LOGOUT
DOCUMENT_UPLOADED
DOCUMENT_PROCESSING_STARTED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
AI_PROVIDER_FALLBACK_USED
SECURITY_FORBIDDEN_ACCESS
ADMIN_CHANGED_ROLE
```

---

#### Các loại Notification

User notification:

```text id="nbu2cc"
DOCUMENT_UPLOADED
DOCUMENT_QUEUED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
```

Admin notification:

```text id="xjmjdv"
DOCUMENT_PROCESSING_FAILED
TEXTRACT_OCR_FAILED
AI_PROVIDER_FAILED
AI_PROVIDER_FALLBACK
SQS_DLQ_MESSAGE_DETECTED
SECURITY_ALERT
SYSTEM_ERROR
```

---

#### Severity Level

Notification và Audit Log nên có các mức độ:

| Severity   | Ý nghĩa                            |
| ---------- | ---------------------------------- |
| `info`     | Thông tin bình thường              |
| `success`  | Tác vụ hoàn tất                    |
| `warning`  | Có vấn đề cần theo dõi             |
| `error`    | Lỗi cần xử lý                      |
| `critical` | Lỗi nghiêm trọng cần kiểm tra ngay |

Ví dụ:

```text id="dh6qgh"
success: AI analysis completed
warning: AI provider fallback used
error: Document processing failed
critical: Message moved to Dead Letter Queue
```

---

#### API Notification và Admin

Các API dành cho user:

| API                                   | Mục đích                     |
| ------------------------------------- | ---------------------------- |
| `GET /api/notifications`              | Lấy notification của user    |
| `GET /api/notifications/unread-count` | Lấy số notification chưa đọc |
| `PUT /api/notifications/:id/read`     | Đánh dấu đã đọc              |
| `PUT /api/notifications/read-all`     | Đánh dấu tất cả đã đọc       |
| `DELETE /api/notifications/:id`       | Xóa notification             |

Các API dành cho Admin:

| API                                       | Mục đích                     |
| ----------------------------------------- | ---------------------------- |
| `GET /api/admin/overview`                 | Lấy thống kê tổng quan       |
| `GET /api/admin/notifications`            | Lấy admin notifications      |
| `GET /api/admin/audit-logs`               | Lấy audit logs               |
| `GET /api/admin/users`                    | Lấy danh sách user           |
| `GET /api/admin/documents`                | Lấy toàn bộ tài liệu         |
| `GET /api/admin/processing-errors`        | Lấy danh sách lỗi xử lý      |
| `POST /api/admin/notifications/broadcast` | Gửi thông báo đến nhiều user |

Tất cả API admin cần kiểm tra:

```text id="lam3om"
requireAuth
requireAdmin
```

---

#### Giao diện User Notification Center

Frontend nên có:

* Icon chuông notification.
* Badge số lượng unread.
* Dropdown danh sách notification.
* Trạng thái unread/read.
* Button mark as read.
* Button mark all as read.
* Link mở Document Detail nếu notification có `documentId`.

Ví dụ:

```text id="hi3k4n"
AI analysis completed
Your document invoice-demo.pdf has been analyzed successfully.
View document
```

---

#### Giao diện Admin Panel

Admin Panel nên có các khu vực:

```text id="in754a"
Overview
Users
Documents
Processing Errors
Notifications
Audit Logs
Security
Monitoring
```

Overview có thể hiển thị:

```text id="v9ujzz"
Total Users
Total Documents
Processing Documents
Completed Documents
Failed Documents
Unread Admin Alerts
Default AI Provider
Queue Health
```

Admin Notifications hiển thị các cảnh báo theo severity:

```text id="l41vqr"
info
warning
error
critical
```

Audit Logs hiển thị bảng truy vết hoạt động:

```text id="69lckz"
Time | User | Action | Target | Severity | Detail
```

---

#### Real-time Notification tùy chọn

Nếu muốn realtime notification, có thể dùng Socket.IO.

Luồng:

```text id="9bmkid"
Backend/Worker creates notification
  |
  v
Emit socket event
  |
  v
Frontend receives notification
  |
  v
Show toast + update unread badge
```

Room đề xuất:

```text id="in8pvm"
user:{userId}
role:ADMIN
```

Event đề xuất:

```text id="fvc9ox"
notification:new
admin:alert
```

Nếu chưa triển khai realtime, frontend có thể polling notification API mỗi vài giây hoặc refresh khi người dùng mở Notification Center.

---

#### Quy tắc bảo mật

Khi xây dựng Notification và Admin, cần đảm bảo:

* User chỉ xem được notification của chính họ.
* User thường không truy cập được Admin Panel.
* API admin bắt buộc kiểm tra role `ADMIN`.
* Không log password, JWT token, API key hoặc AWS credentials.
* Không đưa Gemini/OpenAI API key ra frontend.
* Audit Log chỉ lưu metadata cần thiết.
* Notification không chứa toàn bộ nội dung tài liệu nhạy cảm.
* Admin action quan trọng cần được ghi Audit Log.

---

#### Logging cần có

Backend và worker nên ghi log:

```text id="u0ms65"
[USER_NOTIFICATION_CREATED]
[ADMIN_NOTIFICATION_CREATED]
[AUDIT_LOG_CREATED]
[ADMIN_PANEL_OVERVIEW_REQUESTED]
[ADMIN_NOTIFICATION_READ]
[SECURITY_FORBIDDEN_ACCESS_LOGGED]
```

Nếu có lỗi:

```text id="7id36w"
[NOTIFICATION_CREATE_FAILED]
[AUDIT_LOG_CREATE_FAILED]
[ADMIN_ACCESS_DENIED]
```

---

#### Test end-to-end

Các bước test:

1. Đăng nhập bằng user thường.
2. Upload tài liệu.
3. Kiểm tra User Notification sau khi upload.
4. Chạy worker xử lý OCR và AI.
5. Kiểm tra notification khi AI Analysis hoàn tất.
6. Tạo một lỗi xử lý tài liệu.
7. Kiểm tra User Notification lỗi.
8. Đăng nhập bằng Admin.
9. Mở Admin Panel.
10. Kiểm tra Admin Notification lỗi.
11. Kiểm tra Audit Log có các sự kiện login, upload, OCR, AI Analysis.
12. Thử user thường truy cập Admin Panel.
13. Kiểm tra bị chặn 403 Forbidden.

---

#### Các lỗi thường gặp

| Lỗi                                   | Nguyên nhân                           | Cách xử lý                       |
| ------------------------------------- | ------------------------------------- | -------------------------------- |
| User không thấy notification          | Chưa gọi notification service         | Kiểm tra backend/worker          |
| User thấy notification của người khác | Query thiếu `userId`                  | Kiểm tra API và auth middleware  |
| Admin không thấy cảnh báo             | Query thiếu `roleTarget=ADMIN`        | Kiểm tra admin notification API  |
| Audit log rỗng                        | Chưa gọi audit service                | Kiểm tra controller/worker       |
| User thường vào được Admin Panel      | Thiếu protected route hoặc middleware | Kiểm tra frontend/backend        |
| Unread count sai                      | Query chưa filter `isRead=false`      | Kiểm tra unread API              |
| Metadata quá lớn                      | Lưu quá nhiều dữ liệu                 | Chỉ lưu metadata cần thiết       |
| Lộ dữ liệu nhạy cảm                   | Log/token/API key bị lưu              | Mask hoặc loại bỏ field nhạy cảm |

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được:

* User nhận notification khi upload tài liệu.
* User nhận notification khi OCR/AI hoàn tất.
* User nhận notification khi tài liệu xử lý thất bại.
* Admin nhận cảnh báo khi hệ thống có lỗi.
* Audit Log ghi lại các hành động quan trọng.
* Admin Panel hiển thị overview, notifications, documents và audit logs.
* User thường không truy cập được Admin Panel.
* Admin có thể kiểm tra lỗi xử lý và sự kiện hệ thống.
* Notification và Audit Log không lưu thông tin nhạy cảm.

---

#### Kết quả kỳ vọng

Sau phần này, DocuMind AI đã có lớp notification và quản trị hệ thống hoàn chỉnh. Người dùng có thể nhận thông báo về tài liệu của mình, còn Admin có thể theo dõi lỗi, audit log và tình trạng vận hành của hệ thống để đảm bảo pipeline xử lý tài liệu hoạt động ổn định.
