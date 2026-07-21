---

title : "Tạo OCR Worker"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.5.2. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng **OCR Worker** cho hệ thống **DocuMind AI**.

OCR Worker là tiến trình nền có nhiệm vụ nhận message từ **Amazon SQS**, lấy tài liệu từ **Amazon S3**, gọi **Amazon Textract** để trích xuất văn bản, sau đó lưu kết quả OCR vào **PostgreSQL thông qua Prisma**.

Worker giúp tách phần xử lý tài liệu nặng ra khỏi Backend API. Nhờ đó, khi người dùng upload tài liệu, backend có thể phản hồi nhanh, còn quá trình OCR và AI Analysis sẽ được xử lý bất đồng bộ ở phía sau.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Hiểu vai trò của OCR Worker trong pipeline DocuMind AI.
* Tạo worker nhận message từ Amazon SQS.
* Parse message để lấy `documentId`, `userId`, `s3Bucket` và `s3Key`.
* Gọi Amazon Textract để OCR tài liệu.
* Lưu kết quả OCR vào PostgreSQL.
* Cập nhật trạng thái tài liệu trong database.
* Chuẩn bị dữ liệu OCR để gửi sang Gemini/OpenAI AI Gateway ở bước sau.

---

#### Vai trò của OCR Worker

Trong hệ thống DocuMind AI, OCR Worker thực hiện các công việc sau:

* Poll message từ Amazon SQS.
* Kiểm tra message có hợp lệ hay không.
* Cập nhật trạng thái tài liệu sang `PROCESSING`.
* Lấy tài liệu từ Amazon S3 thông qua `s3Bucket` và `s3Key`.
* Gọi Amazon Textract để OCR nội dung.
* Lưu OCR text vào bảng `OCRResult`.
* Cập nhật trạng thái tài liệu sang `OCR_COMPLETED`.
* Gửi tiếp nội dung OCR sang bước AI Analysis nếu hệ thống đã tích hợp.
* Ghi log xử lý vào CloudWatch hoặc console.

Luồng worker:

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
```

![ocr-worker](/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.2-create-ocr-worker/ocr-worker.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình log worker khi nhận message từ SQS và gọi Textract thành công, hoặc vẽ sơ đồ SQS → Worker → S3 → Textract → PostgreSQL.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.5-Integrate-Textract-OCR/5.5.2-create-ocr-worker/ocr-worker.png`

---

#### Message đầu vào từ SQS

Worker sẽ nhận message từ SQS có dạng:

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

Các trường quan trọng:

| Trường       | Mục đích                         |
| ------------ | -------------------------------- |
| `documentId` | Xác định tài liệu trong database |
| `userId`     | Xác định người sở hữu tài liệu   |
| `s3Bucket`   | Bucket chứa file gốc             |
| `s3Key`      | Đường dẫn object trong S3        |
| `action`     | Loại tác vụ cần xử lý            |

Nếu message thiếu `documentId` hoặc `s3Key`, worker nên log lỗi và không xử lý tiếp.

---

#### Cài đặt AWS SDK cần thiết

Nếu backend/worker dùng Node.js, có thể cài các package cần thiết:

```bash
npm install @aws-sdk/client-sqs @aws-sdk/client-textract @aws-sdk/client-s3
```

Nếu dùng Prisma:

```bash
npm install @prisma/client
```

---

#### Cấu trúc file đề xuất

Bạn có thể tổ chức worker như sau:

```text
src/
├── workers/
│   └── document.worker.ts
├── services/
│   ├── textract.service.ts
│   ├── sqs.service.ts
│   └── document.service.ts
├── prisma/
│   └── client.ts
└── config/
    └── aws.config.ts
```

Vai trò từng file:

| File                  | Vai trò                                      |
| --------------------- | -------------------------------------------- |
| `document.worker.ts`  | Worker chính nhận message và điều phối xử lý |
| `textract.service.ts` | Gọi Amazon Textract OCR                      |
| `sqs.service.ts`      | Nhận/xóa message từ SQS                      |
| `document.service.ts` | Cập nhật trạng thái document và lưu OCR      |
| `aws.config.ts`       | Cấu hình AWS clients                         |

---

#### Logic xử lý của OCR Worker

Pseudo flow:

```text
Start worker
  |
  v
Poll message từ SQS
  |
  v
Parse message JSON
  |
  v
Validate documentId + s3Key
  |
  v
Update document status = PROCESSING
  |
  v
Call Textract OCR
  |
  v
Save OCRResult to PostgreSQL
  |
  v
Update document status = OCR_COMPLETED
  |
  v
Delete message khỏi SQS
```

Nếu xảy ra lỗi:

```text
Error
  |
  v
Log error
  |
  v
Update document status = FAILED
  |
  v
Không delete message nếu muốn SQS retry
```

Trong demo, nếu muốn tránh retry liên tục, có thể delete message sau khi đã cập nhật `FAILED`. Tuy nhiên trong production, nên dùng retry và DLQ.

---

#### Ví dụ hàm gọi Textract

Ví dụ logic OCR bằng Textract:

```ts
import { TextractClient, DetectDocumentTextCommand } from "@aws-sdk/client-textract";

const textractClient = new TextractClient({
  region: process.env.AWS_REGION
});

export async function extractTextFromS3(s3Bucket: string, s3Key: string) {
  const command = new DetectDocumentTextCommand({
    Document: {
      S3Object: {
        Bucket: s3Bucket,
        Name: s3Key
      }
    }
  });

  const response = await textractClient.send(command);

  const lines = response.Blocks
    ?.filter((block) => block.BlockType === "LINE")
    .map((block) => block.Text)
    .filter(Boolean) || [];

  return lines.join("\n");
}
```

---

#### Ví dụ worker xử lý message

Ví dụ rút gọn:

```ts
async function processDocumentMessage(messageBody: string) {
  const payload = JSON.parse(messageBody);

  const { documentId, userId, s3Bucket, s3Key } = payload;

  if (!documentId || !s3Bucket || !s3Key) {
    throw new Error("Invalid SQS message payload");
  }

  await prisma.document.update({
    where: { id: documentId },
    data: { status: "PROCESSING" }
  });

  const extractedText = await extractTextFromS3(s3Bucket, s3Key);

  await prisma.oCRResult.create({
    data: {
      documentId,
      rawText: extractedText,
      provider: "AWS_TEXTRACT"
    }
  });

  await prisma.document.update({
    where: { id: documentId },
    data: { status: "OCR_COMPLETED" }
  });

  return extractedText;
}
```

Tên model Prisma có thể khác tùy schema của bạn, ví dụ `OCRResult`, `OcrResult` hoặc `oCRResult`. Hãy chỉnh lại theo schema thật của dự án.

---

#### Trạng thái tài liệu

Trong pipeline OCR, có thể dùng các trạng thái sau:

```text
QUEUED
PROCESSING
OCR_COMPLETED
FAILED
```

Sau khi tích hợp AI Analysis, có thể mở rộng thêm:

```text
AI_ANALYZING
COMPLETED
```

---

#### Lưu OCR Result vào database

Dữ liệu OCR nên lưu vào bảng riêng, ví dụ `OCRResult`:

```json
{
  "id": "ocr_001",
  "documentId": "doc_001",
  "rawText": "Extracted text from invoice...",
  "provider": "AWS_TEXTRACT",
  "createdAt": "2026-06-10T10:00:00.000Z"
}
```

Nếu tài liệu không có text, worker nên lưu trạng thái lỗi hoặc cảnh báo để người dùng biết tài liệu không thể OCR.

---

#### Logging trong worker

Worker nên ghi log theo các mốc:

```text
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[DOCUMENT_STATUS_UPDATED]
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

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Worker có thể poll message từ SQS.
* Worker parse được `documentId`, `s3Bucket`, `s3Key`.
* Worker gọi được Amazon Textract.
* OCR text được trích xuất thành công.
* OCR result được lưu vào PostgreSQL.
* Trạng thái document được cập nhật sang `OCR_COMPLETED`.
* Message được xóa khỏi SQS sau khi xử lý thành công.
* Worker log rõ ràng từng bước xử lý.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có OCR Worker để xử lý tài liệu nền. Tài liệu sau khi upload lên S3 có thể được lấy ra, OCR bằng Amazon Textract và lưu kết quả vào PostgreSQL, sẵn sàng cho bước phân tích AI bằng Gemini/OpenAI.
