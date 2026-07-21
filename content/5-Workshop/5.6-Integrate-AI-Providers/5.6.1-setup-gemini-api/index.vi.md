---

title : "Thiết lập Gemini API"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.6.1. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ thiết lập **Google Gemini API** để tích hợp vào hệ thống **DocuMind AI**.

Gemini API là một trong hai AI provider chính của dự án, được sử dụng để phân tích nội dung tài liệu sau khi OCR bằng Amazon Textract. Sau khi worker trích xuất được văn bản từ tài liệu, nội dung OCR sẽ được gửi đến Gemini để thực hiện các tác vụ như tóm tắt tài liệu, phân loại tài liệu, trích xuất thông tin quan trọng, tạo JSON có cấu trúc và hỗ trợ AI Chat/RAG.

Việc tích hợp Gemini giúp hệ thống có thêm một AI provider linh hoạt, có thể dùng làm provider chính hoặc provider dự phòng khi OpenAI gặp lỗi, timeout hoặc giới hạn quota.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được Gemini API Key.
* Thêm Gemini API Key vào file `.env`.
* Cài đặt SDK cần thiết cho backend.
* Tạo Gemini Service trong backend.
* Kiểm tra Gemini có thể phân tích văn bản OCR.
* Chuẩn bị Gemini để tích hợp vào AI Gateway ở các bước sau.

---

#### Vai trò của Gemini API trong DocuMind AI

Trong hệ thống **DocuMind AI**, Gemini API được sử dụng để:

* Tóm tắt nội dung tài liệu.
* Phân loại tài liệu như hóa đơn, hợp đồng, biểu mẫu hoặc báo cáo.
* Trích xuất thông tin quan trọng từ OCR text.
* Tạo kết quả JSON có cấu trúc.
* Hỗ trợ AI Chat với tài liệu.
* Làm provider chính hoặc fallback provider trong AI Gateway.
* Tăng tính linh hoạt khi hệ thống có nhiều AI model.

Luồng xử lý Gemini trong pipeline:

```text id="9wlf4b"
OCR Text từ Amazon Textract
  |
  v
Gemini Service
  |
  v
Gemini API
  |
  v
AI Analysis Result
  |
  v
PostgreSQL + Prisma
```

![setup-gemini-api](/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.1-setup-gemini-api/setup-gemini-api.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình trang tạo API Key trong Google AI Studio hoặc vẽ sơ đồ OCR Text → Gemini Service → Gemini API → AI Analysis Result.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.1-setup-gemini-api/setup-gemini-api.png`

---

#### Bước 1: Truy cập Google AI Studio

Mở trình duyệt và truy cập:

```text id="3erwdv"
https://aistudio.google.com/app/apikey
```

Đăng nhập bằng tài khoản Google của bạn.

Sau đó chọn:

```text id="fk584w"
Create API Key
```

Nếu bạn có nhiều Google Cloud Project, hãy chọn đúng project dùng cho DocuMind AI.

---

#### Bước 2: Tạo Gemini API Key

Sau khi chọn project, nhấn tạo key.

Bạn sẽ nhận được API key dạng:

```text id="y7o3ci"
AIzaSyxxxxxxxxxxxxxxxxxxxxxxxx
```

Copy API key này để thêm vào file môi trường backend.

> ⚠️ **Lưu ý bảo mật:**
> Không chia sẻ Gemini API Key cho người khác. Không commit API key lên GitHub. Chỉ lưu key trong `.env` local hoặc AWS Secrets Manager khi deploy production.

---

#### Bước 3: Cấu hình Gemini trong file `.env`

Thêm các biến sau vào file `.env` của backend:

```env id="fu30bb"
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
GEMINI_EMBEDDING_MODEL="text-embedding-004"
```

Nếu hệ thống dùng AI Provider mặc định là Gemini, thêm:

```env id="xqvccg"
AI_PROVIDER="gemini"
```

Nếu Gemini chỉ dùng làm provider dự phòng, có thể để:

```env id="8302w9"
AI_PROVIDER="openai"
```

và AI Gateway sẽ fallback sang Gemini khi OpenAI lỗi.

---

#### Bước 4: Cài đặt Gemini SDK

Trong thư mục backend, cài đặt package:

```bash id="h2y5jv"
npm install @google/generative-ai
```

Nếu dùng TypeScript, kiểm tra dự án đã có cấu hình TypeScript phù hợp.

---

#### Bước 5: Tạo Gemini Service

Tạo file:

```text id="m832re"
src/services/ai/gemini.service.ts
```

Ví dụ service cơ bản:

```ts id="v52pvd"
import { GoogleGenerativeAI } from "@google/generative-ai";

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || "");

const model = genAI.getGenerativeModel({
  model: process.env.GEMINI_MODEL || "gemini-1.5-flash"
});

export async function analyzeWithGemini(ocrText: string) {
  const prompt = `
You are DocuMind AI, a document intelligence assistant.

Analyze the OCR text below and return JSON only.

Rules:
- Do not return markdown.
- Do not wrap the response in code block.
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
`;

  const result = await model.generateContent(prompt);
  const text = result.response.text();

  return JSON.parse(text);
}
```

Trong dự án thật, bạn nên thêm xử lý lỗi JSON parsing, timeout và rate limit.

---

#### Bước 6: Chuẩn hóa kết quả trả về

Gemini Service nên trả về dữ liệu theo format thống nhất để AI Gateway có thể dùng chung với OpenAI.

Ví dụ:

```json id="sda721"
{
  "provider": "gemini",
  "model": "gemini-1.5-flash",
  "documentType": "Invoice",
  "summary": "This document is an invoice for cloud service usage.",
  "entities": [
    {
      "type": "vendor",
      "value": "AWS"
    }
  ],
  "keywords": ["invoice", "cloud", "payment"],
  "importantInformation": [
    "Invoice date: 2026-06-10",
    "Total amount: 120 USD"
  ],
  "confidence": 0.86
}
```

---

#### Bước 7: Tạo API test Gemini

Tạo route test tạm thời để kiểm tra Gemini hoạt động.

Ví dụ endpoint:

```text id="bd5fkm"
POST /api/ai/test-gemini
```

Request body:

```json id="e983me"
{
  "text": "Invoice No: INV-001. Vendor: AWS. Total: 120 USD."
}
```

Response mong đợi:

```json id="s2skhw"
{
  "success": true,
  "provider": "gemini",
  "model": "gemini-1.5-flash",
  "result": {
    "documentType": "Invoice",
    "summary": "The document is an invoice from AWS.",
    "confidence": 0.86
  }
}
```

---

#### Bước 8: Test bằng Postman

Mở Postman và tạo request:

```text id="qcfmhh"
Method: POST
URL: http://localhost:3000/api/ai/test-gemini
```

Body dạng JSON:

```json id="br299p"
{
  "text": "Invoice No INV-2026-001. Customer: Tuan Quang. Total amount: 2,000,000 VND."
}
```

Kết quả mong đợi là Gemini trả về JSON phân tích tài liệu.

---

#### Bước 9: Xử lý lỗi thường gặp

| Lỗi                      | Nguyên nhân                                  | Cách xử lý                           |
| ------------------------ | -------------------------------------------- | ------------------------------------ |
| `API key not valid`      | Sai Gemini API Key                           | Tạo lại key và cập nhật `.env`       |
| `403 Permission denied`  | API key chưa có quyền hoặc project chưa đúng | Kiểm tra project Google AI Studio    |
| `429 Resource exhausted` | Vượt quota                                   | Chờ quota reset hoặc đổi provider    |
| `JSON.parse error`       | Gemini trả về markdown hoặc text ngoài JSON  | Siết prompt và thêm hàm clean JSON   |
| `model not found`        | Sai tên model                                | Kiểm tra biến `GEMINI_MODEL`         |
| Timeout                  | Request AI phản hồi chậm                     | Thêm timeout và fallback sang OpenAI |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo Gemini API Key.
* API key đã được thêm vào `.env`.
* Backend đã cài package `@google/generative-ai`.
* Đã tạo Gemini Service.
* Endpoint test Gemini hoạt động.
* Gemini trả về kết quả JSON.
* Backend không expose API key ra frontend.
* Gemini sẵn sàng tích hợp vào AI Gateway.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã tích hợp thành công Gemini API ở mức backend service. Hệ thống có thể gửi OCR text sang Gemini để phân tích tài liệu và chuẩn bị tích hợp Gemini cùng OpenAI vào AI Gateway ở các bước tiếp theo.
