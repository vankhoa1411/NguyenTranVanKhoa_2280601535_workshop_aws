---

title : "Xử lý SQS Worker"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.8.2. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng và kiểm tra **SQS Worker Processing** cho hệ thống **DocuMind AI**.

SQS Worker là tiến trình nền có nhiệm vụ nhận message từ **Amazon SQS**, đọc thông tin tài liệu, gọi **Amazon Textract** để OCR, sau đó gửi kết quả OCR sang **AI Gateway** để phân tích bằng Gemini hoặc OpenAI. Kết quả cuối cùng sẽ được lưu vào **PostgreSQL thông qua Prisma**.

Worker giúp backend API không phải xử lý OCR và AI trực tiếp trong request upload, từ đó cải thiện hiệu năng và trải nghiệm người dùng.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo worker nhận message từ Amazon SQS.
* Parse message chứa `documentId`, `userId`, `s3Bucket`, `s3Key`.
* Cập nhật trạng thái tài liệu theo từng giai đoạn.
* Gọi Amazon Textract để OCR.
* Gọi AI Gateway để phân tích OCR text.
* Lưu OCRResult và AIAnalysis vào PostgreSQL.
* Xóa message khỏi SQS khi xử lý thành công.
* Xử lý lỗi và retry/DLQ khi thất bại.

---

#### Luồng SQS Worker Processing

Luồng worker trong DocuMind AI:

```text id="t52jxz"
Amazon SQS
  |
  v
SQS Worker
  |
  v
Update Document status = PROCESSING
  |
  v
Amazon Textract OCR
  |
  v
Save OCRResult
  |
  v
AI Gateway
  |
  v
Gemini / OpenAI
  |
  v
Save AIAnalysis
  |
  v
Update Document status = COMPLETED
  |
  v
Delete SQS message
```

![sqs-worker-processing](/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.2-sqs-worker-processing/sqs-worker-processing.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình log worker khi nhận message từ SQS, xử lý OCR, gọi AI Gateway và cập nhật document status thành `COMPLETED`.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.2-sqs-worker-processing/sqs-worker-processing.png`

---

#### Cấu trúc worker đề xuất

Có thể tổ chức source code:

```text id="o281pn"
src/
├── workers/
│   └── document.worker.ts
├── services/
│   ├── sqs.service.ts
│   ├── textract.service.ts
│   ├── document.service.ts
│   └── ai/
│       └── ai.gateway.ts
└── prisma/
    └── client.ts
```

---

#### Message đầu vào

Worker nhận message dạng:

```json id="krmh93"
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Worker cần validate các field bắt buộc:

```text id="20zhhb"
documentId
userId
s3Bucket
s3Key
action
```

Nếu thiếu field quan trọng, worker nên log lỗi và cập nhật trạng thái `FAILED` nếu có thể.

---

#### Logic xử lý worker

Pseudo flow:

```text id="3e0cpi"
Start worker
  |
  v
Poll message from SQS
  |
  v
Parse message JSON
  |
  v
Validate payload
  |
  v
Update document status = PROCESSING
  |
  v
Call Textract OCR
  |
  v
Save OCRResult
  |
  v
Update document status = OCR_COMPLETED
  |
  v
Update document status = AI_ANALYZING
  |
  v
Call AI Gateway
  |
  v
Save AIAnalysis
  |
  v
Update document status = COMPLETED
  |
  v
Delete message from SQS
```

Nếu lỗi:

```text id="6h4nw7"
Catch error
  |
  v
Log error
  |
  v
Update document status = FAILED
  |
  v
Do not delete message if retry is needed
```

---

#### Ví dụ worker rút gọn

```ts id="1a1txt"
async function processMessage(message: any) {
  const payload = JSON.parse(message.Body);

  const { documentId, userId, s3Bucket, s3Key } = payload;

  if (!documentId || !userId || !s3Bucket || !s3Key) {
    throw new Error("Invalid SQS message payload");
  }

  await prisma.document.update({
    where: { id: documentId },
    data: { status: "PROCESSING" }
  });

  const ocrText = await textractService.extractTextFromS3(s3Bucket, s3Key);

  await prisma.oCRResult.create({
    data: {
      documentId,
      provider: "AWS_TEXTRACT",
      rawText: ocrText
    }
  });

  await prisma.document.update({
    where: { id: documentId },
    data: { status: "AI_ANALYZING" }
  });

  const aiResult = await analyzeDocumentWithGateway(ocrText);

  await prisma.aIAnalysis.create({
    data: {
      documentId,
      provider: aiResult.provider,
      model: aiResult.model,
      documentType: aiResult.documentType,
      summary: aiResult.summary,
      entities: aiResult.entities,
      keywords: aiResult.keywords,
      importantInformation: aiResult.importantInformation,
      confidence: aiResult.confidence
    }
  });

  await prisma.document.update({
    where: { id: documentId },
    data: { status: "COMPLETED" }
  });

  await sqsService.deleteMessage(message.ReceiptHandle);
}
```

Tên model Prisma như `oCRResult` hoặc `aIAnalysis` có thể khác tùy schema. Bạn cần chỉnh theo Prisma Client thực tế của dự án.

---

#### Retry và Dead Letter Queue

Khi worker xử lý lỗi tạm thời, ví dụ AI provider timeout hoặc Textract lỗi, có thể để SQS retry bằng cách không xóa message khỏi queue.

Nếu message lỗi quá số lần retry, SQS sẽ chuyển sang DLQ.

Ví dụ:

```text id="zdh503"
Worker failed
  |
  v
Do not DeleteMessage
  |
  v
SQS retry
  |
  v
Exceed maximum receives
  |
  v
Move to DLQ
```

Trong demo, nếu không muốn retry nhiều lần, có thể cập nhật document status thành `FAILED` và xóa message để tránh xử lý lặp. Tuy nhiên, với production, nên dùng DLQ để debug.

---

#### Trạng thái tài liệu

Worker nên cập nhật trạng thái theo pipeline:

```text id="eyyzzv"
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

Frontend Dashboard sẽ dựa vào trạng thái này để hiển thị tiến trình cho người dùng.

---

#### Logging cần có

Worker nên ghi log:

```text id="71tm9q"
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_SAVED]
[DOCUMENT_STATUS_COMPLETED]
[SQS_MESSAGE_DELETED]
```

Nếu lỗi:

```text id="h5b3gq"
[DOCUMENT_PROCESSING_FAILED]
[SQS_MESSAGE_RETRY_PENDING]
[SQS_MESSAGE_MOVED_TO_DLQ]
```

---

#### Biến môi trường liên quan

```env id="i3rjmx"
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_S3_BUCKET_NAME="documind-document-storage"

DATABASE_URL="postgresql://docmind:docmind_password@localhost:5432/docmind"

AI_PROVIDER="openai"
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

---

#### Cách chạy worker

Thêm script vào `package.json`:

```json id="3ri26b"
{
  "scripts": {
    "worker": "ts-node src/workers/document.worker.ts",
    "worker:dev": "tsx watch src/workers/document.worker.ts"
  }
}
```

Chạy worker:

```bash id="ljov8s"
npm run worker
```

Hoặc:

```bash id="qx5vih"
npm run worker:dev
```

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Worker có thể nhận message từ SQS.
* Worker đọc được `documentId`, `s3Bucket`, `s3Key`.
* Worker cập nhật trạng thái `PROCESSING`.
* Worker gọi Textract OCR.
* Worker lưu OCRResult.
* Worker gọi AI Gateway.
* Worker lưu AIAnalysis.
* Worker cập nhật trạng thái `COMPLETED`.
* Worker xóa message khỏi SQS khi xử lý thành công.
* Lỗi được cập nhật `FAILED` hoặc chuyển sang DLQ.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI có worker xử lý tài liệu hoàn chỉnh. Tài liệu sau khi upload được đưa vào queue, worker xử lý OCR và AI Analysis ở nền, kết quả được lưu vào PostgreSQL và sẵn sàng hiển thị trên Dashboard.
