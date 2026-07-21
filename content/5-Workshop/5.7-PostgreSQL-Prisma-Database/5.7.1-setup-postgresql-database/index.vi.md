---

title : "Thiết lập PostgreSQL Database"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.7.1. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ thiết lập **PostgreSQL Database** cho hệ thống **DocuMind AI**.

PostgreSQL được sử dụng để lưu trữ dữ liệu chính của ứng dụng như thông tin người dùng, tài liệu upload, kết quả OCR, kết quả phân tích AI, lịch sử chat, thông báo và audit log. Đây là thành phần quan trọng giúp hệ thống quản lý trạng thái xử lý tài liệu từ lúc upload đến khi hoàn thành AI Analysis.

Trong workshop này, hệ thống sử dụng **PostgreSQL kết hợp Prisma ORM**, không sử dụng Amazon RDS. Database có thể chạy local trên máy cá nhân hoặc trên một PostgreSQL server riêng tùy môi trường triển khai.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Cài đặt hoặc chuẩn bị PostgreSQL Database.
* Tạo database riêng cho DocuMind AI.
* Cấu hình connection string `DATABASE_URL`.
* Kiểm tra backend có thể kết nối PostgreSQL.
* Chuẩn bị database để Prisma tạo bảng ở bước tiếp theo.

---

#### Vai trò của PostgreSQL trong DocuMind AI

Trong hệ thống **DocuMind AI**, PostgreSQL được dùng để lưu:

* Thông tin người dùng và phân quyền User/Admin.
* Metadata tài liệu upload.
* Đường dẫn file trên Amazon S3 thông qua `s3Bucket` và `s3Key`.
* Trạng thái xử lý tài liệu như `PENDING`, `QUEUED`, `PROCESSING`, `OCR_COMPLETED`, `AI_ANALYZING`, `COMPLETED`, `FAILED`.
* Kết quả OCR từ Amazon Textract.
* Kết quả AI Analysis từ Gemini/OpenAI.
* Lịch sử AI Chat/RAG.
* Notification cho User/Admin.
* Audit Log cho các thao tác quan trọng.

Luồng dữ liệu liên quan đến database:

```text
Backend API / Worker
  |
  v
Prisma ORM
  |
  v
PostgreSQL Database
```

![postgresql-database](/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.1-setup-postgresql-database/postgresql-database.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình PostgreSQL database đã được tạo trong pgAdmin, DBeaver, TablePlus hoặc terminal. Có thể chụp thêm sơ đồ Backend API → Prisma ORM → PostgreSQL Database.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.1-setup-postgresql-database/postgresql-database.png`

---

#### Bước 1: Cài đặt PostgreSQL

Nếu bạn chạy local, có thể cài PostgreSQL theo hệ điều hành đang dùng.

Với macOS, có thể dùng Homebrew:

```bash
brew install postgresql
brew services start postgresql
```

Với Ubuntu:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Với Windows, bạn có thể tải PostgreSQL từ trang cài đặt chính thức hoặc sử dụng Docker.

Nếu dùng Docker:

```bash
docker run --name docmind-postgres \
  -e POSTGRES_USER=docmind \
  -e POSTGRES_PASSWORD=docmind_password \
  -e POSTGRES_DB=docmind \
  -p 5432:5432 \
  -d postgres:16
```

---

#### Bước 2: Tạo database cho DocuMind AI

Nếu dùng terminal PostgreSQL:

```bash
psql -U postgres
```

Tạo user và database:

```sql
CREATE USER docmind WITH PASSWORD 'docmind_password';
CREATE DATABASE docmind OWNER docmind;
GRANT ALL PRIVILEGES ON DATABASE docmind TO docmind;
```

Thoát khỏi PostgreSQL:

```sql
\q
```

Nếu dùng pgAdmin, DBeaver hoặc TablePlus, bạn có thể tạo database bằng giao diện.

Tên database đề xuất:

```text
docmind
```

---

#### Bước 3: Cấu hình biến môi trường DATABASE_URL

Trong file `.env` backend, thêm connection string:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Giải thích:

| Thành phần         | Ý nghĩa                      |
| ------------------ | ---------------------------- |
| `postgresql://`    | Giao thức kết nối PostgreSQL |
| `docmind`          | Username                     |
| `docmind_password` | Password                     |
| `localhost`        | Host database                |
| `5432`             | Port mặc định của PostgreSQL |
| `docmind`          | Tên database                 |

Nếu database chạy trên server khác, thay `localhost` bằng IP hoặc domain của server đó.

---

#### Bước 4: Kiểm tra kết nối database

Bạn có thể kiểm tra bằng lệnh:

```bash
psql "postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Nếu kết nối thành công, PostgreSQL sẽ mở giao diện dòng lệnh.

Có thể kiểm tra danh sách bảng:

```sql
\dt
```

Ban đầu có thể chưa có bảng nào vì Prisma chưa chạy migration.

---

#### Bước 5: Kiểm tra backend đọc được DATABASE_URL

Trong backend, đảm bảo file `.env` được load đúng.

Nếu dùng Node.js, cần có package:

```bash
npm install dotenv
```

Trong file entry point, ví dụ `src/server.ts` hoặc `src/app.ts`, cần load:

```ts
import dotenv from "dotenv";

dotenv.config();
```

Sau đó Prisma sẽ sử dụng `DATABASE_URL` trong file `.env`.

---

#### Bước 6: Quy tắc bảo mật database

Khi dùng PostgreSQL cho DocuMind AI, cần lưu ý:

* Không commit `DATABASE_URL` thật lên GitHub.
* Không dùng password quá đơn giản trong production.
* Chỉ cho backend được kết nối database.
* Nếu database chạy trên server riêng, cần giới hạn IP truy cập.
* Không public database port `5432` ra internet nếu không cần.
* Nên dùng `.env.example` để mô tả biến môi trường mẫu.
* Nếu deploy production, có thể lưu `DATABASE_URL` trong AWS Secrets Manager.

---

#### Bước 7: Kiểm tra trạng thái database

Nếu chạy PostgreSQL bằng Docker:

```bash
docker ps
```

Kiểm tra log container:

```bash
docker logs docmind-postgres
```

Nếu chạy PostgreSQL service trên máy:

```bash
pg_isready
```

Kết quả mong đợi:

```text
accepting connections
```

---

#### Các lỗi thường gặp

| Lỗi                              | Nguyên nhân           | Cách xử lý                           |
| -------------------------------- | --------------------- | ------------------------------------ |
| `password authentication failed` | Sai username/password | Kiểm tra lại `DATABASE_URL`          |
| `database does not exist`        | Chưa tạo database     | Tạo database `docmind`               |
| `ECONNREFUSED`                   | PostgreSQL chưa chạy  | Start PostgreSQL service hoặc Docker |
| `port 5432 already in use`       | Port bị chiếm         | Đổi port hoặc dừng service khác      |
| `role does not exist`            | Chưa tạo user         | Tạo user PostgreSQL                  |
| Backend không đọc `.env`         | Chưa load dotenv      | Kiểm tra cấu hình backend            |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* PostgreSQL đã được cài đặt hoặc chạy bằng Docker.
* Đã tạo database cho DocuMind AI.
* Đã tạo user database.
* File `.env` có `DATABASE_URL`.
* Backend có thể đọc biến môi trường.
* Kết nối PostgreSQL thành công.
* Hệ thống sẵn sàng cấu hình Prisma schema ở bước 5.7.2.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có PostgreSQL Database để lưu dữ liệu ứng dụng. Đây là nền tảng để Prisma ORM tạo các bảng như User, Document, OCRResult, AIAnalysis, ChatHistory, Notification và AuditLog ở các bước tiếp theo.
