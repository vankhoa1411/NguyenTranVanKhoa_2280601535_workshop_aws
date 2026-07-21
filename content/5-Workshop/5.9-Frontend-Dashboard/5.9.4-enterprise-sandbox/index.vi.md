
---
title : "Enterprise Sandbox"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.9.4. </b> "
---

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng và kiểm tra **Enterprise Sandbox** cho hệ thống **DocuMind AI**.

Enterprise Sandbox là khu vực nâng cao cho phép người dùng thử nghiệm các chức năng phân tích tài liệu chuyên sâu như multi-model document analytics, semantic search simulation, AI Chat/RAG, auto-classification, entity extraction và export analytics.

Trong dự án hiện tại, Enterprise Sandbox sử dụng hai AI provider chính là **OpenAI ChatGPT API** và **Google Gemini API**. Không sử dụng Amazon Bedrock trong phần này.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

- Xây dựng giao diện Enterprise Sandbox.
- Cho phép chọn AI provider: OpenAI hoặc Gemini.
- Hiển thị AI Embeddings Engine ở dạng provider-neutral.
- Test semantic search simulation.
- Test RAG question answering.
- Test auto-classification và entity extraction.
- Hiển thị kết quả analytics.
- Cho phép export kết quả nếu có.
- Loại bỏ toàn bộ nội dung Bedrock khỏi UI và backend của phần này.

---

#### Vai trò của Enterprise Sandbox

Enterprise Sandbox giúp mô phỏng môi trường phân tích tài liệu cấp doanh nghiệp. Người dùng có thể:

- Thử nghiệm nhiều model AI trên cùng một tài liệu.
- So sánh kết quả giữa OpenAI và Gemini.
- Kiểm tra mức độ liên quan semantic search.
- Đặt các câu hỏi RAG phức tạp.
- Trích xuất entities và thông tin quan trọng.
- Xem confidence score.
- Xuất kết quả phân tích phục vụ báo cáo.

Nội dung mô tả đề xuất:

```text
Execute professional multi-model document analytics using Gemini and OpenAI. Train vector schemas, test semantic relevance, ask contextual RAG questions, and export analytics.