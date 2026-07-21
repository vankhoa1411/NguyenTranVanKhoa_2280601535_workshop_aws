---

title : "Kiểm tra AI Analysis"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.6.4. </b> "
-------------------------

#### Tổng quan

Sau khi đã thiết lập Gemini API, OpenAI ChatGPT API và AI Gateway, bước tiếp theo là kiểm tra chức năng **AI Analysis** trong hệ thống **DocuMind AI**.

Ở bước này, chúng ta sẽ test việc gửi nội dung OCR text sang AI Gateway, chọn provider là Gemini hoặc OpenAI, nhận kết quả phân tích tài liệu, lưu kết quả vào **PostgreSQL + Prisma** và cập nhật trạng thái tài liệu.

Đây là bước xác nhận rằng hệ thống đã có thể chuyển từ tài liệu gốc sang OCR text, sau đó sang kết quả phân tích AI có cấu trúc.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Kiểm tra Gemini API hoạt động trong backend.
* Kiểm tra OpenAI ChatGPT API hoạt động trong backend.
* Kiểm tra AI Gateway chọn provider đúng.
* Kiểm tra fallback giữa OpenAI và Gemini.
* Kiểm tra AI response trả về đúng JSON.
* Kiểm tra AI Analysis được lưu vào PostgreSQL.
* Kiểm tra trạng thái tài liệu được cập nhật thành `COMPLETED`.

---

#### Luồng kiểm tra AI Analysis

Luồng kiểm tra trong DocuMind AI:

```text
OCR Text
  |
  v
AI Gateway
  |
  |----------------------|
  |                      |
  v                      v
OpenAI Service       Gemini Service
  |                      |
  |----------------------|
           |
           v
AI Analysis Result
           |
           v
PostgreSQL + Prisma
           |
           v
Document status = COMPLETED
```

![test-ai-analysis](/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.4-test-ai-analysis/test-ai-analysis.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình Postman khi test AI Analysis thành công, log backend hiển thị provider được chọn hoặc dữ liệu AIAnalysis trong PostgreSQL.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.4-test-ai-analysis/test-ai-analysis.png`

---

#### Điều kiện trước khi kiểm tra

Trước khi test AI Analysis, hãy đảm bảo:

* Đã có Gemini API Key.
* Đã có OpenAI API Key.
* Backend đã có Gemini Service.
* Backend đã có OpenAI Service.
* Backend đã có AI Gateway Service.
* Đã có OCR text trong database.
* PostgreSQL và Prisma hoạt động.
* File `.env` đã có đầy đủ biến AI provider.

Ví dụ `.env`:

```env
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

---

#### Bước 1: Chuẩn bị OCR text test

Bạn có thể dùng OCR text từ Amazon Textract hoặc nhập text mẫu.

Ví dụ OCR text:

```text
Invoice No: INV-2026-001
Vendor: AWS
Customer: Tuan Quang
Invoice Date: 2026-06-10
Total Amount: 120 USD
Payment Status: Unpaid
```

---

#### Bước 2: Test AI Analysis bằng OpenAI

Tạo request trong Postman:

```text
Method: POST
URL: http://localhost:3000/api/ai/analyze
```

Body dạng JSON:

```json
{
  "text": "Invoice No: INV-2026-001. Vendor: AWS. Customer: Tuan Quang. Total Amount: 120 USD.",
  "provider": "openai"
}
```

Response mong đợi:

```json
{
  "success": true,
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "result": {
    "documentType": "Invoice",
    "summary": "This document is an invoice from AWS for Tuan Quang.",
    "entities": [
      {
        "type": "invoiceNumber",
        "value": "INV-2026-001"
      },
      {
        "type": "vendor",
        "value": "AWS"
      },
      {
        "type": "customer",
        "value": "Tuan Quang"
      },
      {
        "type": "totalAmount",
        "value": "120 USD"
      }
    ],
    "keywords": ["invoice", "AWS", "payment"],
    "importantInformation": [
      "Invoice number: INV-2026-001",
      "Total amount: 120 USD"
    ],
    "confidence": 0.9
  }
}
```

---

#### Bước 3: Test AI Analysis bằng Gemini

Tạo request tương tự nhưng đổi provider:

```json
{
  "text": "Contract Agreement between Company A and Company B. Effective date: 2026-06-10. Term: 12 months.",
  "provider": "gemini"
}
```

Response mong đợi:

```json
{
  "success": true,
  "provider": "gemini",
  "model": "gemini-1.5-flash",
  "result": {
    "documentType": "Contract",
    "summary": "This document is a contract agreement between Company A and Company B.",
    "entities": [
      {
        "type": "effectiveDate",
        "value": "2026-06-10"
      },
      {
        "type": "term",
        "value": "12 months"
      }
    ],
    "keywords": ["contract", "agreement", "effective date"],
    "importantInformation": [
      "Effective date: 2026-06-10",
      "Term: 12 months"
    ],
    "confidence": 0.85
  }
}
```

---

#### Bước 4: Test phân tích bằng documentId

Nếu hệ thống đã lưu OCR text trong database, có thể test theo `documentId`.

Endpoint ví dụ:

```text
POST /api/documents/:id/analyze
```

Request:

```json
{
  "provider": "openai"
}
```

Backend sẽ:

1. Tìm document theo `id`.
2. Lấy OCR text từ bảng `OCRResult`.
3. Gửi OCR text sang AI Gateway.
4. Lưu kết quả vào `AIAnalysis`.
5. Cập nhật document status thành `COMPLETED`.

Response mong đợi:

```json
{
  "success": true,
  "message": "AI analysis completed successfully",
  "documentId": "doc_001",
  "provider": "openai",
  "status": "COMPLETED"
}
```

---

#### Bước 5: Kiểm tra dữ liệu trong PostgreSQL

Dùng Prisma Studio:

```bash
npx prisma studio
```

Kiểm tra bảng:

```text
AIAnalysis
```

Dữ liệu mong đợi:

```text
documentId: doc_001
provider: openai
model: gpt-4.1-mini
summary: This document is an invoice...
documentType: Invoice
confidence: 0.9
createdAt: 2026-06-10
```

Kiểm tra bảng:

```text
Document
```

Trạng thái mong đợi:

```text
COMPLETED
```

---

#### Bước 6: Kiểm tra fallback provider

Để test fallback, có thể tạm thời làm sai API key của provider chính.

Ví dụ:

```env
AI_PROVIDER="openai"
OPENAI_API_KEY="wrong-key"
```

Sau đó gọi lại API analyze.

Kết quả mong đợi:

* OpenAI bị lỗi.
* AI Gateway log provider chính thất bại.
* Hệ thống chuyển sang Gemini.
* Response trả về provider là `gemini`.

Log mong đợi:

```text
[AI_PROVIDER_SELECTED] openai
[AI_PROVIDER_PRIMARY_FAILED] openai
[AI_PROVIDER_FALLBACK_USED] gemini
[AI_ANALYSIS_COMPLETED]
```

---

#### Bước 7: Kiểm tra lỗi OCR text rỗng

Gửi request với text rỗng:

```json
{
  "text": "",
  "provider": "openai"
}
```

Response mong đợi:

```json
{
  "success": false,
  "message": "OCR text is empty. Cannot analyze document."
}
```

Không nên gọi Gemini/OpenAI nếu OCR text rỗng vì sẽ tốn chi phí và kết quả không có ý nghĩa.

---

#### Bước 8: Kiểm tra response không đúng JSON

Trong một số trường hợp AI trả về markdown hoặc text không đúng JSON. Hệ thống cần xử lý lỗi rõ ràng.

Response lỗi mong đợi:

```json
{
  "success": false,
  "message": "AI response is not valid JSON."
}
```

Cách giảm lỗi:

* Siết prompt yêu cầu JSON only.
* Dùng `response_format` với OpenAI nếu có.
* Clean markdown trước khi `JSON.parse`.
* Retry một lần nếu response không hợp lệ.

---

#### Bước 9: Kiểm tra log backend

Log mong đợi:

```text
[AI_ANALYSIS_STARTED]
[AI_PROVIDER_SELECTED]
[OPENAI_REQUEST_STARTED]
[OPENAI_RESPONSE_RECEIVED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_SAVED]
[DOCUMENT_STATUS_UPDATED_TO_COMPLETED]
```

Hoặc nếu dùng Gemini:

```text
[GEMINI_REQUEST_STARTED]
[GEMINI_RESPONSE_RECEIVED]
```

Không log API key hoặc toàn bộ nội dung tài liệu nhạy cảm.

---

#### Các lỗi thường gặp

| Lỗi                      | Nguyên nhân                | Cách xử lý                      |
| ------------------------ | -------------------------- | ------------------------------- |
| `401 Unauthorized`       | OpenAI API key sai         | Kiểm tra `OPENAI_API_KEY`       |
| `API key not valid`      | Gemini API key sai         | Kiểm tra `GEMINI_API_KEY`       |
| `429 Rate limit`         | Vượt quota                 | Retry hoặc fallback provider    |
| `JSON.parse error`       | AI trả về không đúng JSON  | Siết prompt, clean response     |
| `OCR text is empty`      | Chưa có OCR result         | Kiểm tra phần Textract OCR      |
| `Provider not supported` | Sai provider trong request | Chỉ dùng `openai` hoặc `gemini` |
| Timeout                  | AI phản hồi chậm           | Thêm timeout và fallback        |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Test OpenAI Analysis thành công.
* Test Gemini Analysis thành công.
* AI Gateway chọn đúng provider.
* Fallback hoạt động khi provider chính lỗi.
* AI trả về JSON đúng format.
* AIAnalysis được lưu vào PostgreSQL.
* Document status được cập nhật thành `COMPLETED`.
* Log backend thể hiện rõ provider, model và trạng thái xử lý.
* Hệ thống sẵn sàng kết nối kết quả AI lên Dashboard.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã hoàn thành phần AI Analysis cơ bản. Hệ thống có thể nhận OCR text, phân tích bằng OpenAI hoặc Gemini, fallback khi cần, lưu kết quả vào PostgreSQL và hiển thị kết quả cho người dùng ở các phần Dashboard, AI Chat/RAG và Enterprise Sandbox.
