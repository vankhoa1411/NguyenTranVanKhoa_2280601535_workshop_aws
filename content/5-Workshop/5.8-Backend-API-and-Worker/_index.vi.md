---

title : "Backend API và Worker"
date : 2026-06-10
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
-----------------------

#### Xây dựng Backend API và Worker

Trong phần này, chúng ta sẽ xây dựng các thành phần backend chính cho hệ thống **DocuMind AI**, bao gồm **Document Upload API**, **SQS Worker Processing** và **Document Status API**.

Sau khi các dịch vụ nền tảng như **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Gemini/OpenAI** và **PostgreSQL + Prisma** đã được chuẩn bị, backend sẽ đóng vai trò điều phối toàn bộ luồng xử lý tài liệu. Backend nhận file từ người dùng, upload tài liệu lên S3, lưu metadata vào PostgreSQL, gửi message vào SQS và cung cấp API để frontend theo dõi trạng thái xử lý.

Worker là tiến trình nền nhận message từ SQS, gọi Textract để OCR tài liệu, sau đó gửi nội dung OCR sang AI Gateway để phân tích bằng Gemini hoặc OpenAI. Kết quả cuối cùng được lưu vào PostgreSQL và hiển thị lại trên Dashboard.

---

#### Vai trò của Backend API trong DocuMind AI

Trong dự án **DocuMind AI**, Backend API có nhiệm vụ:

* Xác thực người dùng bằng JWT.
* Nhận tài liệu upload từ frontend.
* Kiểm tra định dạng và dung lượng file.
* Upload tài liệu lên Amazon S3.
* Lưu metadata tài liệu vào PostgreSQL thông qua Prisma.
* Gửi job xử lý tài liệu vào Amazon SQS.
* Cung cấp API lấy danh sách tài liệu.
* Cung cấp API lấy trạng thái xử lý tài liệu.
* Trả kết quả OCR và AI Analysis cho frontend.
* Kiểm tra quyền truy cập tài liệu giữa User và Admin.

---

#### Vai trò của Worker trong DocuMind AI

Worker là tiến trình xử lý nền, không trực tiếp nhận request từ người dùng. Worker có nhiệm vụ:

* Poll message từ Amazon SQS.
* Đọc thông tin `documentId`, `userId`, `s3Bucket` và `s3Key`.
* Cập nhật trạng thái tài liệu sang `PROCESSING`.
* Gọi Amazon Textract để OCR tài liệu.
* Lưu kết quả OCR vào PostgreSQL.
* Gửi OCR text sang AI Gateway.
* Gọi Gemini hoặc OpenAI để phân tích tài liệu.
* Lưu kết quả AI Analysis vào PostgreSQL.
* Cập nhật trạng thái tài liệu thành `COMPLETED` hoặc `FAILED`.
* Xóa message khỏi SQS khi xử lý thành công.
* Ghi log xử lý vào console hoặc CloudWatch.

---

#### Kiến trúc Backend API và Worker

Luồng xử lý tổng thể trong phần này:

```text
React Frontend
  |
  v
Backend API Node.js/Express
  |
  |-- Upload file --> Amazon S3
  |
  |-- Save metadata --> PostgreSQL + Prisma
  |
  |-- Send job --> Amazon SQS
                         |
                         v
                    SQS Worker
                         |
                         v
                  Amazon Textract OCR
                         |
                         v
                    AI Gateway
                         |
          |--------------|--------------|
          v                             v
     Gemini API                  OpenAI ChatGPT API
          |--------------|--------------|
                         v
               PostgreSQL + Prisma
                         |
                         v
              Dashboard / Notification
```

![backend-api-worker](/static/images/5-Workshop/5.8-Backend-API-and-Worker/backend-api-worker.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa luồng xử lý gồm các thành phần: React Frontend, Backend API, Amazon S3, PostgreSQL + Prisma, Amazon SQS, SQS Worker, Amazon Textract, AI Gateway, Gemini API và OpenAI ChatGPT API.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/backend-api-worker.png`

---

#### Workflow xử lý tài liệu

Quy trình xử lý tài liệu trong DocuMind AI:

1. Người dùng đăng nhập vào hệ thống.
2. Người dùng upload tài liệu từ frontend.
3. Backend kiểm tra file type và file size.
4. Backend upload file lên Amazon S3.
5. Backend lưu metadata tài liệu vào PostgreSQL.
6. Backend gửi message xử lý vào Amazon SQS.
7. Worker nhận message từ SQS.
8. Worker gọi Amazon Textract để OCR tài liệu.
9. Worker lưu OCR Result vào PostgreSQL.
10. Worker gửi OCR text sang AI Gateway.
11. Gemini hoặc OpenAI phân tích tài liệu.
12. Worker lưu AI Analysis vào PostgreSQL.
13. Worker cập nhật trạng thái tài liệu thành `COMPLETED`.
14. Frontend gọi Document Status API để hiển thị kết quả.
15. Notification System gửi thông báo cho người dùng hoặc Admin.

---

#### Trạng thái xử lý tài liệu

Trong phần Backend API và Worker, trạng thái tài liệu được cập nhật xuyên suốt pipeline:

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

| Trạng thái      | Ý nghĩa                      |
| --------------- | ---------------------------- |
| `PENDING`       | Tạo metadata ban đầu         |
| `UPLOADED`      | File đã upload lên S3        |
| `QUEUED`        | Job đã gửi vào SQS           |
| `PROCESSING`    | Worker đang xử lý            |
| `OCR_COMPLETED` | Textract OCR hoàn tất        |
| `AI_ANALYZING`  | Gemini/OpenAI đang phân tích |
| `COMPLETED`     | Tài liệu xử lý xong          |
| `FAILED`        | Quá trình xử lý thất bại     |

Frontend Dashboard sẽ dựa vào các trạng thái này để hiển thị tiến trình xử lý cho người dùng.

---

#### Nội dung chi tiết

Trong phần **5.8 - Backend API and Worker**, chúng ta sẽ thực hiện các bước sau:

* [Xây dựng Document Upload API](5.8.1-document-upload-api/)
* [Xử lý SQS Worker](5.8.2-sqs-worker-processing/)
* [Xây dựng Document Status API](5.8.3-document-status-api/)

---

#### Cấu trúc source code đề xuất

Cấu trúc backend có thể tổ chức như sau:

```text
src/
├── controllers/
│   └── document.controller.ts
├── routes/
│   └── document.routes.ts
├── services/
│   ├── document.service.ts
│   ├── s3.service.ts
│   ├── sqs.service.ts
│   ├── textract.service.ts
│   └── ai/
│       ├── ai.gateway.ts
│       ├── gemini.service.ts
│       └── openai.service.ts
├── workers/
│   └── document.worker.ts
├── middlewares/
│   ├── auth.middleware.ts
│   └── upload.middleware.ts
├── prisma/
│   └── client.ts
└── config/
    └── aws.config.ts
```

---

#### Các API chính

Các API quan trọng trong phần này:

| API                               | Mục đích                        |
| --------------------------------- | ------------------------------- |
| `POST /api/documents/upload`      | Upload tài liệu lên hệ thống    |
| `GET /api/documents`              | Lấy danh sách tài liệu của user |
| `GET /api/documents/:id`          | Lấy chi tiết tài liệu           |
| `GET /api/documents/:id/status`   | Lấy trạng thái xử lý tài liệu   |
| `GET /api/documents/:id/ocr`      | Lấy kết quả OCR                 |
| `GET /api/documents/:id/analysis` | Lấy kết quả AI Analysis         |

Nếu có Admin Panel, có thể mở rộng thêm:

| API                                   | Mục đích                             |
| ------------------------------------- | ------------------------------------ |
| `GET /api/admin/documents`            | Admin xem toàn bộ tài liệu           |
| `GET /api/admin/documents/:id/status` | Admin xem trạng thái tài liệu bất kỳ |
| `GET /api/admin/processing-errors`    | Admin xem các lỗi xử lý              |

---

#### Biến môi trường liên quan

Backend và worker cần các biến môi trường sau:

```env
NODE_ENV="development"
PORT=3000

DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"

AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"

AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

> ⚠️ **Lưu ý bảo mật:**
> Không commit file `.env` thật lên GitHub. Trong production, nên dùng AWS Secrets Manager và IAM Role thay vì hardcode credentials.

---

#### Logging cần có

Backend và worker nên ghi log theo các mốc:

```text
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[DOCUMENT_METADATA_SAVED]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_STATUS_COMPLETED]
[DOCUMENT_PROCESSING_FAILED]
```

Nếu đã tích hợp CloudWatch, các log này nên được gửi vào log group:

```text
/docmind/application
```

Không nên log API key, JWT token hoặc toàn bộ nội dung tài liệu nhạy cảm.

---

#### Quy tắc bảo mật

Khi xây dựng Backend API và Worker, cần đảm bảo:

* Chỉ user đã đăng nhập mới được upload tài liệu.
* User chỉ xem được tài liệu của chính họ.
* Admin có thể xem toàn bộ tài liệu nếu có quyền phù hợp.
* File upload phải kiểm tra định dạng và dung lượng.
* Không expose S3 object public trực tiếp.
* Không đưa Gemini/OpenAI API key vào frontend.
* Không hardcode AWS Access Key trong production.
* Worker chỉ xóa SQS message khi xử lý thành công.
* Lỗi xử lý cần được log và cập nhật trạng thái `FAILED`.

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được:

* Backend có API upload tài liệu.
* File được lưu vào Amazon S3.
* Metadata được lưu vào PostgreSQL.
* Message xử lý được gửi vào Amazon SQS.
* Worker có thể nhận message từ SQS.
* Worker xử lý OCR bằng Amazon Textract.
* Worker gọi AI Gateway để phân tích tài liệu.
* Kết quả OCR và AI Analysis được lưu vào PostgreSQL.
* Document Status API trả về trạng thái xử lý đúng.
* Frontend có thể theo dõi tiến trình xử lý tài liệu.

---

#### Kết quả kỳ vọng

Sau phần này, DocuMind AI sẽ có backend pipeline hoàn chỉnh từ upload tài liệu đến xử lý OCR và AI Analysis. Đây là phần lõi giúp hệ thống hoạt động thực tế, kết nối các dịch vụ AWS, PostgreSQL, Gemini/OpenAI và frontend dashboard thành một quy trình xử lý tài liệu tự động.
