---

title : "Tích hợp AI Providers"
date : 2026-06-10
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
-----------------------

#### Tích hợp Gemini và OpenAI AI Providers

Trong phần này, chúng ta sẽ tích hợp hai AI provider chính cho hệ thống **DocuMind AI**, bao gồm **Google Gemini API** và **OpenAI ChatGPT API**.

Sau khi tài liệu được xử lý OCR bằng **Amazon Textract**, hệ thống sẽ nhận được nội dung văn bản của tài liệu. Nội dung này sẽ được gửi đến **AI Gateway Service** để thực hiện các tác vụ phân tích như tóm tắt tài liệu, phân loại tài liệu, trích xuất thông tin quan trọng, tạo kết quả JSON có cấu trúc và hỗ trợ hỏi đáp tài liệu theo ngữ cảnh.

Việc sử dụng hai AI provider giúp hệ thống linh hoạt hơn. DocuMind AI có thể chọn Gemini hoặc OpenAI làm provider chính, đồng thời sử dụng provider còn lại làm phương án fallback khi provider chính gặp lỗi, timeout hoặc giới hạn quota.

---

#### Vai trò của AI Providers trong DocuMind AI

Trong dự án **DocuMind AI**, Gemini và OpenAI được sử dụng để:

* Phân tích nội dung OCR từ tài liệu.
* Tóm tắt tài liệu.
* Phân loại tài liệu như hóa đơn, hợp đồng, biểu mẫu hoặc báo cáo.
* Trích xuất thực thể như tên khách hàng, nhà cung cấp, ngày tháng, tổng tiền hoặc điều khoản quan trọng.
* Tạo dữ liệu JSON có cấu trúc.
* Hỗ trợ AI Chat/RAG cho phép người dùng đặt câu hỏi trên tài liệu.
* Hỗ trợ Semantic Search và Enterprise Sandbox.
* Tăng độ ổn định bằng cơ chế multi-provider và fallback.

---

#### Kiến trúc AI Provider

Luồng xử lý AI trong phần này:

```text
OCR Text từ Amazon Textract
  |
  v
AI Gateway Service
  |
  |--------------------------|
  |                          |
  v                          v
Google Gemini API        OpenAI ChatGPT API
  |                          |
  |--------------------------|
              |
              v
        AI Analysis Result
              |
              v
PostgreSQL + Prisma lưu kết quả
```

![ai-provider-flow](/images/5-Workshop/5.6-Integrate-AI-Providers/ai-provider-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa luồng OCR Text từ Amazon Textract đi vào AI Gateway, sau đó Gateway chọn Gemini hoặc OpenAI để phân tích tài liệu và lưu kết quả vào PostgreSQL + Prisma.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/ai-provider-flow.png`

---

#### Workflow xử lý AI Analysis

Quy trình AI Analysis trong DocuMind AI hoạt động như sau:

1. Tài liệu được upload lên Amazon S3.
2. Backend gửi job xử lý vào Amazon SQS.
3. Worker nhận message từ SQS.
4. Worker gọi Amazon Textract để OCR tài liệu.
5. OCR text được lưu vào PostgreSQL.
6. Worker hoặc backend gửi OCR text sang AI Gateway.
7. AI Gateway chọn provider theo cấu hình hoặc request của người dùng.
8. Gemini hoặc OpenAI phân tích nội dung tài liệu.
9. Kết quả AI Analysis được lưu vào PostgreSQL.
10. Trạng thái tài liệu được cập nhật thành `COMPLETED`.
11. Người dùng nhận thông báo và xem kết quả trên Dashboard.

Các trạng thái tài liệu trong phần này có thể gồm:

```text
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

---

#### Nội dung chi tiết

Trong phần **5.6 - Integrate AI Providers**, chúng ta sẽ thực hiện các bước sau:

* [Thiết lập Gemini API](5.6.1-setup-gemini-api/)
* [Thiết lập OpenAI ChatGPT API](5.6.2-setup-openai-api/)
* [Tạo AI Gateway](5.6.3-create-ai-gateway/)
* [Kiểm tra AI Analysis](5.6.4-test-ai-analysis/)

---

#### AI Analysis Result

Sau khi AI provider xử lý OCR text, hệ thống nên lưu kết quả vào bảng `AIAnalysis`.

Ví dụ kết quả AI:

```json
{
  "documentId": "doc_001",
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "documentType": "Invoice",
  "summary": "This document is an invoice for cloud service usage.",
  "entities": [
    {
      "type": "vendor",
      "value": "AWS"
    },
    {
      "type": "totalAmount",
      "value": "120 USD"
    }
  ],
  "keywords": ["invoice", "payment", "cloud"],
  "importantInformation": [
    "Invoice number: INV-001",
    "Total amount: 120 USD"
  ],
  "confidence": 0.9
}
```

Kết quả này sẽ được hiển thị ở Dashboard, Document Detail, Enterprise Sandbox hoặc dùng làm dữ liệu cho AI Chat/RAG.

---

#### Biến môi trường liên quan

Backend cần các biến môi trường cho Gemini và OpenAI:

```env
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

Nếu muốn dùng Gemini làm provider mặc định:

```env
AI_PROVIDER="gemini"
```

Nếu muốn dùng OpenAI làm provider mặc định:

```env
AI_PROVIDER="openai"
```

---

#### Nguyên tắc bảo mật API Key

Khi tích hợp AI provider, cần đảm bảo:

* Không đưa Gemini/OpenAI API key vào frontend.
* Không commit file `.env` thật lên GitHub.
* Chỉ backend được gọi trực tiếp Gemini/OpenAI API.
* Production nên lưu API key trong AWS Secrets Manager.
* Không log API key trong console hoặc CloudWatch.
* Không log toàn bộ nội dung tài liệu nếu tài liệu chứa thông tin nhạy cảm.

---

#### Logging trong quá trình AI Analysis

Hệ thống nên ghi log theo các mốc:

```text
[AI_ANALYSIS_STARTED]
[AI_PROVIDER_SELECTED]
[GEMINI_REQUEST_STARTED]
[GEMINI_RESPONSE_RECEIVED]
[OPENAI_REQUEST_STARTED]
[OPENAI_RESPONSE_RECEIVED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_SAVED]
[AI_PROVIDER_FALLBACK_USED]
[AI_ANALYSIS_FAILED]
```

Nếu đã tích hợp CloudWatch, các log này nên được gửi vào log group:

```text
/docmind/application
```

---

#### Các lỗi thường gặp

| Lỗi                     | Nguyên nhân                             | Cách xử lý                                        |
| ----------------------- | --------------------------------------- | ------------------------------------------------- |
| `API key not valid`     | Sai Gemini/OpenAI API key               | Kiểm tra lại `.env` hoặc Secrets Manager          |
| `401 Unauthorized`      | OpenAI API key sai                      | Tạo lại OpenAI key                                |
| `403 Permission denied` | Gemini key chưa đúng project/quyền      | Kiểm tra Google AI Studio                         |
| `429 Rate limit`        | Vượt quota                              | Retry hoặc fallback provider                      |
| `JSON.parse error`      | AI trả về text không đúng JSON          | Siết prompt, clean JSON hoặc dùng response format |
| Timeout                 | Provider phản hồi chậm                  | Thêm timeout và fallback                          |
| Empty OCR text          | Textract không trích xuất được nội dung | Kiểm tra OCR result trước khi gọi AI              |

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được các kết quả sau:

* Gemini API đã được cấu hình và test thành công.
* OpenAI ChatGPT API đã được cấu hình và test thành công.
* AI Gateway có thể chọn provider theo cấu hình.
* Hệ thống có thể fallback giữa Gemini và OpenAI.
* OCR text được phân tích thành kết quả AI có cấu trúc.
* Kết quả AI Analysis được lưu vào PostgreSQL thông qua Prisma.
* Trạng thái tài liệu được cập nhật thành `COMPLETED`.
* Dashboard có thể hiển thị kết quả phân tích tài liệu.

---

#### Liên kết với bước tiếp theo

Sau khi tích hợp AI Providers, hệ thống đã có thể xử lý tài liệu từ file gốc đến kết quả phân tích AI. Ở phần tiếp theo, chúng ta sẽ cấu hình **PostgreSQL + Prisma Database** để đảm bảo các bảng dữ liệu như `User`, `Document`, `OCRResult`, `AIAnalysis`, `ChatHistory`, `Notification` và `AuditLog` được thiết kế đúng cho toàn bộ hệ thống.

Luồng dữ liệu sau phần này:

```text
S3
  |
  v
SQS
  |
  v
Textract OCR
  |
  v
Gemini / OpenAI
  |
  v
PostgreSQL + Prisma
  |
  v
Dashboard / AI Chat / Notification
```

Như vậy, phần 5.6 đóng vai trò xây dựng lớp trí tuệ nhân tạo cốt lõi cho DocuMind AI.
