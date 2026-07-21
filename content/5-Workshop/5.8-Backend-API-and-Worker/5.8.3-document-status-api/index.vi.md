---

title : "Xây dựng Document Status API"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.8.3. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng **Document Status API** cho hệ thống **DocuMind AI**.

Sau khi người dùng upload tài liệu, quá trình OCR và AI Analysis được xử lý bất đồng bộ bởi worker. Vì vậy, frontend cần một API để kiểm tra trạng thái xử lý của tài liệu theo thời gian thực hoặc theo cơ chế polling.

Document Status API giúp frontend biết tài liệu đang ở trạng thái nào: đã upload, đang chờ xử lý, đang OCR, đang phân tích AI, đã hoàn thành hoặc thất bại.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo API lấy trạng thái xử lý tài liệu.
* Trả về trạng thái hiện tại của document.
* Trả về thông tin OCR result nếu đã có.
* Trả về AI analysis nếu đã hoàn thành.
* Hỗ trợ frontend polling hoặc dashboard refresh.
* Giúp người dùng theo dõi tiến trình xử lý tài liệu.
* Giúp Admin kiểm tra tài liệu lỗi.

---

#### Luồng Document Status API

Luồng kiểm tra trạng thái:

```text id="fdr36l"
React Frontend Dashboard
  |
  v
GET /api/documents/:id/status
  |
  v
Backend API
  |
  v
PostgreSQL + Prisma
  |
  v
Return document status + progress
```

![document-status-api](/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.3-document-status-api/document-status-api.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình Postman hoặc Dashboard hiển thị trạng thái tài liệu như `QUEUED`, `PROCESSING`, `AI_ANALYZING` hoặc `COMPLETED`.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.3-document-status-api/document-status-api.png`

---

#### Endpoint đề xuất

Lấy trạng thái tài liệu:

```text id="rpwha8"
GET /api/documents/:id/status
```

Lấy chi tiết tài liệu:

```text id="8g65mo"
GET /api/documents/:id
```

Lấy danh sách tài liệu của user:

```text id="0gi75b"
GET /api/documents
```

Nếu là Admin, có thể có API:

```text id="vu32mo"
GET /api/admin/documents/:id/status
```

---

#### Response trạng thái cơ bản

Ví dụ khi tài liệu đang chờ xử lý:

```json id="l8etff"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "QUEUED",
    "progress": 25,
    "message": "Document is waiting for OCR processing."
  }
}
```

Ví dụ khi đang OCR:

```json id="72pwpz"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "PROCESSING",
    "progress": 45,
    "message": "OCR processing is running."
  }
}
```

Ví dụ khi đang phân tích AI:

```json id="vvqkz4"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "AI_ANALYZING",
    "progress": 75,
    "message": "AI analysis is running."
  }
}
```

Ví dụ khi hoàn thành:

```json id="zy0x3y"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "COMPLETED",
    "progress": 100,
    "message": "Document processing completed.",
    "ocrResult": {
      "provider": "AWS_TEXTRACT",
      "rawTextPreview": "Invoice No: INV-2026-001..."
    },
    "aiAnalysis": {
      "provider": "openai",
      "model": "gpt-4.1-mini",
      "documentType": "Invoice",
      "summary": "This document is an invoice from AWS.",
      "confidence": 0.9
    }
  }
}
```

Ví dụ khi lỗi:

```json id="0pkmfh"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "fileName": "invoice-demo.pdf",
    "status": "FAILED",
    "progress": 0,
    "message": "Document processing failed.",
    "error": "Unsupported document format."
  }
}
```

---

#### Mapping trạng thái sang progress

Frontend có thể dùng progress để hiển thị thanh tiến trình.

| Status          | Progress | Message             |
| --------------- | -------: | ------------------- |
| `PENDING`       |        5 | Metadata created    |
| `UPLOADED`      |       15 | File uploaded to S3 |
| `QUEUED`        |       25 | Waiting for worker  |
| `PROCESSING`    |       45 | OCR processing      |
| `OCR_COMPLETED` |       60 | OCR completed       |
| `AI_ANALYZING`  |       75 | AI analysis running |
| `COMPLETED`     |      100 | Completed           |
| `FAILED`        |        0 | Processing failed   |

---

#### Kiểm tra quyền truy cập

Document Status API cần kiểm tra quyền người dùng:

* User chỉ được xem tài liệu của chính họ.
* Admin có thể xem tất cả tài liệu.
* Nếu user không sở hữu document, trả về `403 Forbidden`.
* Nếu document không tồn tại, trả về `404 Not Found`.

Ví dụ response forbidden:

```json id="9hrl2d"
{
  "success": false,
  "message": "You do not have permission to access this document."
}
```

---

#### Ví dụ logic backend

```ts id="7fmj2z"
export async function getDocumentStatus(documentId: string, user: any) {
  const document = await prisma.document.findUnique({
    where: { id: documentId },
    include: {
      ocrResult: true,
      aiAnalysis: true
    }
  });

  if (!document) {
    throw new Error("Document not found");
  }

  if (user.role !== "ADMIN" && document.userId !== user.id) {
    throw new Error("Forbidden");
  }

  return {
    documentId: document.id,
    fileName: document.fileName,
    status: document.status,
    progress: mapStatusToProgress(document.status),
    ocrResult: document.ocrResult,
    aiAnalysis: document.aiAnalysis
  };
}
```

---

#### Hàm map progress

```ts id="aobnz2"
function mapStatusToProgress(status: string) {
  const progressMap: Record<string, number> = {
    PENDING: 5,
    UPLOADED: 15,
    QUEUED: 25,
    PROCESSING: 45,
    OCR_COMPLETED: 60,
    AI_ANALYZING: 75,
    COMPLETED: 100,
    FAILED: 0
  };

  return progressMap[status] ?? 0;
}
```

---

#### Test bằng Postman

Tạo request:

```text id="5gs0q7"
GET http://localhost:3000/api/documents/doc_001/status
```

Header nếu cần login:

```text id="kz2axv"
Authorization: Bearer your_jwt_token
```

Kết quả mong đợi:

```json id="9qs7k8"
{
  "success": true,
  "data": {
    "documentId": "doc_001",
    "status": "COMPLETED",
    "progress": 100
  }
}
```

---

#### Frontend polling

Frontend có thể gọi API này mỗi vài giây khi tài liệu chưa hoàn thành.

Ví dụ logic:

```text id="mx13jn"
Nếu status chưa phải COMPLETED hoặc FAILED:
  gọi lại API sau 3-5 giây
Nếu COMPLETED:
  hiển thị AI Analysis
Nếu FAILED:
  hiển thị lỗi
```

Không nên polling quá nhanh để tránh tạo quá nhiều request.

Khuyến nghị:

```text id="rq73uj"
Polling interval: 3-5 seconds
```

---

#### Logging cần có

Backend nên log:

```text id="d1ci6i"
[DOCUMENT_STATUS_REQUESTED]
[DOCUMENT_STATUS_RETURNED]
[DOCUMENT_NOT_FOUND]
[DOCUMENT_ACCESS_FORBIDDEN]
```

Không log thông tin nhạy cảm hoặc toàn bộ nội dung tài liệu.

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* API lấy trạng thái tài liệu hoạt động.
* API trả về đúng status.
* API trả về progress phù hợp.
* User chỉ xem được tài liệu của chính họ.
* Admin có thể xem tài liệu của tất cả người dùng.
* Frontend có thể polling trạng thái.
* Khi document `COMPLETED`, response có OCR/AI summary.
* Khi document `FAILED`, response có message lỗi rõ ràng.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có Document Status API để frontend theo dõi tiến trình xử lý tài liệu. Người dùng có thể biết tài liệu đang ở bước nào trong pipeline, còn Admin có thể theo dõi và xử lý các tài liệu bị lỗi.
