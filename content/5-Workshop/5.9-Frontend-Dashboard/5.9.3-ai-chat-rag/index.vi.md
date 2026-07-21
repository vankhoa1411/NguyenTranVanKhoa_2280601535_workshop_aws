
---
title : "AI Chat và RAG"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.9.3. </b> "
---

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng chức năng **AI Chat/RAG** cho hệ thống **DocuMind AI**.

AI Chat/RAG cho phép người dùng đặt câu hỏi trực tiếp trên nội dung tài liệu đã được OCR và phân tích. Thay vì phải đọc toàn bộ tài liệu, người dùng có thể hỏi các câu như “Tài liệu này nói về gì?”, “Tổng tiền là bao nhiêu?”, “Ai là nhà cung cấp?”, “Tóm tắt các điểm quan trọng”, hoặc “Trích xuất các điều khoản chính”.

RAG là viết tắt của **Retrieval-Augmented Generation**, tức là hệ thống sẽ tìm các đoạn nội dung liên quan trong tài liệu, sau đó gửi ngữ cảnh đó cho Gemini hoặc OpenAI để tạo câu trả lời chính xác hơn.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

- Tạo giao diện AI Chat cho từng tài liệu.
- Cho phép người dùng đặt câu hỏi dựa trên nội dung tài liệu.
- Lấy OCR text hoặc document chunks làm context.
- Gọi AI Gateway để trả lời bằng Gemini hoặc OpenAI.
- Lưu lịch sử chat vào PostgreSQL.
- Hiển thị câu trả lời, provider, model và confidence.
- Kiểm tra quyền truy cập tài liệu trước khi chat.
- Chuẩn bị nền tảng cho Enterprise Sandbox.

---

#### Vai trò của AI Chat/RAG trong DocuMind AI

AI Chat/RAG giúp người dùng:

- Hỏi đáp với tài liệu theo ngữ cảnh.
- Tìm nhanh thông tin quan trọng.
- Tóm tắt nội dung dài.
- Trích xuất dữ liệu cụ thể từ tài liệu.
- Giảm thời gian đọc tài liệu thủ công.
- Tăng trải nghiệm tương tác với hệ thống AI document analytics.

Ví dụ câu hỏi:

```text
Tài liệu này nói về nội dung gì?
Tổng số tiền trong hóa đơn là bao nhiêu?
Ai là khách hàng?
Ai là nhà cung cấp?
Ngày phát hành tài liệu là ngày nào?
Tóm tắt tài liệu trong 5 dòng.
Các thông tin quan trọng nhất là gì?