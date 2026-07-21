---

title : "Admin Notifications"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.10.2. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng chức năng **Admin Notifications** cho hệ thống **DocuMind AI**.

Admin Notifications giúp quản trị viên theo dõi các sự kiện quan trọng của hệ thống như lỗi xử lý tài liệu, lỗi OCR, lỗi AI provider, message bị chuyển sang Dead Letter Queue, user upload nhiều tài liệu bất thường hoặc các cảnh báo bảo mật.

Khác với User Notifications chỉ liên quan đến tài liệu của từng người dùng, Admin Notifications tập trung vào việc giám sát toàn hệ thống.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo notification dành cho Admin.
* Lưu admin notification vào PostgreSQL.
* Hiển thị notification trong Admin Panel.
* Phân loại notification theo mức độ nghiêm trọng.
* Tạo cảnh báo khi worker xử lý lỗi.
* Tạo cảnh báo khi AI provider lỗi hoặc fallback.
* Tạo cảnh báo khi có hành động bảo mật đáng chú ý.
* Chuẩn bị dữ liệu cho monitoring và audit log.

---

#### Vai trò của Admin Notifications

Admin Notifications được sử dụng để theo dõi:

* Document processing failed.
* Textract OCR failed.
* Gemini/OpenAI provider failed.
* AI provider fallback được kích hoạt.
* SQS message chuyển sang DLQ.
* Upload file sai định dạng nhiều lần.
* User có hoạt động bất thường.
* Lỗi kết nối database.
* Lỗi quyền IAM hoặc AWS service.
* Cảnh báo bảo mật từ hệ thống.

Ví dụ:

```text id="wir8ds"
AI provider fallback used.
Document processing failed.
SQS message moved to DLQ.
Multiple failed uploads detected.
```

---

#### Luồng Admin Notifications

Luồng xử lý:

```text id="5ujurf"
Backend API / Worker / Security Middleware
  |
  v
Create Admin Notification
  |
  v
PostgreSQL + Prisma
  |
  v
Admin Panel
  |
  v
Admin reviews system event
```



#### Notification dành cho Admin

Admin notification có thể dùng cùng model `Notification`, nhưng dùng `roleTarget`.

Ví dụ:

```json id="n1lylo"
{
  "roleTarget": "ADMIN",
  "type": "AI_PROVIDER_FALLBACK",
  "severity": "warning",
  "title": "AI provider fallback used",
  "message": "OpenAI failed. The system used Gemini as fallback provider.",
  "metadata": {
    "primaryProvider": "openai",
    "fallbackProvider": "gemini",
    "documentId": "doc_001"
  },
  "isRead": false
}
```

---

#### Severity Level

Admin Notifications nên có các mức độ:

| Severity   | Ý nghĩa                |
| ---------- | ---------------------- |
| `info`     | Thông tin bình thường  |
| `success`  | Tác vụ hoàn tất        |
| `warning`  | Có vấn đề cần theo dõi |
| `error`    | Lỗi nghiêm trọng       |
| `critical` | Cần kiểm tra ngay      |

Ví dụ:

```text id="evoyf3"
info
warning
error
critical
```

---

#### Admin Notification Service

Tạo hàm:

```ts id="q9l8h0"
import { prisma } from "../prisma/client";

export async function createAdminNotification(params: {
  type: string;
  severity: string;
  title: string;
  message: string;
  metadata?: any;
}) {
  return prisma.notification.create({
    data: {
      roleTarget: "ADMIN",
      type: params.type,
      severity: params.severity,
      title: params.title,
      message: params.message,
      metadata: params.metadata || {},
      isRead: false
    }
  });
}
```

---

#### Tạo cảnh báo khi AI fallback

Trong AI Gateway, nếu provider chính lỗi và dùng provider fallback:

```ts id="o1x1ej"
await createAdminNotification({
  type: "AI_PROVIDER_FALLBACK",
  severity: "warning",
  title: "AI provider fallback used",
  message: `${primaryProvider} failed. ${fallbackProvider} was used as fallback.`,
  metadata: {
    primaryProvider,
    fallbackProvider,
    documentId
  }
});
```

---

#### Tạo cảnh báo khi document processing failed

Trong worker:

```ts id="u3xkjj"
await createAdminNotification({
  type: "DOCUMENT_PROCESSING_FAILED",
  severity: "error",
  title: "Document processing failed",
  message: "A document failed during OCR or AI Analysis processing.",
  metadata: {
    documentId,
    userId,
    error: error.message
  }
});
```

---

#### Tạo cảnh báo khi message vào DLQ

Nếu có cơ chế kiểm tra DLQ:

```ts id="uw55za"
await createAdminNotification({
  type: "SQS_DLQ_MESSAGE_DETECTED",
  severity: "critical",
  title: "Message detected in Dead Letter Queue",
  message: "A document processing message has been moved to DLQ.",
  metadata: {
    queueName: "docmind-document-processing-dlq"
  }
});
```

---

#### API Admin Notifications

Các API đề xuất:

| API                                         | Mục đích                         |
| ------------------------------------------- | -------------------------------- |
| `GET /api/admin/notifications`              | Admin lấy danh sách notification |
| `GET /api/admin/notifications/unread-count` | Lấy số cảnh báo chưa đọc         |
| `PUT /api/admin/notifications/:id/read`     | Đánh dấu đã đọc                  |
| `PUT /api/admin/notifications/read-all`     | Đánh dấu tất cả đã đọc           |
| `POST /api/admin/notifications/broadcast`   | Gửi thông báo đến nhiều user     |
| `DELETE /api/admin/notifications/:id`       | Xóa notification                 |

Tất cả API admin cần middleware:

```text id="fj4xyp"
requireAuth
requireAdmin
```

---

#### Response Admin Notification

Ví dụ:

```json id="urich4"
{
  "success": true,
  "data": [
    {
      "id": "noti_admin_001",
      "type": "DOCUMENT_PROCESSING_FAILED",
      "severity": "error",
      "title": "Document processing failed",
      "message": "A document failed during OCR processing.",
      "metadata": {
        "documentId": "doc_001",
        "error": "InvalidS3ObjectException"
      },
      "isRead": false,
      "createdAt": "2026-06-10T10:00:00.000Z"
    }
  ]
}
```

---

#### Giao diện Admin Notifications

Admin Panel nên hiển thị:

* Danh sách notification.
* Bộ lọc theo severity.
* Bộ lọc theo type.
* Bộ lọc unread/read.
* Thời gian tạo.
* Chi tiết metadata.
* Button mark as read.
* Button mở document liên quan.
* Badge cảnh báo critical.

Ví dụ filter:

```text id="e0ed30"
All
Unread
Warning
Error
Critical
```

---

#### Broadcast Notification

Admin có thể gửi thông báo đến user.

Ví dụ request:

```json id="5n20dp"
{
  "target": "ALL_USERS",
  "severity": "info",
  "title": "System maintenance",
  "message": "DocuMind AI will be under maintenance tonight."
}
```

Response:

```json id="z2vf2o"
{
  "success": true,
  "message": "Broadcast notification sent successfully."
}
```

---

#### Test Admin Notifications

Các bước test:

1. Đăng nhập bằng tài khoản Admin.
2. Mở Admin Panel.
3. Tạo lỗi xử lý document.
4. Kiểm tra Admin Notification xuất hiện.
5. Test AI provider fallback.
6. Kiểm tra notification warning.
7. Test message lỗi vào DLQ nếu có.
8. Kiểm tra notification critical.
9. Mark as read.
10. Test filter theo severity.

---

#### Các lỗi thường gặp

| Lỗi                                     | Nguyên nhân                    | Cách xử lý                    |
| --------------------------------------- | ------------------------------ | ----------------------------- |
| User thường xem được admin notification | Thiếu `requireAdmin`           | Kiểm tra middleware           |
| Admin không thấy notification           | Query thiếu `roleTarget=ADMIN` | Kiểm tra API                  |
| Severity hiển thị sai                   | Frontend map màu sai           | Kiểm tra UI badge             |
| Metadata không có documentId            | Service chưa truyền metadata   | Kiểm tra nơi tạo notification |
| Broadcast không gửi                     | Logic target user sai          | Kiểm tra query user           |
| Unread count sai                        | Chưa filter `isRead=false`     | Kiểm tra unread API           |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Admin notification được tạo khi có lỗi hệ thống.
* AI fallback tạo notification warning.
* Worker failed tạo notification error.
* DLQ alert tạo notification critical nếu có.
* Admin Panel hiển thị danh sách notification.
* Admin có thể mark as read.
* User thường không xem được admin notification.
* Filter severity hoạt động.
* Broadcast notification hoạt động nếu có triển khai.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có hệ thống notification dành cho Admin. Quản trị viên có thể theo dõi lỗi xử lý tài liệu, lỗi AI provider, cảnh báo queue và các sự kiện quan trọng để vận hành hệ thống ổn định hơn.
