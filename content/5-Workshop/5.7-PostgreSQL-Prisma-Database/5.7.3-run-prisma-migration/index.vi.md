---

title : "Chạy Prisma Migration"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.7.3. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ chạy **Prisma Migration** để tạo các bảng cần thiết trong **PostgreSQL Database** cho hệ thống **DocuMind AI**.

Sau khi đã cấu hình `schema.prisma`, Prisma cần tạo migration để chuyển các model như `User`, `Document`, `OCRResult`, `AIAnalysis`, `ChatHistory`, `Notification` và `AuditLog` thành các bảng thật trong PostgreSQL.

Migration giúp quản lý thay đổi database một cách có kiểm soát, dễ theo dõi và phù hợp với môi trường phát triển backend.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Chạy migration đầu tiên cho DocuMind AI.
* Tạo các bảng trong PostgreSQL.
* Generate Prisma Client.
* Kiểm tra bảng bằng Prisma Studio.
* Kiểm tra backend có thể thao tác dữ liệu.
* Chuẩn bị database để kết nối với upload, OCR và AI Analysis.

---

#### Prisma Migration là gì?

**Prisma Migration** là cơ chế chuyển đổi schema trong file `schema.prisma` thành cấu trúc bảng trong database.

Ví dụ, khi bạn có model:

```prisma
model Document {
  id       String @id @default(uuid())
  fileName String
  s3Key    String
}
```

Prisma migration sẽ tạo bảng tương ứng trong PostgreSQL:

```text
Document
```

với các cột như:

```text
id
fileName
s3Key
```

Luồng hoạt động:

```text
schema.prisma
  |
  v
Prisma Migration
  |
  v
PostgreSQL Tables
  |
  v
Prisma Client
```

![prisma-migration](/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.3-run-prisma-migration/prisma-migration.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình terminal sau khi chạy `npx prisma migrate dev` thành công, hoặc chụp Prisma Studio hiển thị các bảng đã được tạo.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.7-PostgreSQL-Prisma-Database/5.7.3-run-prisma-migration/prisma-migration.png`

---

#### Điều kiện trước khi chạy migration

Trước khi chạy migration, hãy đảm bảo:

* PostgreSQL đang chạy.
* Database `docmind` đã được tạo.
* File `.env` có `DATABASE_URL`.
* File `schema.prisma` đã cấu hình datasource PostgreSQL.
* Các model Prisma đã được định nghĩa.
* `npx prisma validate` chạy thành công.

Kiểm tra lại `DATABASE_URL`:

```env
DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"
```

---

#### Bước 1: Format và validate schema

Chạy lệnh format:

```bash
npx prisma format
```

Sau đó kiểm tra schema:

```bash
npx prisma validate
```

Nếu schema hợp lệ, Prisma sẽ trả về thông báo không có lỗi.

---

#### Bước 2: Chạy migration đầu tiên

Chạy lệnh:

```bash
npx prisma migrate dev --name init
```

Lệnh này sẽ:

* Tạo folder migration trong thư mục `prisma/migrations`.
* Tạo các bảng trong PostgreSQL.
* Generate Prisma Client.
* Đồng bộ database với schema hiện tại.

Sau khi chạy thành công, bạn sẽ thấy thư mục dạng:

```text
prisma/
└── migrations/
    └── 20260610120000_init/
        └── migration.sql
```

---

#### Bước 3: Generate Prisma Client

Thông thường `migrate dev` đã tự generate Prisma Client. Nếu cần chạy thủ công:

```bash
npx prisma generate
```

Prisma Client giúp backend truy vấn database bằng TypeScript.

Ví dụ:

```ts
const documents = await prisma.document.findMany();
```

---

#### Bước 4: Kiểm tra bảng bằng Prisma Studio

Chạy:

```bash
npx prisma studio
```

Prisma Studio sẽ mở trình duyệt, thường tại:

```text
http://localhost:5555
```

Kiểm tra các bảng đã được tạo:

```text
User
Document
OCRResult
AIAnalysis
ChatHistory
Notification
AuditLog
```

Nếu các bảng hiển thị đúng, migration đã thành công.

---

#### Bước 5: Tạo dữ liệu test tùy chọn

Bạn có thể tạo thử một user hoặc document bằng Prisma Studio hoặc seed script.

Ví dụ seed đơn giản:

```ts
await prisma.user.create({
  data: {
    email: "demo@docmind.ai",
    fullName: "Demo User",
    role: "USER"
  }
});
```

Nếu có trường `passwordHash` bắt buộc, cần thêm giá trị phù hợp.

---

#### Bước 6: Kiểm tra backend kết nối Prisma

Tạo hoặc kiểm tra file Prisma Client:

```text
src/prisma/client.ts
```

Ví dụ:

```ts
import { PrismaClient } from "@prisma/client";

export const prisma = new PrismaClient();
```

Test kết nối trong backend:

```ts
const users = await prisma.user.findMany();
console.log(users);
```

Nếu không lỗi, backend đã kết nối database thành công.

---

#### Bước 7: Migration khi thay đổi schema

Khi bạn thêm hoặc sửa model trong `schema.prisma`, chạy migration mới:

```bash
npx prisma migrate dev --name add_notification_table
```

Không nên sửa trực tiếp database bằng tay nếu đang dùng Prisma migration, vì có thể làm schema và database lệch nhau.

---

#### Bước 8: Lệnh cho production

Trong môi trường production, không dùng `migrate dev`. Nên dùng:

```bash
npx prisma migrate deploy
```

Lệnh này sẽ áp dụng migration đã có sẵn vào database production mà không tạo migration mới.

Với workshop local, dùng:

```bash
npx prisma migrate dev
```

là phù hợp.

---

#### Các lỗi thường gặp

| Lỗi                       | Nguyên nhân                 | Cách xử lý                             |
| ------------------------- | --------------------------- | -------------------------------------- |
| `P1001`                   | Không kết nối được database | Kiểm tra PostgreSQL có đang chạy không |
| `P1000`                   | Sai username/password       | Kiểm tra `DATABASE_URL`                |
| `P1012`                   | Prisma schema sai           | Chạy `npx prisma validate`             |
| `database does not exist` | Chưa tạo database           | Tạo database `docmind`                 |
| `shadow database` error   | Quyền database chưa đủ      | Kiểm tra quyền user PostgreSQL         |
| Prisma Studio không mở    | Port bị chiếm               | Chạy lại hoặc đổi port                 |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* `npx prisma format` chạy thành công.
* `npx prisma validate` không báo lỗi.
* `npx prisma migrate dev --name init` chạy thành công.
* Prisma Client đã được generate.
* Prisma Studio mở được.
* Các bảng chính đã xuất hiện trong PostgreSQL.
* Backend có thể truy vấn database bằng Prisma Client.
* Database sẵn sàng cho upload, OCR, AI Analysis, Notification và Admin.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có database schema hoạt động trên PostgreSQL. Các bảng dữ liệu chính đã được tạo và backend có thể sử dụng Prisma Client để lưu metadata tài liệu, kết quả OCR, AI Analysis, lịch sử chat, notification và audit log.
