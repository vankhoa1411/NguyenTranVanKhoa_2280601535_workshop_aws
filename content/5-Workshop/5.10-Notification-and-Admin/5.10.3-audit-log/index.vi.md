---

title : "Audit Log"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.10.3. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng chức năng **Audit Log** cho hệ thống **DocuMind AI**.

Audit Log dùng để ghi lại các hành động quan trọng của người dùng, Admin và hệ thống. Đây là thành phần cần thiết trong các hệ thống xử lý tài liệu vì nó giúp truy vết hoạt động, kiểm tra lỗi, hỗ trợ bảo mật và tạo nền tảng cho compliance.

Trong DocuMind AI, Audit Log sẽ ghi lại các sự kiện như đăng nhập, upload tài liệu, xử lý OCR, AI Analysis, thay đổi quyền, xóa tài liệu, lỗi worker và các hành động Admin.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo model AuditLog trong PostgreSQL.
* Tạo service ghi audit log.
* Ghi log cho các hành động quan trọng.
* Hiển thị audit log trong Admin Panel.
* Lọc audit log theo user, action, thời gian và severity.
* Hỗ trợ kiểm tra sự kiện xử lý tài liệu.
* Chuẩn bị nền tảng cho security monitoring và compliance.

---

#### Vai trò của Audit Log trong DocuMind AI

Audit Log giúp hệ thống trả lời các câu hỏi:

* Ai đã upload tài liệu?
* Tài liệu nào đã được xử lý?
* Khi nào OCR hoàn tất?
* AI provider nào đã được sử dụng?
* Ai đã xem hoặc xóa tài liệu?
* Admin đã thay đổi gì trong hệ thống?
* Lỗi xử lý xảy ra ở bước nào?
* Có hoạt động bất thường nào không?

---

#### Luồng Audit Log

Luồng ghi audit log:

```text id="i6rbpd"
User / Admin / Worker Action
  |
  v
Audit Log Service
  |
  v
PostgreSQL + Prisma
  |
  v
Admin Audit Log Page
```

![audit-log](/images/5-Workshop/5.10-Notification-and-Admin/5.10.3-audit-log/audit-log.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình bảng Audit Log trong Admin Panel hoặc Prisma Studio hiển thị các sự kiện như login, upload document, OCR completed và AI analysis completed.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.10-Notification-and-Admin/5.10.3-audit-log/audit-log.png`

---

#### AuditLog Model

Ví dụ Prisma model:

```prisma id="e345un"
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

Ý nghĩa:

| Trường        | Ý nghĩa                   |
| ------------- | ------------------------- |
| `userId`      | Người thực hiện hành động |
| `action`      | Loại hành động            |
| `target`      | Đối tượng bị tác động     |
| `description` | Mô tả sự kiện             |
| `ipAddress`   | IP của request            |
| `userAgent`   | Trình duyệt hoặc client   |
| `metadata`    | Dữ liệu phụ               |
| `severity`    | Mức độ sự kiện            |
| `createdAt`   | Thời gian ghi log         |

---

#### Các action nên ghi log

Các action đề xuất:

```text id="8p00i2"
USER_LOGIN
USER_LOGOUT
USER_REGISTER
DOCUMENT_UPLOADED
DOCUMENT_VIEWED
DOCUMENT_DELETED
DOCUMENT_PROCESSING_STARTED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
DOCUMENT_PROCESSING_FAILED
AI_PROVIDER_FALLBACK_USED
ADMIN_VIEWED_USER
ADMIN_CHANGED_ROLE
ADMIN_BROADCAST_NOTIFICATION
SECURITY_FORBIDDEN_ACCESS
```

---

#### Audit Log Service

Tạo file:

```text id="6pdf87"
src/services/audit-log.service.ts
```

Ví dụ:

```ts id="ys4z6q"
import { prisma } from "../prisma/client";

export async function createAuditLog(params: {
  userId?: string;
  action: string;
  target?: string;
  description?: string;
  ipAddress?: string;
  userAgent?: string;
  metadata?: any;
  severity?: string;
}) {
  return prisma.auditLog.create({
    data: {
      userId: params.userId,
      action: params.action,
      target: params.target,
      description: params.description,
      ipAddress: params.ipAddress,
      userAgent: params.userAgent,
      metadata: params.metadata || {},
      severity: params.severity || "info"
    }
  });
}
```

---

#### Ghi log khi user đăng nhập

Trong login controller:

```ts id="nl9rej"
await createAuditLog({
  userId: user.id,
  action: "USER_LOGIN",
  target: "Auth",
  description: "User logged in successfully.",
  ipAddress: req.ip,
  userAgent: req.headers["user-agent"],
  severity: "info"
});
```

---

#### Ghi log khi upload tài liệu

Sau khi upload tài liệu thành công:

```ts id="n5d4u4"
await createAuditLog({
  userId,
  action: "DOCUMENT_UPLOADED",
  target: "Document",
  description: "User uploaded a document.",
  metadata: {
    documentId: document.id,
    fileName: document.fileName,
    s3Key: document.s3Key
  },
  severity: "info"
});
```

---

#### Ghi log khi OCR hoàn tất

Trong worker:

```ts id="gzenak"
await createAuditLog({
  userId,
  action: "OCR_COMPLETED",
  target: "Document",
  description: "Amazon Textract OCR completed.",
  metadata: {
    documentId,
    provider: "AWS_TEXTRACT"
  },
  severity: "info"
});
```

---

#### Ghi log khi AI Analysis hoàn tất

```ts id="txlxxn"
await createAuditLog({
  userId,
  action: "AI_ANALYSIS_COMPLETED",
  target: "Document",
  description: "AI analysis completed successfully.",
  metadata: {
    documentId,
    provider: aiResult.provider,
    model: aiResult.model
  },
  severity: "info"
});
```

---

#### Ghi log khi truy cập bị từ chối

Nếu user cố truy cập document không thuộc quyền:

```ts id="4xwh2z"
await createAuditLog({
  userId: user.id,
  action: "SECURITY_FORBIDDEN_ACCESS",
  target: "Document",
  description: "User attempted to access a document without permission.",
  metadata: {
    documentId
  },
  ipAddress: req.ip,
  userAgent: req.headers["user-agent"],
  severity: "warning"
});
```

---

#### API Audit Log cho Admin

Các API đề xuất:

| API                                | Mục đích                 |
| ---------------------------------- | ------------------------ |
| `GET /api/admin/audit-logs`        | Lấy danh sách audit logs |
| `GET /api/admin/audit-logs/:id`    | Xem chi tiết audit log   |
| `GET /api/admin/audit-logs/export` | Export audit logs        |
| `DELETE /api/admin/audit-logs/:id` | Xóa log nếu cần          |

Tất cả API cần middleware:

```text id="zfwzqp"
requireAuth
requireAdmin
```

---

#### Query filter Audit Log

Admin nên có thể lọc theo:

```text id="o8xrt1"
userId
action
severity
dateFrom
dateTo
target
```

Ví dụ request:

```text id="j1uz3z"
GET /api/admin/audit-logs?action=DOCUMENT_UPLOADED&severity=info
```

---

#### Response Audit Log

Ví dụ:

```json id="z1ahmd"
{
  "success": true,
  "data": [
    {
      "id": "audit_001",
      "userId": "user_001",
      "action": "DOCUMENT_UPLOADED",
      "target": "Document",
      "description": "User uploaded a document.",
      "metadata": {
        "documentId": "doc_001",
        "fileName": "invoice-demo.pdf"
      },
      "severity": "info",
      "createdAt": "2026-06-10T10:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

---

#### Giao diện Audit Log trong Admin Panel

Admin Panel nên hiển thị audit log dạng bảng:

| Time       | User                                      | Action            | Target   | Severity | Detail |
| ---------- | ----------------------------------------- | ----------------- | -------- | -------- | ------ |
| 2026-06-10 | [demo@docmind.ai](mailto:demo@docmind.ai) | DOCUMENT_UPLOADED | Document | info     | View   |

Các chức năng nên có:

* Search.
* Filter theo action.
* Filter theo severity.
* Filter theo thời gian.
* Xem metadata chi tiết.
* Export JSON/CSV nếu cần.

---

#### Quy tắc bảo mật Audit Log

Khi ghi audit log:

* Không log password.
* Không log JWT token.
* Không log API key Gemini/OpenAI.
* Không log AWS credentials.
* Không log toàn bộ nội dung tài liệu nếu nhạy cảm.
* Chỉ log metadata cần thiết.
* Chỉ Admin mới được xem audit log.

---

#### Test Audit Log

Các bước test:

1. Đăng nhập user.
2. Kiểm tra log `USER_LOGIN`.
3. Upload tài liệu.
4. Kiểm tra log `DOCUMENT_UPLOADED`.
5. Chạy worker.
6. Kiểm tra log `OCR_COMPLETED`.
7. Kiểm tra log `AI_ANALYSIS_COMPLETED`.
8. Thử truy cập document không thuộc quyền.
9. Kiểm tra log `SECURITY_FORBIDDEN_ACCESS`.
10. Đăng nhập Admin và mở Audit Log Page.

---

#### Các lỗi thường gặp

| Lỗi                      | Nguyên nhân                          | Cách xử lý                         |
| ------------------------ | ------------------------------------ | ---------------------------------- |
| Không có audit log       | Chưa gọi audit service               | Kiểm tra controller/worker         |
| Log thiếu userId         | Middleware chưa gắn user vào request | Kiểm tra auth middleware           |
| Admin không xem được log | Thiếu API hoặc sai role              | Kiểm tra `requireAdmin`            |
| Metadata quá lớn         | Log quá nhiều dữ liệu                | Chỉ lưu metadata cần thiết         |
| Lộ thông tin nhạy cảm    | Log password/token/API key           | Mask hoặc không log field nhạy cảm |
| Query log chậm           | Bảng log lớn                         | Thêm pagination/filter/index       |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* AuditLog model đã có trong Prisma.
* Service ghi audit log hoạt động.
* Login/logout được ghi log.
* Upload document được ghi log.
* OCR và AI Analysis được ghi log.
* Lỗi xử lý được ghi log.
* Truy cập trái quyền được ghi log.
* Admin Panel hiển thị audit log.
* Audit log có filter và pagination.
* Không ghi dữ liệu nhạy cảm vào audit log.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có hệ thống Audit Log giúp truy vết hoạt động người dùng, Admin và worker. Đây là nền tảng quan trọng để tăng tính minh bạch, hỗ trợ debug, bảo mật và compliance cho hệ thống xử lý tài liệu.
