---

title : "Tích hợp Textract OCR"
date : 2026-06-10
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
-----------------------

#### Tích hợp Amazon Textract OCR

Trong phần này, chúng ta sẽ tích hợp **Amazon Textract** vào hệ thống **DocuMind AI** để trích xuất văn bản từ tài liệu đã được upload lên **Amazon S3**.

Sau khi tài liệu được lưu vào S3 và message xử lý được gửi vào **Amazon SQS**, worker sẽ nhận message từ queue, lấy thông tin `s3Bucket` và `s3Key`, sau đó gọi **Amazon Textract** để thực hiện OCR. Kết quả OCR sẽ được lưu vào **PostgreSQL thông qua Prisma** và được sử dụng làm dữ liệu đầu vào cho bước phân tích AI bằng **Gemini API** hoặc **OpenAI ChatGPT API** ở phần tiếp theo.

Amazon Textract đóng vai trò quan trọng trong pipeline vì hệ thống cần chuyển tài liệu dạng PDF hoặc hình ảnh thành văn bản trước khi AI có thể tóm tắt, phân loại, trích xuất thông tin hoặc hỗ trợ hỏi đáp tài liệu.

---

#### Vai trò của Amazon Textract trong DocuMind AI

Trong dự án **DocuMind AI**, Amazon Textract được sử dụng để:

* Trích xuất văn bản từ tài liệu PDF, PNG, JPG hoặc JPEG.
* Chuyển tài liệu phi cấu trúc thành dữ liệu văn bản.
* Cung cấp nội dung đầu vào cho Gemini/OpenAI AI Gateway.
* Hỗ trợ các chức năng AI Analysis, AI Chat/RAG, Semantic Search và Enterprise Sandbox.
* Giảm thao tác đọc và nhập liệu thủ công.
* Tự động hóa bước OCR trong quy trình xử lý tài liệu.

Textract không thay thế AI provider. Textract chỉ thực hiện OCR, còn phần hiểu nội dung, tóm tắt, phân loại và hỏi đáp sẽ được xử lý bởi Gemini/OpenAI ở phần sau.

---

#### Kiến trúc OCR Processing

Luồng xử lý OCR trong phần này:

```text
Amazon SQS
  |
  v
OCR Worker
  |
  v
Amazon S3 lấy tài liệu
  |
  v
Amazon Textract OCR
  |
  v
PostgreSQL + Prisma lưu OCR Result
  |
  v
Chuẩn bị dữ liệu cho Gemini/OpenAI
```

![textract-ocr-flow](/static/images/5-Workshop/5.5-Integrate-Textract-OCR/textract-ocr-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ hoặc chụp sơ đồ minh họa luồng xử lý OCR gồm các thành phần: Amazon SQS, OCR Worker, Amazon S3, Amazon Textract và PostgreSQL + Prisma.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/textract-ocr-flow.png`

---

#### Workflow xử lý OCR

Quy trình OCR trong DocuMind AI hoạt động như sau:

1. Người dùng upload tài liệu lên hệ thống.
2. Backend lưu tài liệu vào Amazon S3.
3. Backend gửi message xử lý vào Amazon SQS.
4. OCR Worker nhận message từ SQS.
5. Worker lấy `documentId`, `userId`, `s3Bucket` và `s3Key`.
6. Worker gọi Amazon Textract để OCR tài liệu.
7. Kết quả OCR được lưu vào PostgreSQL.
8. Trạng thái tài liệu được cập nhật thành `OCR_COMPLETED`.
9. Nội dung OCR được chuyển sang bước AI Analysis.

Các trạng thái tài liệu có thể dùng trong phần này:

```text
QUEUED
PROCESSING
OCR_COMPLETED
FAILED
```

Sau khi tích hợp AI ở phần 5.6, hệ thống có thể mở rộng thêm:

```text
AI_ANALYZING
COMPLETED
```

---

#### Định dạng tài liệu hỗ trợ

Trong workshop này, tài liệu nên sử dụng các định dạng phù hợp với Amazon Textract:

```text
PDF
PNG
JPG
JPEG
```

Không nên upload trực tiếp các định dạng sau vào pipeline OCR:

```text
DOCX
XLSX
PPTX
EXE
SH
BAT
```

Nếu cần xử lý tài liệu DOCX, XLSX hoặc PPTX, nên chuyển đổi sang PDF trước khi upload vào hệ thống.

---

#### Nội dung chi tiết

Trong phần **5.5 - Integrate Textract OCR**, chúng ta sẽ thực hiện các bước sau:

* [Cấu hình quyền Amazon Textract](5.5.1-configure-textract-permission/)
* [Tạo OCR Worker](5.5.2-create-ocr-worker/)
* [Kiểm tra xử lý OCR](5.5.3-test-ocr-processing/)

---

#### Dữ liệu OCR lưu trong database

Sau khi Textract xử lý thành công, hệ thống nên lưu kết quả OCR vào bảng riêng, ví dụ `OCRResult`.

Ví dụ dữ liệu OCR:

```json
{
  "id": "ocr_001",
  "documentId": "doc_001",
  "provider": "AWS_TEXTRACT",
  "rawText": "Extracted text from document...",
  "createdAt": "2026-06-10T10:00:00.000Z"
}
```

Bảng `Document` cũng cần được cập nhật trạng thái:

```text
OCR_COMPLETED
```

Nếu OCR thất bại, trạng thái nên là:

```text
FAILED
```

và hệ thống cần lưu lại lỗi để người dùng hoặc Admin có thể kiểm tra.

---

#### Biến môi trường liên quan

Backend/worker cần các biến môi trường sau:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"

DATABASE_URL="postgresql://username:password@localhost:5432/docmind"
```

Nếu muốn tách region riêng cho Textract, có thể thêm:

```env
AWS_TEXTRACT_REGION="ap-southeast-1"
```

Trong production, backend/worker nên dùng **IAM Role** thay vì hardcode AWS Access Key trong file `.env`.

---

#### Quyền IAM cần thiết

Worker cần các quyền chính sau:

```text
s3:GetObject
s3:ListBucket
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
textract:DetectDocumentText
textract:AnalyzeDocument
```

Nếu sử dụng Textract dạng asynchronous job, có thể cần thêm:

```text
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

Các quyền này sẽ được cấu hình chi tiết hơn trong phần **IAM Role and Policy** của workshop.

---

#### Logging trong quá trình OCR

Worker nên ghi log theo các mốc xử lý quan trọng:

```text
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[DOCUMENT_STATUS_UPDATED_TO_OCR_COMPLETED]
[SQS_MESSAGE_DELETED]
```

Nếu có lỗi:

```text
[TEXTRACT_OCR_FAILED]
[DOCUMENT_PROCESSING_FAILED]
```

Nếu đã tích hợp CloudWatch, các log này nên được gửi vào log group:

```text
/docmind/application
```

Không nên log toàn bộ nội dung tài liệu nếu tài liệu có thông tin nhạy cảm.

---

#### Các lỗi thường gặp

| Lỗi                            | Nguyên nhân                         | Cách xử lý                               |
| ------------------------------ | ----------------------------------- | ---------------------------------------- |
| `AccessDeniedException`        | Worker thiếu quyền gọi Textract     | Kiểm tra IAM Policy                      |
| `InvalidS3ObjectException`     | Textract không đọc được file S3     | Kiểm tra bucket, key, region và quyền S3 |
| `NoSuchKey`                    | Sai `s3Key` hoặc file không tồn tại | Kiểm tra object trong S3                 |
| `UnsupportedDocumentException` | File không đúng định dạng           | Dùng PDF, PNG, JPG hoặc JPEG             |
| `QueueDoesNotExist`            | Sai Queue URL hoặc region           | Kiểm tra `AWS_SQS_QUEUE_URL`             |
| OCR text rỗng                  | File mờ hoặc không có nội dung chữ  | Dùng file rõ hơn                         |

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được các kết quả sau:

* Worker có thể nhận message từ Amazon SQS.
* Worker lấy đúng tài liệu từ Amazon S3.
* Worker gọi Amazon Textract thành công.
* Nội dung OCR được trích xuất từ tài liệu.
* Kết quả OCR được lưu vào PostgreSQL thông qua Prisma.
* Trạng thái tài liệu được cập nhật sang `OCR_COMPLETED`.
* Message được xóa khỏi SQS sau khi xử lý thành công.
* Hệ thống sẵn sàng chuyển sang phần 5.6 để tích hợp Gemini/OpenAI AI Analysis.

---

#### Liên kết với bước tiếp theo

Sau khi OCR hoàn tất, hệ thống sẽ có được nội dung văn bản của tài liệu. Ở phần tiếp theo, nội dung này sẽ được gửi sang **AI Gateway** để phân tích bằng **Google Gemini API** hoặc **OpenAI ChatGPT API**.

Luồng xử lý tiếp theo:

```text
OCR Text
  |
  v
AI Gateway
  |
  v
Gemini API / OpenAI ChatGPT API
  |
  v
AI Analysis Result
  |
  v
PostgreSQL + Prisma
```

Như vậy, phần 5.5 đóng vai trò chuyển tài liệu từ file gốc thành văn bản, làm nền tảng cho các chức năng AI Analysis, AI Chat/RAG và Enterprise Sandbox của DocuMind AI.
