---

title : "Kiểm tra xử lý OCR"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.5.3. </b> "
-------------------------

#### Tổng quan

Sau khi đã cấu hình quyền Textract và tạo OCR Worker, bước tiếp theo là kiểm tra toàn bộ quá trình **OCR Processing** trong hệ thống **DocuMind AI**.

Ở bước này, chúng ta sẽ test luồng từ message trong **Amazon SQS**, worker lấy tài liệu từ **Amazon S3**, gọi **Amazon Textract**, lưu kết quả OCR vào **PostgreSQL + Prisma**, và cập nhật trạng thái tài liệu.

Đây là bước quan trọng để xác nhận hệ thống đã có thể chuyển tài liệu gốc thành văn bản. Văn bản OCR này sẽ là dữ liệu đầu vào cho **Gemini/OpenAI AI Gateway** ở phần tiếp theo.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Kiểm tra worker nhận message từ SQS.
* Kiểm tra worker lấy đúng file từ S3.
* Kiểm tra Amazon Textract OCR hoạt động.
* Kiểm tra text OCR được lưu vào PostgreSQL.
* Kiểm tra trạng thái tài liệu được cập nhật.
* Kiểm tra lỗi với file không hỗ trợ hoặc file không tồn tại.
* Chuẩn bị dữ liệu OCR cho phần AI Analysis.

---

#### Luồng kiểm tra OCR

Luồng OCR trong DocuMind AI:

```text
Amazon SQS
  |
  v
OCR Worker
  |
  v
Amazon S3
  |
  v
Amazon Textract
  |
  v
PostgreSQL + Prisma
  |
  v
Document status = OCR_COMPLETED
```

![test-ocr-processing](/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.3-test-ocr-processing/test-ocr-processing.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình log worker khi OCR thành công, kết quả OCR trong database hoặc trạng thái document chuyển sang `OCR_COMPLETED`.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.3-test-ocr-processing/test-ocr-processing.png`

---

#### Điều kiện trước khi kiểm tra

Trước khi test OCR, hãy đảm bảo:

* Đã tạo S3 bucket.
* Đã upload file PDF/PNG/JPG/JPEG lên S3.
* Đã tạo SQS processing queue.
* Đã gửi message chứa `documentId`, `s3Bucket`, `s3Key` vào SQS.
* Worker có quyền đọc S3.
* Worker có quyền gọi Amazon Textract.
* PostgreSQL và Prisma đang hoạt động.
* Worker đã được cấu hình đúng `.env`.

Ví dụ `.env`:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"

DATABASE_URL="postgresql://username:password@localhost:5432/docmind"
```

---

#### Bước 1: Chuẩn bị tài liệu test

Sử dụng file hợp lệ:

```text
invoice-demo.pdf
receipt-demo.jpg
contract-demo.pdf
```

Định dạng nên dùng:

```text
PDF
PNG
JPG
JPEG
```

Không dùng trực tiếp:

```text
DOCX
XLSX
PPTX
```

vì Amazon Textract có thể báo lỗi không hỗ trợ định dạng.

---

#### Bước 2: Upload tài liệu lên S3

Upload tài liệu bằng API đã test ở phần 5.3.3:

```text
POST http://localhost:3000/api/documents/upload
```

Kết quả mong đợi:

```json
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED"
  }
}
```

---

#### Bước 3: Kiểm tra message trong SQS

Vào AWS Console:

```text
Amazon SQS
```

Chọn queue:

```text
docmind-document-processing-queue
```

Kiểm tra `Available messages` có message mới hay không.

Message mong đợi:

```json
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

---

#### Bước 4: Khởi động OCR Worker

Mở terminal tại thư mục backend và chạy worker.

Ví dụ:

```bash
npm run worker
```

Hoặc nếu worker có script riêng:

```bash
npm run worker:document
```

Log mong đợi:

```text
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
```

---

#### Bước 5: Kiểm tra Textract OCR thành công

Khi Textract xử lý thành công, worker nên log:

```text
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[DOCUMENT_STATUS_UPDATED_TO_OCR_COMPLETED]
```

Nếu xử lý thành công, message nên được xóa khỏi SQS:

```text
[SQS_MESSAGE_DELETED]
```

---

#### Bước 6: Kiểm tra OCR Result trong PostgreSQL

Dùng Prisma Studio:

```bash
npx prisma studio
```

Mở bảng OCR result, ví dụ:

```text
OCRResult
```

Kiểm tra có dữ liệu mới:

```text
documentId: doc_001
provider: AWS_TEXTRACT
rawText: Nội dung OCR từ tài liệu
createdAt: 2026-06-10
```

Nếu OCR text rỗng, kiểm tra lại chất lượng file hoặc loại tài liệu.

---

#### Bước 7: Kiểm tra trạng thái document

Trong bảng `Document`, trạng thái sau OCR nên là:

```text
OCR_COMPLETED
```

Hoặc nếu hệ thống chuyển ngay sang AI Analysis, có thể là:

```text
AI_ANALYZING
```

Nếu lỗi, trạng thái nên là:

```text
FAILED
```

---

#### Bước 8: Test lỗi file không tồn tại

Gửi message sai `s3Key` vào SQS:

```json
{
  "documentId": "doc_error_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_error_001/not-found.pdf",
  "action": "PROCESS_DOCUMENT"
}
```

Worker nên log lỗi:

```text
[DOCUMENT_PROCESSING_FAILED]
InvalidS3ObjectException
```

hoặc:

```text
NoSuchKey
```

Tài liệu nên được cập nhật trạng thái `FAILED` nếu có record trong database.

---

#### Bước 9: Test lỗi file sai định dạng

Upload thử file không hỗ trợ như:

```text
test.docx
```

Kết quả mong đợi:

* Backend chặn file ngay từ bước upload.

Hoặc nếu file vẫn vào pipeline, Textract có thể báo:

```text
UnsupportedDocumentException
```

Cách xử lý tốt nhất là backend chặn định dạng không hỗ trợ trước khi upload hoặc trước khi gửi SQS message.

---

#### Bước 10: Kiểm tra CloudWatch Logs nếu đã tích hợp

Nếu worker đã gửi log lên CloudWatch, kiểm tra:

```text
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Tìm các log:

```text
TEXTRACT_OCR_STARTED
TEXTRACT_OCR_COMPLETED
TEXTRACT_OCR_FAILED
```

Không log toàn bộ nội dung tài liệu nếu tài liệu có thông tin nhạy cảm.

---

#### Các lỗi thường gặp

| Lỗi                            | Nguyên nhân                             | Cách xử lý                                   |
| ------------------------------ | --------------------------------------- | -------------------------------------------- |
| `AccessDeniedException`        | Thiếu quyền Textract                    | Thêm quyền Textract vào IAM Policy           |
| `InvalidS3ObjectException`     | Textract không đọc được object S3       | Kiểm tra bucket, key, region, quyền S3       |
| `NoSuchKey`                    | Sai `s3Key`                             | Kiểm tra object trong S3                     |
| `UnsupportedDocumentException` | File không hỗ trợ                       | Dùng PDF, PNG, JPG hoặc JPEG                 |
| Worker nhận message lặp        | Không delete message sau khi thành công | Gọi `DeleteMessage`                          |
| OCR text rỗng                  | File scan mờ hoặc không có text         | Dùng file rõ hơn                             |
| `QueueDoesNotExist`            | Sai Queue URL hoặc region               | Kiểm tra `AWS_SQS_QUEUE_URL` và `AWS_REGION` |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Upload tài liệu hợp lệ thành công.
* Message được gửi vào SQS.
* Worker nhận message từ SQS.
* Worker gọi Textract thành công.
* OCR text được lưu vào PostgreSQL.
* Document status cập nhật sang `OCR_COMPLETED`.
* Message được xóa khỏi SQS sau khi xử lý thành công.
* Lỗi file sai định dạng hoặc sai `s3Key` được xử lý rõ ràng.
* Hệ thống sẵn sàng chuyển sang phần 5.6 - Integrate AI Providers.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã hoàn thành phần OCR Processing. Hệ thống có thể lấy tài liệu từ S3, xử lý bằng Amazon Textract, lưu kết quả OCR vào PostgreSQL và chuẩn bị dữ liệu văn bản để gửi sang Gemini/OpenAI cho AI Analysis ở phần tiếp theo.
