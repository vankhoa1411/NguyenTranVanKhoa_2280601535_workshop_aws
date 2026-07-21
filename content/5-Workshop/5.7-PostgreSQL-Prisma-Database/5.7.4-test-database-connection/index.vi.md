---

title : "Kiểm tra kết nối Database"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.7.4. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ kiểm tra kết nối giữa backend **DocuMind AI**, **Prisma ORM** và **PostgreSQL Database**.

Sau khi đã thiết lập PostgreSQL, cấu hình Prisma Schema và chạy Prisma Migration, backend cần được kiểm tra để đảm bảo có thể đọc và ghi dữ liệu vào database. Đây là bước quan trọng trước khi kết nối database với các chức năng như upload tài liệu, lưu OCR Result, lưu AI Analysis, Notification và Audit Log.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Kiểm tra `DATABASE_URL` có đúng hay không.
* Kiểm tra Prisma Client đã được generate.
* Kiểm tra backend kết nối được PostgreSQL.
* Kiểm tra tạo dữ liệu thử nghiệm.
* Kiểm tra truy vấn dữ liệu bằng Prisma.
* Kiểm tra database sẵn sàng cho các module S3, SQS, Textract và AI Analysis.

---

#### Luồng kiểm tra kết nối

Luồng kiểm tra database trong DocuMind AI:

```text
Backend API
  |
  v
Prisma Client
  |
  v
PostgreSQL Database
  |
  v
Return test result
```

![test-database-connection](/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.4-test-database-connection/test-database-connection.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình terminal hoặc Postman khi test API kết nối database thành công, hoặc chụp Prisma Studio hiển thị dữ liệu test.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.4-test-database-connection/test-database-connection.png`

---

#### Điều kiện trước khi kiểm tra

Trước khi test, hãy đảm bảo:

* PostgreSQL đang chạy.
* Database `docmind` đã được tạo.
* File `.env` có `DATABASE_URL`.
* Prisma Schema đã được validate.
* Prisma Migration đã chạy thành công.
* Prisma Client đã được generate.
* Backend đã load file `.env`.

Kiểm tra `.env`:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

---

#### Bước 1: Kiểm tra PostgreSQL đang chạy

Nếu dùng PostgreSQL local:

```bash
pg_isready
```

Kết quả mong đợi:

```text
accepting connections
```

Nếu dùng Docker:

```bash
docker ps
```

Kiểm tra container PostgreSQL:

```bash
docker logs docmind-postgres
```

Nếu container đang chạy bình thường, bạn có thể tiếp tục bước tiếp theo.

---

#### Bước 2: Kiểm tra kết nối bằng psql

Chạy lệnh:

```bash
psql "postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Nếu kết nối thành công, bạn sẽ vào được PostgreSQL shell.

Kiểm tra danh sách bảng:

```sql
\dt
```

Các bảng mong đợi:

```text
User
Document
OCRResult
AIAnalysis
ChatHistory
Notification
AuditLog
```

Tên bảng thực tế có thể khác tùy cách Prisma map model sang PostgreSQL.

---

#### Bước 3: Kiểm tra Prisma Client

Chạy:

```bash
npx prisma generate
```

Sau đó kiểm tra Prisma Studio:

```bash
npx prisma studio
```

Prisma Studio thường mở tại:

```text
http://localhost:5555
```

Nếu các bảng hiển thị đúng, Prisma đã kết nối được với database.

---

#### Bước 4: Tạo API test database

Có thể tạo một endpoint test tạm thời trong backend:

```text
GET /api/health/database
```

Response mong đợi:

```json
{
  "success": true,
  "message": "Database connection is healthy",
  "database": "PostgreSQL",
  "orm": "Prisma"
}
```

Ví dụ logic kiểm tra:

```ts
import { prisma } from "../prisma/client";

export async function checkDatabaseHealth() {
  await prisma.$queryRaw`SELECT 1`;
  return {
    success: true,
    message: "Database connection is healthy",
    database: "PostgreSQL",
    orm: "Prisma"
  };
}
```

---

#### Bước 5: Test bằng Postman

Mở Postman và gọi:

```text
GET http://localhost:3000/api/health/database
```

Nếu kết nối thành công, response mong đợi:

```json
{
  "success": true,
  "message": "Database connection is healthy",
  "database": "PostgreSQL",
  "orm": "Prisma"
}
```

Nếu lỗi, backend cần trả về response rõ ràng thay vì crash.

---

#### Bước 6: Tạo dữ liệu User test

Bạn có thể tạo thử một user bằng Prisma Studio hoặc seed script.

Ví dụ seed:

```ts
await prisma.user.create({
  data: {
    email: "demo@docmind.ai",
    fullName: "Demo User",
    role: "USER"
  }
});
```

Sau đó kiểm tra lại bằng Prisma Studio.

Nếu schema có trường `passwordHash` bắt buộc, cần thêm giá trị:

```ts
passwordHash: "hashed_password_demo"
```

---

#### Bước 7: Tạo dữ liệu Document test

Tạo document test để kiểm tra quan hệ User → Document.

Ví dụ:

```ts
await prisma.document.create({
  data: {
    userId: "user_id_here",
    fileName: "invoice-demo.pdf",
    fileType: "application/pdf",
    fileSize: 204800,
    s3Bucket: "documind-document-storage",
    s3Key: "uploads/user_001/doc_001/invoice-demo.pdf",
    status: "UPLOADED"
  }
});
```

Sau đó kiểm tra bảng `Document`.

---

#### Bước 8: Kiểm tra relation dữ liệu

Dùng Prisma để lấy user kèm documents:

```ts
const user = await prisma.user.findFirst({
  include: {
    documents: true
  }
});

console.log(user);
```

Nếu kết quả trả về có danh sách documents, quan hệ giữa `User` và `Document` đã hoạt động.

---

#### Bước 9: Kiểm tra lưu OCR Result

Sau khi có document test, thử tạo OCR result:

```ts
await prisma.oCRResult.create({
  data: {
    documentId: "document_id_here",
    provider: "AWS_TEXTRACT",
    rawText: "Invoice No: INV-001\nTotal: 120 USD"
  }
});
```

Tên model có thể là `oCRResult`, `ocrResult` hoặc `OCRResult` tùy Prisma generate theo schema của bạn. Hãy chỉnh đúng theo dự án thật.

---

#### Bước 10: Kiểm tra lưu AI Analysis

Thử tạo AI analysis:

```ts
await prisma.aIAnalysis.create({
  data: {
    documentId: "document_id_here",
    provider: "openai",
    model: "gpt-4.1-mini",
    documentType: "Invoice",
    summary: "This document is an invoice.",
    entities: [
      {
        type: "totalAmount",
        value: "120 USD"
      }
    ],
    keywords: ["invoice", "payment"],
    importantInformation: ["Total amount: 120 USD"],
    confidence: 0.9
  }
});
```

Tên model có thể khác tùy schema. Nếu Prisma báo lỗi, kiểm tra lại tên model và kiểu dữ liệu Json.

---

#### Bước 11: Kiểm tra trạng thái Document

Sau khi test lưu OCR hoặc AI Analysis, kiểm tra cập nhật trạng thái document:

```ts
await prisma.document.update({
  where: {
    id: "document_id_here"
  },
  data: {
    status: "COMPLETED"
  }
});
```

Trạng thái mong đợi trong database:

```text
COMPLETED
```

---

#### Bước 12: Kiểm tra log backend

Khi test database, backend nên log:

```text
[DATABASE_HEALTH_CHECK_STARTED]
[DATABASE_CONNECTION_SUCCESS]
[DATABASE_TEST_USER_CREATED]
[DATABASE_TEST_DOCUMENT_CREATED]
```

Nếu có lỗi:

```text
[DATABASE_CONNECTION_FAILED]
```

Không log password hoặc toàn bộ `DATABASE_URL`.

---

#### Các lỗi thường gặp

| Lỗi                                            | Nguyên nhân                           | Cách xử lý                             |
| ---------------------------------------------- | ------------------------------------- | -------------------------------------- |
| `P1001`                                        | Không kết nối được database           | Kiểm tra PostgreSQL có đang chạy không |
| `P1000`                                        | Sai username/password                 | Kiểm tra `DATABASE_URL`                |
| `ECONNREFUSED`                                 | Database chưa chạy hoặc sai host/port | Start PostgreSQL hoặc kiểm tra port    |
| `relation does not exist`                      | Chưa chạy migration                   | Chạy `npx prisma migrate dev`          |
| `Environment variable not found: DATABASE_URL` | `.env` chưa đúng vị trí               | Kiểm tra file `.env` và dotenv         |
| `Unique constraint failed`                     | Tạo trùng email hoặc document         | Đổi dữ liệu test                       |
| `Invalid enum value`                           | Status không đúng enum                | Kiểm tra enum `DocumentStatus`         |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* PostgreSQL đang chạy.
* Backend đọc được `DATABASE_URL`.
* Prisma Client generate thành công.
* Prisma Studio mở được.
* API health database trả về thành công.
* Có thể tạo User test.
* Có thể tạo Document test.
* Có thể lưu OCRResult.
* Có thể lưu AIAnalysis.
* Có thể cập nhật Document status.
* Không còn lỗi kết nối database.

---

#### Kết quả kỳ vọng

Sau bước này, hệ thống DocuMind AI đã xác nhận kết nối database hoạt động ổn định. Backend có thể dùng Prisma để lưu và truy vấn dữ liệu, sẵn sàng kết nối với các module upload tài liệu, OCR bằng Amazon Textract, AI Analysis bằng Gemini/OpenAI, Notification và Admin Panel.
