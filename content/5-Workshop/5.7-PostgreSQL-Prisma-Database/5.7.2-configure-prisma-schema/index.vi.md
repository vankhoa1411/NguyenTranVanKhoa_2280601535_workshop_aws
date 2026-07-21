---

title : "Cấu hình Prisma Schema"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.7.2. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ cấu hình **Prisma Schema** cho hệ thống **DocuMind AI**.

Prisma ORM giúp backend Node.js/TypeScript làm việc với PostgreSQL dễ dàng hơn. Thay vì viết SQL thủ công cho mọi thao tác, Prisma cho phép định nghĩa database schema bằng file `schema.prisma`, sau đó generate Prisma Client để backend thao tác với database một cách an toàn và có type support.

Trong DocuMind AI, Prisma Schema sẽ định nghĩa các bảng chính như `User`, `Document`, `OCRResult`, `AIAnalysis`, `ChatHistory`, `Notification` và `AuditLog`.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Cài đặt Prisma cho backend.
* Cấu hình datasource PostgreSQL.
* Tạo các model chính cho DocuMind AI.
* Thiết kế quan hệ giữa User, Document, OCRResult và AIAnalysis.
* Chuẩn bị schema để chạy migration ở bước tiếp theo.

---

#### Vai trò của Prisma trong DocuMind AI

Trong hệ thống **DocuMind AI**, Prisma được dùng để:

* Quản lý database schema.
* Tạo migration cho PostgreSQL.
* Truy vấn dữ liệu từ backend.
* Lưu metadata tài liệu upload.
* Lưu kết quả OCR từ Amazon Textract.
* Lưu kết quả AI Analysis từ Gemini/OpenAI.
* Quản lý notification và audit log.
* Giảm lỗi khi thao tác database nhờ TypeScript type support.

Luồng làm việc:

```text
Backend Service
  |
  v
Prisma Client
  |
  v
PostgreSQL Database
```

![prisma-schema](/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.2-configure-prisma-schema/prisma-schema.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình file `schema.prisma` trong VS Code hoặc vẽ sơ đồ quan hệ giữa User, Document, OCRResult, AIAnalysis, Notification và AuditLog.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.2-configure-prisma-schema/prisma-schema.png`

---

#### Bước 1: Cài đặt Prisma

Trong thư mục backend, cài đặt Prisma:

```bash
npm install prisma --save-dev
npm install @prisma/client
```

Khởi tạo Prisma:

```bash
npx prisma init
```

Sau khi chạy lệnh này, dự án sẽ có:

```text
prisma/
└── schema.prisma

.env
```

---

#### Bước 2: Cấu hình datasource PostgreSQL

Mở file:

```text
prisma/schema.prisma
```

Cấu hình datasource:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

Prisma sẽ lấy connection string từ biến môi trường:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

---

#### Bước 3: Thiết kế model User

Model `User` dùng để lưu thông tin tài khoản.

Ví dụ:

```prisma
model User {
  id           String   @id @default(uuid())
  email        String   @unique
  passwordHash String?
  fullName     String?
  avatarUrl    String?
  role         UserRole @default(USER)
  isActive     Boolean  @default(true)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  documents     Document[]
  notifications Notification[]
  auditLogs     AuditLog[]
  chatHistory   ChatHistory[]
}

enum UserRole {
  USER
  ADMIN
}
```

Trong hệ thống, `role` dùng để phân quyền User/Admin.

---

#### Bước 4: Thiết kế model Document

Model `Document` lưu metadata tài liệu đã upload lên S3.

```prisma
model Document {
  id        String         @id @default(uuid())
  userId    String
  fileName  String
  fileType  String
  fileSize  Int
  s3Bucket  String
  s3Key     String
  status    DocumentStatus @default(PENDING)
  createdAt DateTime       @default(now())
  updatedAt DateTime       @updatedAt

  user       User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  ocrResult  OCRResult?
  aiAnalysis AIAnalysis?
  chatHistory ChatHistory[]
}

enum DocumentStatus {
  PENDING
  UPLOADED
  QUEUED
  PROCESSING
  OCR_COMPLETED
  AI_ANALYZING
  COMPLETED
  FAILED
}
```

Các trường quan trọng:

| Trường     | Ý nghĩa                   |
| ---------- | ------------------------- |
| `s3Bucket` | Tên bucket lưu file       |
| `s3Key`    | Đường dẫn object trong S3 |
| `status`   | Trạng thái xử lý tài liệu |

---

#### Bước 5: Thiết kế model OCRResult

Model `OCRResult` lưu kết quả trích xuất văn bản từ Amazon Textract.

```prisma
model OCRResult {
  id         String   @id @default(uuid())
  documentId String   @unique
  provider   String   @default("AWS_TEXTRACT")
  rawText    String
  createdAt  DateTime @default(now())

  document Document @relation(fields: [documentId], references: [id], onDelete: Cascade)
}
```

Mỗi document thường có một kết quả OCR chính.

---

#### Bước 6: Thiết kế model AIAnalysis

Model `AIAnalysis` lưu kết quả phân tích từ Gemini hoặc OpenAI.

```prisma
model AIAnalysis {
  id                   String   @id @default(uuid())
  documentId           String   @unique
  provider             String
  model                String
  documentType         String?
  summary              String?
  entities             Json?
  keywords             Json?
  importantInformation Json?
  confidence           Float?
  createdAt            DateTime @default(now())

  document Document @relation(fields: [documentId], references: [id], onDelete: Cascade)
}
```

Các trường dạng `Json` giúp lưu danh sách entities, keywords hoặc importantInformation linh hoạt hơn.

---

#### Bước 7: Thiết kế model ChatHistory

Model `ChatHistory` dùng cho AI Chat/RAG.

```prisma
model ChatHistory {
  id         String   @id @default(uuid())
  userId     String
  documentId String
  provider   String
  model      String
  question   String
  answer     String
  sources    Json?
  createdAt  DateTime @default(now())

  user     User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  document Document @relation(fields: [documentId], references: [id], onDelete: Cascade)
}
```

---

#### Bước 8: Thiết kế model Notification

Model `Notification` dùng cho thông báo User/Admin.

```prisma
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

Ví dụ notification:

* Document uploaded successfully.
* OCR completed.
* AI analysis completed.
* AI provider failed.
* Admin security alert.

---

#### Bước 9: Thiết kế model AuditLog

Model `AuditLog` dùng để ghi lại hành động quan trọng.

```prisma
model AuditLog {
  id          String   @id @default(uuid())
  userId      String?
  action      String
  target      String?
  description String?
  ipAddress   String?
  userAgent   String?
  metadata    Json?
  createdAt   DateTime @default(now())

  user User? @relation(fields: [userId], references: [id], onDelete: SetNull)
}
```

Audit log nên dùng cho:

* Login/logout.
* Upload tài liệu.
* Xóa tài liệu.
* Thay đổi role.
* Admin action.
* AI provider error.
* Security event.

---

#### Bước 10: Kiểm tra schema

Sau khi viết schema, kiểm tra format:

```bash
npx prisma format
```

Kiểm tra lỗi schema:

```bash
npx prisma validate
```

Nếu không có lỗi, Prisma sẽ báo schema hợp lệ.

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Prisma đã được cài đặt.
* File `schema.prisma` đã cấu hình PostgreSQL.
* Đã định nghĩa model User.
* Đã định nghĩa model Document.
* Đã định nghĩa model OCRResult.
* Đã định nghĩa model AIAnalysis.
* Đã định nghĩa model ChatHistory.
* Đã định nghĩa model Notification.
* Đã định nghĩa model AuditLog.
* `npx prisma validate` chạy thành công.
* Schema sẵn sàng chạy migration ở bước 5.7.3.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có thiết kế database rõ ràng cho toàn bộ pipeline xử lý tài liệu. Prisma Schema giúp backend quản lý dữ liệu người dùng, tài liệu, OCR, AI Analysis, Chat, Notification và Audit Log một cách thống nhất.
