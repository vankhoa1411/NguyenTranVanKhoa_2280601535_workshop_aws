---

title : "PostgreSQL và Prisma Database"
date : 2026-06-10
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
-----------------------

#### Thiết lập PostgreSQL và Prisma Database

Trong phần này, chúng ta sẽ thiết lập **PostgreSQL Database** và **Prisma ORM** cho hệ thống **DocuMind AI**.

PostgreSQL được sử dụng để lưu trữ toàn bộ dữ liệu chính của ứng dụng như người dùng, tài liệu, kết quả OCR, kết quả phân tích AI, lịch sử chat, thông báo và audit log. Prisma ORM giúp backend Node.js/TypeScript làm việc với PostgreSQL dễ dàng hơn thông qua schema, migration và Prisma Client.

Trong workshop này, hệ thống sử dụng **PostgreSQL + Prisma**, không sử dụng Amazon RDS. Database có thể chạy local bằng PostgreSQL service, Docker hoặc một PostgreSQL server riêng.

---

#### Vai trò của PostgreSQL trong DocuMind AI

Trong dự án **DocuMind AI**, PostgreSQL được sử dụng để lưu:

* Thông tin tài khoản người dùng.
* Phân quyền User/Admin.
* Metadata tài liệu upload.
* Đường dẫn file trên Amazon S3 thông qua `s3Bucket` và `s3Key`.
* Trạng thái xử lý tài liệu.
* Kết quả OCR từ Amazon Textract.
* Kết quả AI Analysis từ Gemini/OpenAI.
* Lịch sử AI Chat/RAG.
* Notification cho User/Admin.
* Audit Log cho các hành động quan trọng.

---

#### Kiến trúc Database

Luồng dữ liệu database trong DocuMind AI:

```text
Backend API / Worker
  |
  v
Prisma ORM
  |
  v
PostgreSQL Database
```

![postgresql-prisma-flow](/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/postgresql-prisma-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa Backend API và Worker sử dụng Prisma ORM để đọc/ghi dữ liệu vào PostgreSQL Database.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/postgresql-prisma-flow.png`

---

#### Các bảng dữ liệu chính

Database của DocuMind AI nên có các bảng chính sau:

| Bảng           | Mục đích                                     |
| -------------- | -------------------------------------------- |
| `User`         | Lưu tài khoản người dùng và quyền User/Admin |
| `Document`     | Lưu metadata tài liệu upload                 |
| `OCRResult`    | Lưu nội dung OCR từ Amazon Textract          |
| `AIAnalysis`   | Lưu kết quả phân tích từ Gemini/OpenAI       |
| `ChatHistory`  | Lưu lịch sử hỏi đáp tài liệu                 |
| `Notification` | Lưu thông báo cho User/Admin                 |
| `AuditLog`     | Lưu nhật ký hành động quan trọng             |

---

#### Trạng thái xử lý tài liệu

Trong pipeline xử lý tài liệu, bảng `Document` cần lưu trạng thái xử lý để frontend hiển thị tiến trình cho người dùng.

Các trạng thái có thể dùng:

```text
PENDING
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

Ý nghĩa:

| Trạng thái      | Ý nghĩa                        |
| --------------- | ------------------------------ |
| `PENDING`       | Tài liệu mới được tạo metadata |
| `UPLOADED`      | File đã upload lên S3          |
| `QUEUED`        | Job đã được gửi vào SQS        |
| `PROCESSING`    | Worker đang xử lý tài liệu     |
| `OCR_COMPLETED` | Textract OCR hoàn tất          |
| `AI_ANALYZING`  | Gemini/OpenAI đang phân tích   |
| `COMPLETED`     | Tài liệu đã xử lý xong         |
| `FAILED`        | Quá trình xử lý lỗi            |

---

#### Nội dung chi tiết

Trong phần **5.7 - PostgreSQL Prisma Database**, chúng ta sẽ thực hiện các bước sau:

* [Thiết lập PostgreSQL Database](5.7.1-setup-postgresql-database/)
* [Cấu hình Prisma Schema](5.7.2-configure-prisma-schema/)
* [Chạy Prisma Migration](5.7.3-run-prisma-migration/)
* [Kiểm tra kết nối Database](5.7.4-test-database-connection/)

---

#### Biến môi trường liên quan

Backend cần biến môi trường:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Nếu dùng Docker PostgreSQL local:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Nếu database nằm trên server riêng, thay `localhost` bằng IP hoặc domain của server đó.

> ⚠️ **Lưu ý bảo mật:**
> Không commit `DATABASE_URL` thật lên GitHub. Chỉ commit file `.env.example`. Nếu deploy production, có thể lưu `DATABASE_URL` trong AWS Secrets Manager.

---

#### Liên kết với các phần khác

PostgreSQL được sử dụng xuyên suốt pipeline:

```text
Upload Document
  |
  v
Save Document Metadata
  |
  v
SQS Worker updates status
  |
  v
Textract saves OCRResult
  |
  v
Gemini/OpenAI saves AIAnalysis
  |
  v
Dashboard displays result
```

Nhờ đó, frontend có thể hiển thị trạng thái xử lý tài liệu, kết quả OCR, phân tích AI, lịch sử chat và notification.

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được:

* PostgreSQL Database đã được tạo.
* Backend có `DATABASE_URL` hợp lệ.
* Prisma Schema đã được cấu hình.
* Các model chính đã được định nghĩa.
* Prisma Migration đã tạo bảng thành công.
* Prisma Client đã được generate.
* Backend có thể đọc/ghi dữ liệu vào PostgreSQL.
* Database sẵn sàng phục vụ upload, OCR, AI Analysis, Notification và Admin Panel.

---

#### Kết quả kỳ vọng

Sau phần này, DocuMind AI sẽ có lớp dữ liệu ổn định để quản lý toàn bộ quá trình xử lý tài liệu. PostgreSQL và Prisma giúp hệ thống lưu trữ dữ liệu có cấu trúc, dễ truy vấn, dễ mở rộng và phù hợp với backend Node.js/TypeScript.
