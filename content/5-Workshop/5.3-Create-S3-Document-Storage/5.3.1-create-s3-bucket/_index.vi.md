---

title : "Tạo S3 Bucket"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.3.1. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tạo **Amazon S3 Bucket** để lưu trữ tài liệu gốc cho hệ thống **DocuMind AI**.

Amazon S3 đóng vai trò là nơi lưu file đầu vào của người dùng, ví dụ như hóa đơn, biểu mẫu, hợp đồng hoặc tài liệu nội bộ. Sau khi người dùng upload tài liệu từ giao diện web, backend sẽ đưa file lên S3 và lưu lại thông tin metadata vào **PostgreSQL + Prisma**.

File được lưu trên S3 sẽ tiếp tục được sử dụng ở các bước sau, đặc biệt là khi worker lấy tài liệu để xử lý OCR bằng **Amazon Textract**.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được một S3 bucket dùng riêng cho DocuMind AI.
* Cấu hình bucket ở chế độ private.
* Bật Block Public Access để bảo vệ tài liệu.
* Bật encryption để tăng mức độ bảo mật.
* Hiểu được vai trò của `bucketName` và `s3Key` trong hệ thống.
* Chuẩn bị bucket để backend upload tài liệu lên S3.

---

#### Vai trò của S3 Bucket trong DocuMind AI

Trong hệ thống DocuMind AI, S3 Bucket được dùng để:

* Lưu file tài liệu gốc do người dùng upload.
* Cung cấp đường dẫn `s3Key` để worker có thể lấy file xử lý.
* Tách phần lưu trữ file khỏi backend server.
* Giúp hệ thống dễ mở rộng khi số lượng tài liệu tăng.
* Đảm bảo tài liệu không bị public trực tiếp ra internet.

Luồng xử lý trong bước này:

```text
User
  |
  v
React Frontend
  |
  v
Backend API
  |
  v
Amazon S3 Bucket
  |
  v
PostgreSQL + Prisma lưu metadata
```

![create-s3-bucket](/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.1-create-s3-bucket/create-s3-bucket.png)

:
> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/5.3.1-create-s3-bucket/create-s3-bucket.png`

---

#### Bước 1: Truy cập dịch vụ Amazon S3

Đăng nhập vào AWS Console, sau đó tìm kiếm dịch vụ:

```text
S3
```

Chọn:

```text
Amazon S3
```

Sau đó nhấn:

```text
Create bucket
```

---

#### Bước 2: Đặt tên bucket

Tại mục **Bucket name**, đặt tên bucket cho dự án.

Ví dụ:

```text
documind-document-storage
```

Nếu tên bucket đã bị trùng, bạn có thể thêm tên cá nhân hoặc mã project:

```text
documind-document-storage-tuan
documind-document-storage-2026
documind-ai-documents-tuan
```

> ⚠️ **Lưu ý:**
> Tên S3 bucket phải là duy nhất trên toàn bộ AWS, không chỉ riêng tài khoản của bạn.

---

#### Bước 3: Chọn AWS Region

Chọn region bạn sử dụng cho toàn bộ workshop.

Ví dụ:

```text
Asia Pacific (Singapore) - ap-southeast-1
```

Hoặc region bạn đã thống nhất trong file `.env`:

```env
AWS_REGION="ap-southeast-1"
```

Khuyến nghị nên dùng cùng một region cho:

* S3
* SQS
* Textract
* CloudWatch
* Secrets Manager
* Backend/EC2 nếu có

Việc dùng cùng region giúp giảm lỗi cấu hình và giúp hệ thống dễ quản lý hơn.

---

#### Bước 4: Cấu hình Object Ownership

Ở phần **Object Ownership**, chọn:

```text
ACLs disabled
```

Cấu hình này giúp quyền sở hữu object thuộc về bucket owner và hạn chế việc dùng ACL thủ công.

Khuyến nghị:

```text
Object Ownership: Bucket owner enforced
ACLs: Disabled
```

---

#### Bước 5: Bật Block Public Access

Tại phần **Block Public Access settings for this bucket**, giữ nguyên cấu hình:

```text
Block all public access: Enabled
```

Đây là cấu hình rất quan trọng vì tài liệu người dùng không nên được public trực tiếp ra internet.

DocuMind AI nên kiểm soát quyền xem tài liệu thông qua backend API, không để người dùng truy cập trực tiếp public URL của S3.

---

#### Bước 6: Bật Bucket Versioning

Ở phần **Bucket Versioning**, bạn có thể chọn:

```text
Disable
```

hoặc:

```text
Enable
```

Đối với workshop/demo, có thể để:

```text
Disable
```

Nếu muốn lưu nhiều phiên bản của cùng một tài liệu, có thể bật:

```text
Enable
```

Tuy nhiên, nếu bật versioning thì chi phí lưu trữ có thể tăng khi upload nhiều file hoặc ghi đè file nhiều lần.

---

#### Bước 7: Cấu hình encryption

Tại phần **Default encryption**, nên bật encryption mặc định.

Chọn:

```text
Server-side encryption with Amazon S3 managed keys (SSE-S3)
```

Cấu hình khuyến nghị:

```text
Default encryption: Enabled
Encryption type: SSE-S3
Bucket key: Enable
```

Việc bật encryption giúp tài liệu được mã hóa khi lưu trữ trên S3.

---

#### Bước 8: Tạo bucket

Sau khi kiểm tra lại các thông tin, nhấn:

```text
Create bucket
```

Sau khi tạo thành công, bạn sẽ thấy bucket xuất hiện trong danh sách S3 buckets.

---

#### Bước 9: Kiểm tra bucket sau khi tạo

Chọn bucket vừa tạo và kiểm tra các phần sau:

| Thành phần       | Trạng thái mong muốn      |
| ---------------- | ------------------------- |
| Bucket name      | Đúng tên bucket của dự án |
| Region           | Đúng region đang dùng     |
| Public access    | Blocked                   |
| Encryption       | Enabled                   |
| Object ownership | ACLs disabled             |

---

#### Bước 10: Cập nhật biến môi trường backend

Sau khi có bucket, thêm thông tin bucket vào file `.env` backend:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
```

Nếu bucket của bạn có tên khác thì thay lại cho đúng.

Ví dụ:

```env
AWS_S3_BUCKET_NAME="documind-document-storage-tuan"
```

---

#### Cách đặt `s3Key` trong dự án

Backend nên lưu file theo cấu trúc rõ ràng để dễ quản lý:

```text
uploads/{userId}/{documentId}/{fileName}
```

Ví dụ:

```text
uploads/user_001/doc_001/invoice-demo.pdf
```

Cách này giúp hệ thống dễ tìm tài liệu theo user và document.

Ví dụ metadata lưu trong PostgreSQL:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "fileName": "invoice-demo.pdf",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "status": "UPLOADED"
}
```

---

#### Kiểm tra hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo S3 bucket thành công.
* Bucket không public.
* Bucket đã bật encryption.
* Bucket name đã được thêm vào file `.env`.
* Backend có thể sử dụng `AWS_S3_BUCKET_NAME`.
* Bạn đã hiểu vai trò của `s3Bucket` và `s3Key` trong hệ thống.

---

#### Kết quả kỳ vọng

Sau bước này, hệ thống DocuMind AI đã có nơi lưu trữ tài liệu gốc an toàn bằng Amazon S3. Đây là nền tảng để các bước tiếp theo như cấu hình CORS, bảo mật bucket, upload file, gửi job sang SQS và xử lý OCR bằng Textract có thể hoạt động đúng.
