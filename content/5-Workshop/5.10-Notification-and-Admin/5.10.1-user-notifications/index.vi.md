---

title : "User Notifications"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.10.1. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng chức năng **User Notifications** cho hệ thống **DocuMind AI**.

User Notifications giúp người dùng nhận thông báo về các sự kiện liên quan đến tài liệu của họ, ví dụ như upload thành công, tài liệu đang xử lý, OCR hoàn tất, AI Analysis hoàn tất hoặc quá trình xử lý thất bại.

Chức năng này giúp người dùng không cần liên tục kiểm tra thủ công. Khi hệ thống xử lý xong tài liệu, frontend có thể hiển thị notification để người dùng biết và mở kết quả phân tích ngay trên Dashboard.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được notification cho người dùng.
* Lưu notification vào PostgreSQL thông qua Prisma.
* Hiển thị danh sách notification trên frontend.
* Đánh dấu notification đã đọc.
* Hiển thị notification theo trạng thái tài liệu.
* Tích hợp notification với upload, OCR và AI Analysis.
* Chuẩn bị nền tảng cho real-time notification nếu cần.

---

#### Vai trò của User Notifications trong DocuMind AI

Trong hệ thống **DocuMind AI**, User Notifications được dùng để thông báo:

* Tài liệu upload thành công.
* Tài liệu đã được đưa vào hàng đợi xử lý.
* OCR bằng Amazon Textract đã hoàn tất.
* AI Analysis bằng Gemini/OpenAI đã hoàn tất.
* Tài liệu xử lý thất bại.
* Người dùng có thể xem kết quả trên Dashboard.
* Có lỗi cần người dùng upload lại tài liệu.

Ví dụ notification:

```text id="c9fc9m"
Document uploaded successfully.
OCR processing completed.
AI analysis completed.
Document processing failed.
```

---

#### Luồng User Notifications

Luồng xử lý notification:

```text id="7o115l"
Backend API / Worker
  |
  v
Create Notification
  |
  v
PostgreSQL + Prisma
  |
  v
Frontend Notification Center
  |
  v
User reads notification
```

---

#### Notification Model

Trong Prisma, notification có thể được lưu bằng model `Notification`.

Ví dụ schema:

```prisma id="e6ep4x"
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

Trong đó:

| Trường       | Ý nghĩa                               |
| ------------ | ------------------------------------- |
| `userId`     | Người nhận notification               |
| `roleTarget` | Role nhận notification, ví dụ `ADMIN` |
| `type`       | Loại notification                     |
| `severity`   | Mức độ: info, success, warning, error |
| `title`      | Tiêu đề notification                  |
| `message`    | Nội dung notification                 |
| `metadata`   | Dữ liệu phụ như documentId            |
| `isRead`     | Đã đọc hay chưa                       |
| `readAt`     | Thời gian đọc                         |

---

#### Các loại notification cho User

Các loại notification đề xuất:

```text id="d6mwt1"
DOCUMENT_UPLOADED
DOCUMENT_QUEUED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
```

Ví dụ dữ liệu:

```json id="5ur88b"
{
  "userId": "user_001",
  "type": "AI_ANALYSIS_COMPLETED",
  "severity": "success",
  "title": "AI analysis completed",
  "message": "Your document invoice-demo.pdf has been analyzed successfully.",
  "metadata": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf"
  },
  "isRead": false
}
```

---

#### Notification Service

Tạo file:

```text id="60p0jg"
src/services/notification.service.ts
```

Ví dụ service tạo notification:

```ts id="kpj8k1"
import { prisma } from "../prisma/client";

export async function createUserNotification(params: {
  userId: string;
  type: string;
  severity: string;
  title: string;
  message: string;
  metadata?: any;
}) {
  return prisma.notification.create({
    data: {
      userId: params.userId,
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

#### Tạo notification sau khi upload

Sau khi upload tài liệu thành công, backend có thể tạo notification:

```ts id="t198ll"
await createUserNotification({
  userId,
  type: "DOCUMENT_UPLOADED",
  severity: "success",
  title: "Document uploaded",
  message: `${file.originalname} has been uploaded successfully.`,
  metadata: {
    documentId: document.id,
    fileName: file.originalname
  }
});
```

---

#### Tạo notification sau khi AI Analysis hoàn tất

Trong worker, sau khi cập nhật document status thành `COMPLETED`, tạo notification:

```ts id="v87y7h"
await createUserNotification({
  userId,
  type: "AI_ANALYSIS_COMPLETED",
  severity: "success",
  title: "AI analysis completed",
  message: "Your document has been analyzed successfully.",
  metadata: {
    documentId,
    provider: aiResult.provider,
    model: aiResult.model
  }
});
```

---

#### Tạo notification khi xử lý lỗi

Nếu worker xử lý thất bại:

```ts id="r8mxfw"
await createUserNotification({
  userId,
  type: "DOCUMENT_PROCESSING_FAILED",
  severity: "error",
  title: "Document processing failed",
  message: "Your document could not be processed. Please check the file format or try again.",
  metadata: {
    documentId,
    errorCode: "PROCESSING_FAILED"
  }
});
```

---

#### API Notification đề xuất

Các API cho user:

| API                                   | Mục đích                            |
| ------------------------------------- | ----------------------------------- |
| `GET /api/notifications`              | Lấy danh sách notification của user |
| `GET /api/notifications/unread-count` | Lấy số notification chưa đọc        |
| `PUT /api/notifications/:id/read`     | Đánh dấu một notification đã đọc    |
| `PUT /api/notifications/read-all`     | Đánh dấu tất cả đã đọc              |
| `DELETE /api/notifications/:id`       | Xóa notification                    |

---

#### Response danh sách notification

Ví dụ:

```json id="nsctf0"
{
  "success": true,
  "data": [
    {
      "id": "noti_001",
      "type": "AI_ANALYSIS_COMPLETED",
      "severity": "success",
      "title": "AI analysis completed",
      "message": "Your document invoice-demo.pdf has been analyzed successfully.",
      "isRead": false,
      "metadata": {
        "documentId": "doc_001"
      },
      "createdAt": "2026-06-10T10:00:00.000Z"
    }
  ]
}
```

---

#### Giao diện Notification Center

Frontend nên có:

* Icon chuông notification.
* Badge số lượng unread.
* Dropdown notification list.
* Trạng thái unread/read.
* Button mark as read.
* Button mark all as read.
* Link mở document detail nếu notification có `documentId`.

Ví dụ UI:

```text id="ie2cih"
🔔 3

AI analysis completed
Your document invoice-demo.pdf has been analyzed successfully.
View document
```

---

#### Real-time Notification tùy chọn

Nếu muốn real-time, có thể dùng Socket.IO.

Luồng:

```text id="mjdb1u"
Worker creates notification
  |
  v
Emit socket event to user room
  |
  v
Frontend receives notification
  |
  v
Show toast + update notification badge
```

Room đề xuất:

```text id="1wvf58"
user:{userId}
```

Event đề xuất:

```text id="hfy3v8"
notification:new
```

---

#### Test User Notifications

Các bước test:

1. Đăng nhập bằng tài khoản user.
2. Upload tài liệu.
3. Kiểm tra notification upload thành công.
4. Chạy worker xử lý OCR và AI.
5. Kiểm tra notification AI Analysis hoàn tất.
6. Mở Notification Center.
7. Nhấn notification để mở Document Detail.
8. Mark notification as read.
9. Kiểm tra unread count giảm.

---

#### Các lỗi thường gặp

| Lỗi                                   | Nguyên nhân                          | Cách xử lý                |
| ------------------------------------- | ------------------------------------ | ------------------------- |
| Không có notification                 | Chưa gọi notification service        | Kiểm tra backend/worker   |
| Unread count sai                      | Query chưa filter `isRead=false`     | Kiểm tra API unread count |
| Notification không mở document        | Thiếu `documentId` trong metadata    | Kiểm tra metadata         |
| User thấy notification của người khác | Query thiếu `userId`                 | Kiểm tra auth middleware  |
| Mark read không hoạt động             | Sai notification id hoặc thiếu quyền | Kiểm tra ownership        |
| Toast không hiện real-time            | Socket chưa join room                | Kiểm tra Socket.IO        |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Notification được lưu vào PostgreSQL.
* User nhận notification khi upload tài liệu.
* User nhận notification khi OCR/AI hoàn tất.
* User nhận notification khi xử lý lỗi.
* API lấy notification hoạt động.
* API mark as read hoạt động.
* Frontend hiển thị notification center.
* Unread badge hiển thị đúng.
* User chỉ thấy notification của chính mình.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có hệ thống notification dành cho người dùng. Người dùng có thể biết tài liệu của mình đang được xử lý, đã hoàn tất hoặc gặp lỗi mà không cần kiểm tra thủ công liên tục trên Dashboard.
