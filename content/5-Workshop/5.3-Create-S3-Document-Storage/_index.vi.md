---

title : "Tạo S3 Document Storage"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
-----------------------

#### Tạo Amazon S3 Document Storage

Trong phần này, chúng ta sẽ thiết lập **Amazon S3** làm nơi lưu trữ tài liệu gốc cho hệ thống **DocuMind AI**. Đây là bước đầu tiên trong pipeline xử lý tài liệu của hệ thống.

Khi người dùng tải lên các tài liệu như PDF, PNG, JPG hoặc JPEG từ giao diện web, backend sẽ nhận file, kiểm tra định dạng, sau đó upload tài liệu lên **Amazon S3 Bucket**. Sau khi upload thành công, backend sẽ lưu metadata của tài liệu vào **PostgreSQL thông qua Prisma**, bao gồm tên file, loại file, dung lượng, `s3Bucket`, `s3Key` và trạng thái xử lý ban đầu.

Amazon S3 giúp hệ thống tách phần lưu trữ file ra khỏi backend server, đảm bảo tài liệu được lưu trữ ổn định, dễ mở rộng và có thể được worker sử dụng ở các bước tiếp theo như OCR bằng Amazon Textract và phân tích bằng Gemini/OpenAI.

---

#### Vai trò của Amazon S3 trong DocuMind AI

Trong dự án **DocuMind AI**, Amazon S3 được sử dụng để:

* Lưu trữ tài liệu gốc do người dùng upload.
* Hỗ trợ lưu các định dạng tài liệu phục vụ OCR như PDF, PNG, JPG và JPEG.
* Cung cấp `s3Key` để backend hoặc worker có thể truy xuất tài liệu.
* Tách phần lưu trữ file ra khỏi backend API.
* Giúp hệ thống dễ mở rộng khi số lượng tài liệu tăng lên.
* Bảo vệ tài liệu bằng bucket private, Block Public Access, encryption và IAM Policy.

Tài liệu trên S3 không được public trực tiếp ra internet. Người dùng chỉ nên truy cập tài liệu thông qua backend API sau khi hệ thống kiểm tra quyền hợp lệ.

---

#### Kiến trúc lưu trữ tài liệu

Luồng lưu trữ tài liệu trong phần này:

```text
User
  |
  v
React Frontend
  |
  v
Backend API Node.js/Express
  |
  v
Amazon S3 Bucket
  |
  v
PostgreSQL + Prisma lưu metadata
```

![s3-document-storage](/static/images/5-Workshop/5.3-Create-S3-Document-Storage/s3-document-storage.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ hoặc chụp sơ đồ minh họa luồng upload tài liệu gồm các thành phần: User, React Frontend, Node.js/Express Backend API, Amazon S3 Bucket và PostgreSQL + Prisma.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.3-Create-S3-Document-Storage/s3-document-storage.png`

---

#### Nội dung chi tiết

Trong phần **5.3 - Create S3 Document Storage**, chúng ta sẽ thực hiện các bước sau:

* [Tạo S3 Bucket](5.3.1-create-s3-bucket/)
* [Cấu hình CORS và bảo mật S3](5.3.2-configure-cors-and-security/)
* [Kiểm tra upload tài liệu lên S3](5.3.3-test-s3-upload/)

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được các kết quả sau:

* Đã tạo S3 bucket dùng để lưu tài liệu của DocuMind AI.
* Bucket được cấu hình private và bật Block Public Access.
* Bucket có cấu hình encryption để bảo vệ tài liệu.
* CORS được cấu hình phù hợp cho môi trường development hoặc production.
* Backend có thể upload file lên S3.
* File upload có `s3Key` rõ ràng.
* Metadata tài liệu được lưu vào PostgreSQL thông qua Prisma.
* Hệ thống sẵn sàng chuyển sang phần tiếp theo: gửi job xử lý tài liệu vào Amazon SQS.

---

#### Liên kết với bước tiếp theo

Sau khi tài liệu được upload thành công lên Amazon S3, backend sẽ tiếp tục gửi một message chứa thông tin tài liệu vào **Amazon SQS** ở phần **5.4 - Create SQS Processing Queue**. Message này sẽ giúp worker biết tài liệu nào cần được xử lý OCR và phân tích AI.

Luồng xử lý tiếp theo:

```text
Amazon S3
  |
  v
Amazon SQS
  |
  v
Worker
  |
  v
Amazon Textract
  |
  v
Gemini / OpenAI
```

Như vậy, phần 5.3 đóng vai trò là lớp lưu trữ đầu vào cho toàn bộ pipeline xử lý tài liệu của DocuMind AI.
