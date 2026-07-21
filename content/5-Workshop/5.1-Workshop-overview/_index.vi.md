---
title : "Giới thiệu"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Giới thiệu về DocuMind AI

**DocuMind AI** là nền tảng trích xuất, phân tích và hỏi đáp tài liệu tự động dựa trên kiến trúc Cloud-Native. Hệ thống hỗ trợ người dùng tải lên các tài liệu như hóa đơn, biểu mẫu, hợp đồng, báo cáo hoặc tài liệu nội bộ; sau đó tự động trích xuất nội dung bằng OCR, phân tích bằng AI và hiển thị kết quả trên dashboard.

Trong workshop này, bạn sẽ xây dựng luồng xử lý tài liệu hoàn chỉnh, kết hợp các dịch vụ AWS như **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Amazon RDS PostgreSQL**, **Amazon CloudWatch**, **AWS Secrets Manager**, **IAM Role** và **AWS WAF**. Phần AI của hệ thống sử dụng hai provider chính là **Google Gemini API** và **OpenAI ChatGPT API** để phân tích tài liệu, tóm tắt nội dung, trích xuất thông tin quan trọng và hỗ trợ AI Chat/RAG.

#### Mục tiêu của workshop

Sau khi hoàn thành workshop, bạn có thể:

- Xây dựng hệ thống upload và lưu trữ tài liệu trên Amazon S3.
- Tạo hàng đợi xử lý bất đồng bộ bằng Amazon SQS.
- Sử dụng Amazon Textract để OCR tài liệu PDF hoặc hình ảnh.
- Tích hợp Gemini API và OpenAI ChatGPT API làm hai AI provider chính.
- Lưu metadata, OCR result, AI analysis, notification và audit log vào Amazon RDS PostgreSQL.
- Theo dõi log và lỗi hệ thống bằng Amazon CloudWatch.
- Quản lý secret bằng AWS Secrets Manager thay vì hardcode API key.
- Tăng cường bảo mật request bằng IAM Role và AWS WAF.
- Xây dựng dashboard, AI Chat, Enterprise Sandbox, Notification Center và Admin Panel.

#### Kiến trúc tổng quan của workshop

Workshop được thiết kế theo mô hình **Cloud-Native Document Processing Pipeline** kết hợp xử lý bất đồng bộ bằng hàng đợi tin nhắn. Frontend giao tiếp với Backend API, Backend lưu tài liệu vào S3 và gửi message vào SQS. Worker nhận message, gọi Textract để trích xuất văn bản, sau đó gửi nội dung sang AI Gateway để xử lý bằng Gemini hoặc OpenAI. Kết quả cuối cùng được lưu vào PostgreSQL và hiển thị lại cho người dùng.

![overview](/static/images/5-Workshop/5.1-Workshop-overview/architecture.png)



#### Luồng hoạt động của hệ thống

1. **Người dùng tải tài liệu lên hệ thống**  
   Người dùng upload file PDF, PNG, JPG hoặc JPEG thông qua giao diện React. Backend kiểm tra định dạng, lưu file vào **Amazon S3** và tạo metadata ban đầu trong **Amazon RDS PostgreSQL** với trạng thái `PENDING`.

2. **Backend gửi job vào hàng đợi**  
   Sau khi upload thành công, Backend gửi message chứa `documentId`, `s3Key`, `userId` và thông tin xử lý vào **Amazon SQS**. Cách này giúp hệ thống xử lý tài liệu bất đồng bộ, tránh làm người dùng phải chờ lâu trên giao diện.

3. **Worker xử lý OCR bằng Amazon Textract**  
   Worker nhận message từ SQS, lấy tài liệu từ S3 và gọi **Amazon Textract** để trích xuất văn bản. Kết quả OCR được lưu vào database và trạng thái tài liệu được cập nhật sang `OCR_COMPLETED` hoặc `FAILED` nếu lỗi.

4. **AI Gateway phân tích tài liệu**  
   Văn bản sau OCR được gửi đến **AI Gateway Service**. Gateway lựa chọn provider chính theo cấu hình hoặc lựa chọn của người dùng, gồm **OpenAI ChatGPT API** hoặc **Google Gemini API**. AI thực hiện tóm tắt, phân loại, trích xuất thực thể, tạo JSON có cấu trúc và hỗ trợ hỏi đáp tài liệu theo ngữ cảnh.

5. **Lưu kết quả và gửi thông báo**  
   Kết quả AI Analysis, Chat History, Notification và Audit Log được lưu vào **Amazon RDS PostgreSQL**. Người dùng nhận thông báo khi tài liệu được xử lý xong, còn Admin có thể xem lỗi hệ thống, lỗi AI provider, trạng thái xử lý và audit log.

6. **Giám sát và bảo mật**  
   **CloudWatch** ghi log backend, worker, OCR và AI processing. **Secrets Manager** quản lý các API key và secret quan trọng. **IAM Role** cấp quyền an toàn cho backend/EC2 truy cập AWS services. **AWS WAF** bảo vệ request khỏi các tấn công phổ biến như SQL Injection, XSS, brute force và traffic bất thường.

#### Các thành phần chính trong workshop

- **Frontend Web App**: Home Page, Dashboard, Upload, Document Detail, AI Chat, Enterprise Sandbox, Notification Center và Admin Panel.
- **Backend API**: Xử lý authentication, upload tài liệu, quản lý document, gọi SQS, gọi AI service và cung cấp dữ liệu cho frontend.
- **Document Worker**: Nhận job từ SQS, gọi Textract OCR, gọi AI Gateway và cập nhật kết quả.
- **AI Gateway**: Quản lý hai provider chính là Gemini và OpenAI, hỗ trợ fallback khi một provider lỗi hoặc bị giới hạn quota.
- **Database**: Lưu users, roles, documents, OCR results, AI analysis, chat history, notifications và audit logs.
- **Monitoring & Security**: CloudWatch, Secrets Manager, IAM Role và AWS WAF.

#### Kết quả kỳ vọng sau workshop

Sau workshop, bạn sẽ có một hệ thống DocuMind AI có khả năng upload tài liệu, OCR, phân tích bằng AI, hỏi đáp tài liệu, hiển thị dashboard, gửi thông báo cho User/Admin và theo dõi lỗi hệ thống. Kiến trúc này có thể mở rộng cho các bài toán thực tế như quản lý hóa đơn, phân tích hợp đồng, xử lý biểu mẫu, quản lý tài liệu nội bộ hoặc xây dựng hệ thống quản trị tri thức doanh nghiệp.
