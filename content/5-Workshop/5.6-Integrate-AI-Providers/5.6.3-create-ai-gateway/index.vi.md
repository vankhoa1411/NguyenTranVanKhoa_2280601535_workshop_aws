---

title : "Tạo AI Gateway"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.6.3. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng **AI Gateway Service** cho hệ thống **DocuMind AI**.

AI Gateway là lớp trung gian giữa phần OCR text và các AI provider như **Google Gemini API** và **OpenAI ChatGPT API**. Thay vì để backend hoặc worker gọi trực tiếp từng provider riêng lẻ, hệ thống sẽ gọi thông qua AI Gateway. Gateway sẽ quyết định sử dụng provider nào, chuẩn hóa prompt, xử lý lỗi, fallback khi provider chính thất bại và trả về kết quả phân tích theo cùng một cấu trúc.

Cách thiết kế này giúp hệ thống dễ mở rộng, dễ bảo trì và có thể thêm AI provider mới trong tương lai mà không ảnh hưởng nhiều đến các phần còn lại.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được AI Gateway Service.
* Tạo interface chung cho các AI provider.
* Tích hợp Gemini Service và OpenAI Service vào cùng một gateway.
* Cho phép chọn provider bằng biến môi trường hoặc request.
* Tạo cơ chế fallback giữa Gemini và OpenAI.
* Chuẩn hóa kết quả AI Analysis.
* Chuẩn bị dữ liệu cho Dashboard, AI Chat và Enterprise Sandbox.

---

#### Vai trò của AI Gateway trong DocuMind AI

Trong hệ thống DocuMind AI, AI Gateway có nhiệm vụ:

* Nhận OCR text từ worker hoặc backend.
* Chọn AI provider chính theo cấu hình.
* Gọi Gemini hoặc OpenAI để phân tích tài liệu.
* Fallback sang provider còn lại nếu provider chính lỗi.
* Chuẩn hóa kết quả trả về.
* Ghi log provider, model, thời gian xử lý và lỗi nếu có.
* Lưu kết quả vào PostgreSQL thông qua Prisma.
* Cung cấp nền tảng cho AI Chat/RAG và Semantic Search.

Luồng xử lý:

```text
OCR Text
  |
  v
AI Gateway
  |
  |----------------------|
  |                      |
  v                      v
Gemini Service       OpenAI Service
  |                      |
  |----------------------|
           |
           v
Normalized AI Result
```

![ai-gateway](/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.3-create-ai-gateway/ai-gateway.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ AI Gateway nhận OCR Text, sau đó chọn Gemini hoặc OpenAI, fallback khi một provider lỗi và trả về Normalized AI Result.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.3-create-ai-gateway/ai-gateway.png`

---

#### Cấu trúc thư mục đề xuất

Có thể tổ chức source code như sau:

```text
src/
├── services/
│   └── ai/
│       ├── ai.interface.ts
│       ├── ai.gateway.ts
│       ├── ai.factory.ts
│       ├── gemini.service.ts
│       ├── openai.service.ts
│       └── prompt.builder.ts
├── controllers/
│   └── ai.controller.ts
└── routes/
    └── ai.routes.ts
```

Vai trò từng file:

| File                | Vai trò                                   |
| ------------------- | ----------------------------------------- |
| `ai.interface.ts`   | Định nghĩa cấu trúc chung cho AI provider |
| `ai.gateway.ts`     | Điều phối phân tích AI và fallback        |
| `ai.factory.ts`     | Chọn provider theo cấu hình               |
| `gemini.service.ts` | Gọi Gemini API                            |
| `openai.service.ts` | Gọi OpenAI API                            |
| `prompt.builder.ts` | Tạo prompt dùng chung                     |
| `ai.controller.ts`  | Xử lý request test hoặc API AI            |
| `ai.routes.ts`      | Khai báo route AI                         |

---

#### Tạo AI Provider Interface

Tạo file:

```text
src/services/ai/ai.interface.ts
```

Ví dụ interface:

```ts
export type AIProviderName = "gemini" | "openai";

export interface AIAnalysisResult {
  provider: AIProviderName;
  model: string;
  documentType: string | null;
  summary: string | null;
  entities: Array<{
    type: string;
    value: string;
  }>;
  keywords: string[];
  importantInformation: string[];
  confidence: number;
}

export interface AIProvider {
  analyzeDocument(ocrText: string): Promise<AIAnalysisResult>;
  chatWithDocument?(question: string, context: string): Promise<any>;
  generateEmbedding?(text: string): Promise<number[]>;
}
```

Interface này giúp Gemini và OpenAI trả về cùng format, tránh mỗi provider một kiểu response khác nhau.

---

#### Tạo Prompt Builder

Tạo file:

```text
src/services/ai/prompt.builder.ts
```

Prompt nên yêu cầu AI trả về JSON rõ ràng:

```ts
export function buildDocumentAnalysisPrompt(ocrText: string) {
  return `
You are DocuMind AI, a document intelligence assistant.

Analyze the OCR text below and return JSON only.

Rules:
- Return JSON only.
- Do not use markdown.
- Do not wrap the result in code block.
- If a field is not found, return null.
- Do not invent information.
- Use the same language as the document when possible.

OCR Text:
${ocrText}

Return JSON format:
{
  "documentType": "",
  "summary": "",
  "entities": [
    {
      "type": "",
      "value": ""
    }
  ],
  "keywords": [],
  "importantInformation": [],
  "confidence": 0
}
`;
}
```

---

#### Tạo AI Factory

Tạo file:

```text
src/services/ai/ai.factory.ts
```

Factory dùng để chọn provider:

```ts
import { GeminiService } from "./gemini.service";
import { OpenAIService } from "./openai.service";
import type { AIProvider, AIProviderName } from "./ai.interface";

export function getAIProvider(provider?: AIProviderName): AIProvider {
  const selectedProvider = provider || (process.env.AI_PROVIDER as AIProviderName) || "openai";

  if (selectedProvider === "gemini") {
    return new GeminiService();
  }

  if (selectedProvider === "openai") {
    return new OpenAIService();
  }

  throw new Error(`Unsupported AI provider: ${selectedProvider}`);
}
```

---

#### Tạo AI Gateway Service

Tạo file:

```text
src/services/ai/ai.gateway.ts
```

Ví dụ logic gateway:

```ts
import { getAIProvider } from "./ai.factory";
import type { AIProviderName, AIAnalysisResult } from "./ai.interface";

export async function analyzeDocumentWithGateway(
  ocrText: string,
  preferredProvider?: AIProviderName
): Promise<AIAnalysisResult> {
  const primaryProvider = preferredProvider || (process.env.AI_PROVIDER as AIProviderName) || "openai";
  const fallbackProvider: AIProviderName = primaryProvider === "openai" ? "gemini" : "openai";

  try {
    console.log("[AI_PROVIDER_SELECTED]", primaryProvider);

    const provider = getAIProvider(primaryProvider);
    return await provider.analyzeDocument(ocrText);
  } catch (primaryError) {
    console.error("[AI_PROVIDER_PRIMARY_FAILED]", primaryProvider, primaryError);

    try {
      console.log("[AI_PROVIDER_FALLBACK_USED]", fallbackProvider);

      const fallback = getAIProvider(fallbackProvider);
      return await fallback.analyzeDocument(ocrText);
    } catch (fallbackError) {
      console.error("[AI_PROVIDER_FALLBACK_FAILED]", fallbackError);
      throw new Error("All AI providers failed.");
    }
  }
}
```

---

#### Chuẩn hóa JSON response

AI provider đôi khi trả về JSON kèm markdown. Vì vậy nên có hàm xử lý response:

````ts
export function safeParseAIJson(rawText: string) {
  const cleaned = rawText
    .replace(/```json/g, "")
    .replace(/```/g, "")
    .trim();

  try {
    return JSON.parse(cleaned);
  } catch (error) {
    throw new Error("AI response is not valid JSON");
  }
}
````

---

#### Lưu kết quả AI Analysis vào database

Sau khi Gateway trả về kết quả, hệ thống nên lưu vào bảng `AIAnalysis`.

Ví dụ dữ liệu lưu:

```json
{
  "documentId": "doc_001",
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "documentType": "Invoice",
  "summary": "This document is an invoice.",
  "entities": [],
  "keywords": [],
  "importantInformation": [],
  "confidence": 0.9
}
```

Đồng thời cập nhật trạng thái Document:

```text
COMPLETED
```

Nếu lỗi:

```text
FAILED
```

---

#### Biến môi trường liên quan

```env
AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

Có thể đổi provider chính bằng cách thay:

```env
AI_PROVIDER="gemini"
```

---

#### Logging cần có

AI Gateway nên ghi log:

```text
[AI_ANALYSIS_STARTED]
[AI_PROVIDER_SELECTED]
[AI_PROVIDER_PRIMARY_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_FAILED]
```

Không log API key hoặc toàn bộ nội dung tài liệu nếu tài liệu có thông tin nhạy cảm.

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã có AI Provider Interface.
* Gemini Service và OpenAI Service cùng tuân theo một format.
* AI Factory chọn được provider theo `.env`.
* AI Gateway có fallback giữa OpenAI và Gemini.
* AI response được chuẩn hóa thành JSON.
* Kết quả AI Analysis có thể lưu vào PostgreSQL.
* Không còn gọi Bedrock trong AI Gateway.
* API key không bị expose ra frontend.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có AI Gateway để điều phối Gemini và OpenAI. Hệ thống có thể phân tích OCR text bằng provider được chọn, tự fallback khi provider chính lỗi và trả về kết quả AI Analysis theo format thống nhất.
