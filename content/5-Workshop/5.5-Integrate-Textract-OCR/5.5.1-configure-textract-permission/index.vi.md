---

title : "Cấu hình quyền Amazon Textract"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.5.1. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ cấu hình quyền cần thiết để backend hoặc worker của **DocuMind AI** có thể gọi **Amazon Textract** nhằm trích xuất văn bản từ tài liệu.

Amazon Textract là dịch vụ OCR của AWS, cho phép phân tích tài liệu PDF hoặc hình ảnh và trả về nội dung chữ viết được trích xuất. Trong pipeline của DocuMind AI, Textract được sử dụng sau khi tài liệu đã được upload lên **Amazon S3** và message xử lý đã được gửi vào **Amazon SQS**.

Worker sẽ nhận message từ SQS, lấy thông tin `s3Bucket` và `s3Key`, sau đó gọi Textract để OCR tài liệu.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Hiểu vai trò của Amazon Textract trong DocuMind AI.
* Biết các quyền IAM cần thiết để gọi Textract.
* Cấu hình quyền cho backend/worker gọi Textract.
* Kiểm tra region hỗ trợ Amazon Textract.
* Chuẩn bị môi trường cho OCR Worker ở bước tiếp theo.

---

#### Vai trò của Textract trong DocuMind AI

Trong hệ thống **DocuMind AI**, Amazon Textract được dùng để:

* Trích xuất văn bản từ tài liệu PDF hoặc hình ảnh.
* Chuyển tài liệu phi cấu trúc thành nội dung text.
* Cung cấp dữ liệu đầu vào cho Gemini/OpenAI AI Gateway.
* Hỗ trợ các chức năng tóm tắt, phân loại, trích xuất thực thể và AI Chat/RAG.
* Giảm thao tác đọc và nhập liệu thủ công.

Luồng xử lý trong bước này:

```text
Amazon SQS
  |
  v
Worker
  |
  v
Amazon S3 lấy tài liệu
  |
  v
Amazon Textract OCR
  |
  v
PostgreSQL + Prisma lưu OCR Result
```

![textract-permission](/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.1-configure-textract-permission/textract-permission.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình IAM Policy hoặc IAM Role có quyền gọi Amazon Textract, hoặc vẽ sơ đồ Worker → S3 → Textract → PostgreSQL.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.1-configure-textract-permission/textract-permission.png`

---

#### Định dạng tài liệu được hỗ trợ

Trong workshop này, nên sử dụng các định dạng phù hợp với Amazon Textract:

```text
PDF
PNG
JPG
JPEG
```

Không nên upload trực tiếp các định dạng:

```text
DOCX
XLSX
PPTX
TXT
```

Nếu cần xử lý DOCX, XLSX hoặc PPTX, nên chuyển đổi các file này sang PDF trước khi đưa vào pipeline OCR.

---

#### Kiểm tra region Amazon Textract

Trước khi cấu hình, hãy đảm bảo bạn đang dùng region có hỗ trợ Amazon Textract.

Ví dụ region thường dùng:

```env
AWS_REGION="ap-southeast-1"
```

Nếu Textract không hoạt động ở region bạn chọn, có thể gặp lỗi như:

```text
UnknownEndpoint
```

hoặc:

```text
Textract is not available in this region
```

Do đó, các service trong workshop nên được đặt cùng region nếu có thể:

* Amazon S3
* Amazon SQS
* Amazon Textract
* CloudWatch
* Secrets Manager
* Backend/Worker

---

#### Quyền IAM cần thiết cho Textract

Worker cần quyền gọi Amazon Textract. Các quyền cơ bản gồm:

```text
textract:DetectDocumentText
textract:AnalyzeDocument
```

Nếu hệ thống xử lý tài liệu bất đồng bộ bằng Textract job, có thể cần thêm:

```text
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

Trong workshop này, nếu xử lý file nhỏ hoặc demo đơn giản, có thể dùng `DetectDocumentText` hoặc `AnalyzeDocument`.

---

#### Quyền IAM cần thiết với S3

Vì Textract cần đọc file từ S3, worker cũng cần quyền truy cập object trong S3 bucket:

```text
s3:GetObject
s3:ListBucket
```

Nếu backend cũng upload file lên S3, backend cần thêm:

```text
s3:PutObject
```

---

#### IAM Policy mẫu cho Textract OCR Worker

Policy tham khảo:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindTextractAccess",
      "Effect": "Allow",
      "Action": [
        "textract:DetectDocumentText",
        "textract:AnalyzeDocument",
        "textract:StartDocumentTextDetection",
        "textract:GetDocumentTextDetection",
        "textract:StartDocumentAnalysis",
        "textract:GetDocumentAnalysis"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DocuMindS3ReadAccessForTextract",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::documind-document-storage",
        "arn:aws:s3:::documind-document-storage/*"
      ]
    }
  ]
}
```

> ⚠️ **Lưu ý:**
> Nếu bucket của bạn có tên khác, hãy thay `documind-document-storage` bằng tên bucket thật của bạn.

---

#### Cấu hình IAM Role cho backend/worker

Trong production hoặc khi deploy lên EC2, nên dùng **IAM Role** thay vì hardcode AWS credentials.

Mô hình khuyến nghị:

```text
EC2 / Backend Server
  |
  v
IAM Role: DocMind-EC2-Role
  |
  v
Policy: S3 + SQS + Textract + CloudWatch + Secrets Manager
```

Không nên dùng trong production:

```env
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

AWS SDK sẽ tự lấy quyền từ IAM Role nếu backend chạy trên EC2 có gắn role phù hợp.

---

#### Biến môi trường liên quan

Backend/worker cần các biến môi trường:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
```

Nếu có cấu hình riêng cho Textract:

```env
AWS_TEXTRACT_REGION="ap-southeast-1"
```

Nếu không có biến riêng, có thể dùng chung `AWS_REGION`.

---

#### Kiểm tra quyền bằng AWS CLI

Bạn có thể kiểm tra danh tính AWS hiện tại:

```bash
aws sts get-caller-identity
```

Nếu chạy trên EC2 bằng IAM Role, kết quả nên có dạng:

```text
arn:aws:sts::123456789012:assumed-role/DocMind-EC2-Role/...
```

Có thể kiểm tra nhanh quyền S3:

```bash
aws s3 ls s3://documind-document-storage
```

Nếu lệnh này chạy được, worker có quyền đọc bucket.

---

#### Các lỗi thường gặp

| Lỗi                            | Nguyên nhân                         | Cách xử lý                                                               |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------------ |
| `AccessDeniedException`        | Thiếu quyền Textract                | Thêm quyền `textract:DetectDocumentText` hoặc `textract:AnalyzeDocument` |
| `AccessDenied` với S3          | Worker không có quyền đọc file      | Thêm `s3:GetObject` cho bucket                                           |
| `NoSuchBucket`                 | Sai tên bucket                      | Kiểm tra `AWS_S3_BUCKET_NAME`                                            |
| `InvalidS3ObjectException`     | Sai `s3Key` hoặc file không tồn tại | Kiểm tra object trong S3                                                 |
| `UnsupportedDocumentException` | File không đúng định dạng           | Dùng PDF, PNG, JPG hoặc JPEG                                             |
| `UnknownEndpoint`              | Sai region hoặc region không hỗ trợ | Kiểm tra `AWS_REGION`                                                    |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Worker/backend có quyền gọi Amazon Textract.
* Worker/backend có quyền đọc file từ S3.
* Region AWS đã được cấu hình đúng.
* Bucket name và `s3Key` được kiểm tra chính xác.
* Không hardcode AWS access key trong production.
* Hệ thống sẵn sàng tạo OCR Worker ở bước 5.5.2.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có đủ quyền cần thiết để worker lấy tài liệu từ Amazon S3 và gọi Amazon Textract OCR. Đây là nền tảng để chuyển tài liệu gốc thành văn bản, phục vụ cho bước phân tích AI bằng Gemini/OpenAI.
