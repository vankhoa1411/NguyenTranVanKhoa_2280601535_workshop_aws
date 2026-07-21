---
title : "Create AI Gateway"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.6.3. </b> "
---

#### Overview

In this step, we will build the **AI Gateway Service** for the **DocuMind AI** system.

The AI Gateway is an intermediate layer between the OCR text and AI providers such as **Google Gemini API** and **OpenAI ChatGPT API**. Instead of allowing the backend or worker to call each provider directly, the system will route requests through the AI Gateway. The Gateway will determine which provider to use, standardize the prompt, handle errors, perform fallback operations when the primary provider fails, and return the analysis results in a unified structure.

This design makes the system easy to scale, simple to maintain, and allows adding new AI providers in the future with minimal impact on the rest of the codebase.

---

#### Objectives of This Step

Upon completing this step, you will:

* Create the AI Gateway Service.
* Define a common interface for the AI providers.
* Integrate the Gemini Service and the OpenAI Service into the same gateway.
* Allow provider selection via environment variables or request parameters.
* Implement a fallback mechanism between Gemini and OpenAI.
* Standardize the AI Analysis results format.
* Prepare the AI data for the Dashboard, AI Chat, and Enterprise Sandbox.

---

#### The Role of AI Gateway in DocuMind AI

In the DocuMind AI system, the AI Gateway is responsible for:

* Receiving OCR text from the worker or backend.
* Selecting the primary AI provider based on configuration.
* Invoking Gemini or OpenAI to analyze the document.
* Falling back to the backup provider if the primary provider fails.
* Standardizing the returned response.
* Logging the provider, model, processing duration, and any errors.
* Storing results in PostgreSQL via Prisma.
* Providing the foundation for AI Chat/RAG and Semantic Search.

Processing flow:

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

> ⚠️ **Screenshot Suggestion:**
> Draw a diagram illustrating the AI Gateway receiving OCR Text, selecting Gemini or OpenAI, performing fallback when one provider fails, and returning a Normalized AI Result.
> Save the image at the path:
> `/static/images/5-Workshop/5.6-Integrate-AI-Providers/5.6.3-create-ai-gateway/ai-gateway.png`

---

#### Recommended Directory Structure

You can organize the source code as follows:

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

File descriptions:

| File                | Role                                      |
| ------------------- | ----------------------------------------- |
| `ai.interface.ts`   | Defines the common structure for AI providers |
| `ai.gateway.ts`     | Coordinates AI analysis and fallback logic |
| `ai.factory.ts`     | Selects the provider based on configuration |
| `gemini.service.ts` | Calls the Gemini API                      |
| `openai.service.ts` | Calls the OpenAI API                      |
| `prompt.builder.ts` | Builds shared prompts                     |
| `ai.controller.ts`  | Handles test requests or AI APIs          |
| `ai.routes.ts`      | Declares AI routes                        |

---

#### Create AI Provider Interface

Create the file:

```text
src/services/ai/ai.interface.ts
```

Example interface:

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

This interface ensures that Gemini and OpenAI return responses in the same format, preventing inconsistent outputs across providers.

---

#### Create Prompt Builder

Create the file:

```text
src/services/ai/prompt.builder.ts
```

The prompt should instruct the AI to output a clean JSON object:

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

#### Create AI Factory

Create the file:

```text
src/services/ai/ai.factory.ts
```

The Factory is used to instantiate the selected provider:

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

#### Create AI Gateway Service

Create the file:

```text
src/services/ai/ai.gateway.ts
```

Example gateway implementation logic:

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

#### Standardize JSON Responses

AI providers sometimes return JSON wrapped in markdown code blocks. Therefore, implement a safe JSON parser:

```ts
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
```

---

#### Storing AI Analysis Results in the Database

After the Gateway returns the results, the system should save them in the `AIAnalysis` table.

Example data record:

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

At the same time, update the Document status:

```text
COMPLETED
```

If an error occurs:

```text
FAILED
```

---

#### Related Environment Variables

```env
AI_PROVIDER="openai"
 
OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"
 
GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"
```

You can change the primary provider by updating:

```env
AI_PROVIDER="gemini"
```

---

#### Logging Requirements

The AI Gateway should log key events:

```text
[AI_ANALYSIS_STARTED]
[AI_PROVIDER_SELECTED]
[AI_PROVIDER_PRIMARY_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_FAILED]
```

Do not log API keys or the full document content if it contains sensitive information.

---

#### Completion Checklist

You have completed this step when:

* The AI Provider Interface is defined.
* The Gemini Service and OpenAI Service both adhere to the same output format.
* The AI Factory instantiates the correct provider based on `.env`.
* The AI Gateway supports fallback capabilities between OpenAI and Gemini.
* AI responses are successfully cleaned and parsed into JSON.
* AI Analysis results can be saved in PostgreSQL.
* The AI Gateway does not reference Bedrock.
* API keys are not exposed to the frontend.

---

#### Expected Outcome

Following this step, DocuMind AI has an operational AI Gateway to coordinate Gemini and OpenAI. The system can analyze OCR text using the preferred provider, automatically fall back when the primary provider fails, and output standardized AI Analysis results.
