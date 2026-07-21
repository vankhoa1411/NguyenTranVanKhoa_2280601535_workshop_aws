---

title : "Thiết lập OpenAI ChatGPT API"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.6.2. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ thiết lập **OpenAI ChatGPT API** để tích hợp vào hệ thống **DocuMind AI**.

OpenAI ChatGPT API là AI provider chính thứ hai của dự án, hoạt động song song với Google Gemini API. Sau khi Amazon Textract trích xuất nội dung văn bản từ tài liệu, backend có thể gửi OCR text đến OpenAI để tóm tắt, phân loại, trích xuất thông tin quan trọng, tạo JSON có cấu trúc và hỗ trợ AI Chat/RAG.

Việc tích hợp OpenAI giúp hệ thống có thêm lựa chọn AI mạnh mẽ, đồng thời tăng độ ổn định nhờ cơ chế fallback giữa OpenAI và Gemini.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo hoặc chuẩn bị OpenAI API Key.
* Thêm OpenAI API Key vào file `.env`.
* Cài đặt OpenAI SDK cho backend.
* Tạo OpenAI Service.
* Kiểm tra OpenAI có thể phân tích OCR text.
* Chuẩn bị OpenAI để tích hợp vào AI Gateway.

---

#### Vai trò của OpenAI trong DocuMind AI

Trong hệ thống **DocuMind AI**, OpenAI ChatGPT API được sử dụng để:

* Phân tích nội dung OCR từ tài liệu.
* Tóm tắt tài liệu.
* Phân loại tài liệu.
* Trích xuất thực thể và thông tin quan trọng.
* Hỗ trợ AI Chat/RAG với tài liệu.
* Hỗ trợ semantic search nếu dùng embedding.
* Làm provider chính hoặc fallback provider trong AI Gateway.

Luồng xử lý OpenAI trong pipeline:

```text id="8uejwk"
OCR Text từ Amazon Textract
  |
  v
OpenAI Service
  |
  v
OpenAI ChatGPT API
  |
  v
AI Analysis Result
  |
  v
PostgreSQL + Prisma
```

![setup-openai-api](/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.2-setup-openai-api/setup-openai-api.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình trang tạo OpenAI API Key hoặc vẽ sơ đồ OCR Text → OpenAI Service → OpenAI ChatGPT API → AI Analysis Result.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.2-setup-openai-api/setup-openai-api.png`

---

#### Bước 1: Chuẩn bị OpenAI API Key

Truy cập trang quản lý API key của OpenAI và tạo API key mới cho dự án.

Sau khi tạo, copy API key và lưu lại an toàn.

API key thường có dạng:

```text id="qkhzji"
sk-xxxxxxxxxxxxxxxxxxxxxxxx
```

> ⚠️ **Lưu ý bảo mật:**
> Không chia sẻ OpenAI API Key. Không đưa API key vào frontend. Không commit API key lên GitHub. Chỉ lưu trong `.env` local hoặc AWS Secrets Manager khi deploy production.

---

#### Bước 2: Cấu hình OpenAI trong file `.env`

Thêm các biến môi trường sau vào backend:

```env id="hq21nr"
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
```

Nếu muốn OpenAI là AI provider mặc định:

```env id="ea6p0j"
AI_PROVIDER="openai"
```

Nếu muốn Gemini là mặc định và OpenAI làm fallback, có thể dùng:

```env id="7xtu9q"
AI_PROVIDER="gemini"
```

---

#### Bước 3: Cài đặt OpenAI SDK

Trong thư mục backend, chạy:

```bash id="fbg9c9"
npm install openai
```

---

#### Bước 4: Tạo OpenAI Service

Tạo file:

```text id="l54hpm"
src/services/ai/openai.service.ts
```

Ví dụ service cơ bản:

```ts id="46t3xi"
import OpenAI from "openai";

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

export async function analyzeWithOpenAI(ocrText: string) {
  const response = await openai.chat.completions.create({
    model: process.env.OPENAI_MODEL || "gpt-4.1-mini",
    messages: [
      {
        role: "system",
        content: "You are DocuMind AI, a document intelligence assistant. Return JSON only."
      },
      {
        role: "user",
        content: `
Analyze the OCR text below.

Rules:
- Return JSON only.
- Do not use markdown.
- If a field is not found, return null.
- Do not invent information.

OCR Text:
${ocrText}

Return JSON format:
{
  "documentType": "",
  "summary": "",
  "entities": [],
  "keywords": [],
  "importantInformation": [],
  "confidence": 0
}
`
      }
    ],
    response_format: { type: "json_object" }
  });

  const content = response.choices[0]?.message?.content || "{}";
  return JSON.parse(content);
}
```

---

#### Bước 5: Chuẩn hóa kết quả trả về

OpenAI Service nên trả về cùng format với Gemini để AI Gateway dễ xử lý.

Ví dụ:

```json id="b9jl3t"
{
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "documentType": "Invoice",
  "summary": "This document is an invoice for cloud usage.",
  "entities": [
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

---

#### Bước 6: Tạo API test OpenAI

Tạo route test tạm thời:

```text id="2ues21"
POST /api/ai/test-openai
```

Request body:

```json id="efpaxv"
{
  "text": "Invoice No: INV-001. Vendor: AWS. Total: 120 USD."
}
```

Response mong đợi:

```json id="0d2dqz"
{
  "success": true,
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "result": {
    "documentType": "Invoice",
    "summary": "The document is an invoice from AWS.",
    "confidence": 0.9
  }
}
```

---

#### Bước 7: Test bằng Postman

Mở Postman và tạo request:

```text id="wihj7l"
Method: POST
URL: http://localhost:3000/api/ai/test-openai
```

Body dạng JSON:

```json id="gkmdac"
{
  "text": "Contract Agreement between Company A and Company B. Effective date: 2026-06-10."
}
```

Nếu API key đúng và model hoạt động, backend sẽ trả về kết quả phân tích dạng JSON.

---

#### Bước 8: Tích hợp embedding nếu dùng AI Chat/RAG

Nếu hệ thống có **AI Chat/RAG** hoặc **Semantic Search**, OpenAI có thể dùng embedding model để tạo vector.

Ví dụ biến môi trường:

```env id="f2h39u"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
```

Ví dụ hàm tạo embedding:

```ts id="0qbnb5"
export async function generateOpenAIEmbedding(text: string) {
  const response = await openai.embeddings.create({
    model: process.env.OPENAI_EMBEDDING_MODEL || "text-embedding-3-small",
    input: text
  });

  return response.data[0].embedding;
}
```

Embedding này có thể được dùng để:

* Lưu vector cho document chunks.
* Tìm đoạn tài liệu liên quan nhất với câu hỏi.
* Hỗ trợ AI Chat/RAG và Semantic Search.

---

#### Bước 9: Xử lý lỗi thường gặp

| Lỗi                  | Nguyên nhân                           | Cách xử lý                                    |
| -------------------- | ------------------------------------- | --------------------------------------------- |
| `401 Unauthorized`   | API key sai hoặc hết hiệu lực         | Tạo lại OpenAI API Key                        |
| `429 Rate limit`     | Vượt quota hoặc gọi quá nhiều request | Giảm request, retry hoặc fallback sang Gemini |
| `model_not_found`    | Sai tên model                         | Kiểm tra `OPENAI_MODEL`                       |
| `insufficient_quota` | Tài khoản hết quota/billing           | Kiểm tra billing OpenAI                       |
| `JSON.parse error`   | Response không đúng JSON              | Dùng `response_format` và siết prompt         |
| Timeout              | Request phản hồi chậm                 | Thêm timeout và fallback sang Gemini          |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã có OpenAI API Key.
* API key đã được thêm vào `.env`.
* Backend đã cài package `openai`.
* Đã tạo OpenAI Service.
* Endpoint test OpenAI hoạt động.
* OpenAI trả về kết quả JSON.
* Backend không expose API key ra frontend.
* OpenAI sẵn sàng tích hợp vào AI Gateway.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã tích hợp thành công OpenAI ChatGPT API ở mức backend service. Hệ thống có thể gửi OCR text sang OpenAI để phân tích tài liệu, đồng thời chuẩn bị kết hợp OpenAI với Gemini trong AI Gateway để tạo kiến trúc multi-model linh hoạt.
