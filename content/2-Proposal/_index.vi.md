---
title: "Bản đề xuất"
date: 2026-06-10
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

Tại phần này, bạn cần tóm tắt các nội dung trong workshop mà bạn **dự tính** sẽ làm.

# DocuMind AI – Cloud-Native AI Document Extractor & Analytics
## Giải pháp AWS Cloud kết hợp AI cho trích xuất, phân tích và hỏi đáp tài liệu tự động

### 1. Tóm tắt điều hành
DocuMind AI là một nền tảng trích xuất và phân tích tài liệu tự động, được xây dựng nhằm hỗ trợ người dùng xử lý các loại tài liệu như hóa đơn, biểu mẫu, hợp đồng, báo cáo hoặc tài liệu nội bộ. Hệ thống cho phép người dùng tải tài liệu lên, tự động trích xuất nội dung bằng OCR, phân tích dữ liệu bằng AI, phân loại tài liệu, tóm tắt nội dung, trích xuất thông tin quan trọng và đặt câu hỏi trực tiếp trên tài liệu thông qua chức năng AI Chat/RAG.

Nền tảng được xây dựng theo kiến trúc cloud-native, sử dụng các dịch vụ AWS như Amazon S3, Amazon SQS, Amazon Textract, PostgreSQL, CloudWatch, Secrets Manager, IAM Role và AWS WAF. Phần trí tuệ nhân tạo được tích hợp với hai mô hình chính là Google Gemini API và OpenAI ChatGPT API nhằm tăng tính linh hoạt, đảm bảo khả năng fallback khi một nhà cung cấp AI gặp lỗi hoặc giới hạn quota.

Mục tiêu của dự án là xây dựng một hệ thống xử lý tài liệu hiện đại, có khả năng mở rộng, bảo mật tốt, dễ giám sát và phù hợp với nhu cầu thực tế của doanh nghiệp hoặc tổ chức trong việc số hóa và phân tích tài liệu.

---

### 2. Tuyên bố vấn đề

#### Vấn đề hiện tại
Trong nhiều doanh nghiệp và tổ chức, việc xử lý tài liệu vẫn còn phụ thuộc nhiều vào thao tác thủ công. Người dùng phải đọc từng tài liệu, nhập lại dữ liệu quan trọng, tự phân loại tài liệu và tổng hợp thông tin cần thiết. Quá trình này tốn thời gian, dễ xảy ra sai sót và khó mở rộng khi số lượng tài liệu tăng lên.

Ngoài ra, nhiều hệ thống hiện tại chỉ hỗ trợ lưu trữ tài liệu mà chưa có khả năng hiểu nội dung tài liệu, tóm tắt thông tin, trích xuất dữ liệu có cấu trúc hoặc hỗ trợ người dùng hỏi đáp trực tiếp trên nội dung tài liệu. Việc thiếu hệ thống giám sát, cảnh báo lỗi và quản lý trạng thái xử lý cũng gây khó khăn trong vận hành thực tế.

#### Giải pháp đề xuất
DocuMind AI cung cấp một nền tảng xử lý tài liệu tự động dựa trên AWS Cloud và AI. Người dùng có thể tải tài liệu lên hệ thống, tài liệu sẽ được lưu trữ trên Amazon S3. Sau đó, hệ thống gửi tác vụ xử lý vào Amazon SQS để đảm bảo xử lý bất đồng bộ và tránh quá tải. Amazon Textract được sử dụng để trích xuất văn bản từ tài liệu. Nội dung sau OCR sẽ được gửi đến Gemini API hoặc OpenAI ChatGPT API để thực hiện các tác vụ phân tích như tóm tắt, phân loại, trích xuất thực thể, tạo dữ liệu có cấu trúc và hỗ trợ hỏi đáp tài liệu.

Kết quả xử lý được lưu trữ trong PostgreSQL thông qua Prisma ORM. Người dùng có thể xem trạng thái xử lý, kết quả OCR, kết quả phân tích AI, lịch sử chat và thông báo trên giao diện web. Hệ thống cũng có khu vực quản trị dành cho Admin để theo dõi người dùng, tài liệu, lỗi xử lý, thông báo, audit log và trạng thái hệ thống.

#### Lợi ích và giá trị mang lại
Giải pháp giúp giảm thời gian xử lý tài liệu thủ công, tăng độ chính xác trong việc trích xuất và tổng hợp thông tin, đồng thời cung cấp trải nghiệm thông minh thông qua AI Chat. Hệ thống có thể áp dụng cho nhiều loại tài liệu như hóa đơn, hợp đồng, hồ sơ, báo cáo hoặc tài liệu doanh nghiệp.

Về mặt kỹ thuật, dự án giúp chứng minh khả năng thiết kế và triển khai một hệ thống cloud-native có tích hợp nhiều dịch vụ AWS, xử lý bất đồng bộ bằng queue, bảo mật bằng IAM Role, Secrets Manager và AWS WAF, giám sát bằng CloudWatch, đồng thời tích hợp mô hình AI hiện đại như Gemini và ChatGPT.

---

### 3. Kiến trúc giải pháp
DocuMind AI áp dụng kiến trúc cloud-native kết hợp xử lý bất đồng bộ. Người dùng tương tác với giao diện web React. Khi tài liệu được tải lên, backend Node.js/Express sẽ lưu file vào Amazon S3 và tạo metadata trong PostgreSQL. Sau đó, hệ thống gửi message vào Amazon SQS để worker xử lý tài liệu.

Worker nhận message từ SQS, gọi Amazon Textract để trích xuất văn bản từ tài liệu. Sau khi có kết quả OCR, hệ thống gửi nội dung này đến AI Provider Gateway. AI Provider Gateway cho phép lựa chọn hoặc fallback giữa Google Gemini API và OpenAI ChatGPT API. Kết quả phân tích được lưu vào PostgreSQL và hiển thị lại trên Dashboard.

Kiến trúc tổng quan:

User
→ React Frontend
→ Backend API Node.js/Express
→ Amazon S3 lưu tài liệu
→ Amazon SQS xử lý hàng đợi
→ Amazon Textract OCR
→ AI Provider Gateway
→ Gemini API / OpenAI ChatGPT API
→ Cơ sở dữ liệu PostgreSQL
→ Dashboard / AI Chat / Analytics

#### Dịch vụ AWS sử dụng

- Amazon S3: Lưu trữ tài liệu gốc và hỗ trợ quản lý file tài liệu.
- Amazon SQS: Quản lý hàng đợi xử lý tài liệu bất đồng bộ.
- Amazon Textract: Trích xuất văn bản từ PDF, PNG, JPG hoặc JPEG.
- PostgreSQL: Lưu dữ liệu người dùng, tài liệu, OCR, AI analysis, chat history, notification và audit log.
- Amazon CloudWatch: Ghi log, theo dõi lỗi backend, lỗi AI, lỗi OCR và trạng thái xử lý.
- AWS Secrets Manager: Lưu trữ thông tin nhạy cảm như database URL, JWT secret, OpenAI API key và Gemini API key.
- IAM Role: Cấp quyền an toàn cho backend/EC2 truy cập các dịch vụ AWS mà không hardcode access key.
- AWS WAF: Bảo vệ ứng dụng khỏi các request độc hại như SQL Injection, XSS, brute force và traffic bất thường.

#### Thành phần hệ thống

- Frontend: Giao diện người dùng gồm Home Page, Dashboard, Document Management, AI Models, Enterprise Sandbox, Notification Center và Admin Panel.
- Backend API: Xử lý authentication, upload tài liệu, quản lý document, gọi AI service và cung cấp API cho frontend.
- Document Processing Worker: Nhận job từ SQS, xử lý OCR và gọi AI.
- AI Gateway Service: Quản lý hai AI provider chính là Gemini và OpenAI.
- Admin Module: Quản lý người dùng, tài liệu, thông báo, audit log và trạng thái hệ thống.
- Notification System: Gửi thông báo cho User và Admin khi upload, OCR, AI analysis hoặc lỗi hệ thống xảy ra.

---

### 4. Triển khai kỹ thuật

#### Các giai đoạn triển khai
Dự án được chia thành nhiều giai đoạn triển khai:

1. Phân tích yêu cầu và thiết kế kiến trúc
Xác định chức năng chính của hệ thống, thiết kế kiến trúc cloud-native, lựa chọn dịch vụ AWS phù hợp và xây dựng sơ đồ tổng quan.
2. Xây dựng backend và cơ sở dữ liệu
Phát triển backend bằng Node.js/Express, thiết kế database PostgreSQL với Prisma ORM, xây dựng các API cho authentication, document, OCR, AI analysis và notification.
3. Tích hợp AWS Storage và Queue
Tích hợp Amazon S3 để lưu tài liệu và Amazon SQS để xử lý hàng đợi tài liệu bất đồng bộ.
4. Tích hợp OCR và AI
Sử dụng Amazon Textract để trích xuất văn bản từ tài liệu. Tích hợp Gemini API và OpenAI ChatGPT API để phân tích tài liệu, tóm tắt nội dung, phân loại và hỗ trợ hỏi đáp tài liệu.
5. Xây dựng giao diện người dùng
Phát triển giao diện React/TypeScript với TailwindCSS, bao gồm Home Page, Dashboard, Upload Page, Document Detail, AI Chat, Enterprise Sandbox, Notification Center và Admin Dashboard.
6. Bảo mật và giám sát
Tích hợp JWT authentication, role-based access control, IAM Role, Secrets Manager, CloudWatch, AWS WAF và Audit Log.
7. Kiểm thử và triển khai
Kiểm thử upload tài liệu, OCR, AI analysis, AI Chat, notification, admin module, logging và security. Sau đó triển khai backend lên EC2 và kết nối với PostgreSQL/S3/SQS.

#### Yêu cầu kỹ thuật

- Backend sử dụng Node.js, Express.js và TypeScript.
- Frontend sử dụng React, TypeScript, TailwindCSS và Framer Motion.
- Database sử dụng PostgreSQL.
- Prisma ORM dùng để quản lý schema và migration.
- JWT dùng cho authentication và phân quyền User/Admin.
- Amazon S3 dùng để lưu file tài liệu.
- Amazon SQS dùng để xử lý tác vụ bất đồng bộ.
- Amazon Textract dùng cho OCR.
- Gemini API và OpenAI ChatGPT API dùng cho phân tích tài liệu và AI Chat.
- CloudWatch dùng để theo dõi log và lỗi hệ thống.
- Secrets Manager dùng để quản lý secrets.
- AWS WAF dùng để tăng cường bảo mật request.

---

### 5. Lộ trình và mốc triển khai

- Giai đoạn 1: Phân tích yêu cầu và thiết kế kiến trúc hệ thống.
- Giai đoạn 2: Xây dựng backend, database schema và authentication.
- Giai đoạn 3: Tích hợp upload tài liệu lên S3 và xử lý hàng đợi bằng SQS.
- Giai đoạn 4: Tích hợp Textract OCR và AI Provider Gateway.
- Giai đoạn 5: Tích hợp Gemini API và OpenAI ChatGPT API.
- Giai đoạn 6: Xây dựng Dashboard, AI Chat, Semantic Search và Enterprise Sandbox.
- Giai đoạn 7: Xây dựng Notification System cho User và Admin.
- Giai đoạn 8: Tích hợp Admin Panel, Audit Log, CloudWatch và AWS WAF.
- Giai đoạn 9: Kiểm thử toàn hệ thống, tối ưu giao diện và triển khai lên AWS.

---

### 6. Ước tính ngân sách
Chi phí hạ tầng phụ thuộc vào số lượng tài liệu, dung lượng lưu trữ, số lần gọi OCR và số lần gọi AI API. Trong phạm vi thử nghiệm và đồ án, hệ thống có thể vận hành ở mức chi phí thấp bằng cách sử dụng tài nguyên nhỏ và giới hạn request.

Các thành phần có thể phát sinh chi phí:

- Amazon S3: Lưu trữ tài liệu gốc.
- Amazon SQS: Xử lý message queue.
- Amazon Textract: Trích xuất văn bản từ tài liệu.
- PostgreSQL: Lưu dữ liệu ứng dụng.
- Amazon EC2: Chạy backend API.
- Amazon CloudWatch: Lưu log và theo dõi hệ thống.
- AWS Secrets Manager: Lưu secrets.
- AWS WAF: Bảo vệ ứng dụng web/API.
- OpenAI API: Xử lý AI analysis, AI Chat và embeddings.
- Gemini API: Xử lý AI analysis, AI Chat và fallback provider.
Đối với môi trường demo, có thể tối ưu chi phí bằng cách:

- Giới hạn số lượng tài liệu test.
- Dùng EC2 instance nhỏ.
- Dùng cơ sở dữ liệu cấu hình thấp.
- Xóa dữ liệu test không cần thiết.
- Theo dõi billing bằng AWS Budgets.
- Chỉ bật WAF/CloudWatch logging ở mức cần thiết cho demo.

---

### 7. Đánh giá rủi ro

#### Ma trận rủi ro

- Lỗi OCR do định dạng tài liệu không hỗ trợ: Ảnh hưởng trung bình, xác suất trung bình.
- AI API bị quota hoặc rate limit: Ảnh hưởng cao, xác suất trung bình.
- Upload file lớn gây chậm hệ thống: Ảnh hưởng trung bình, xác suất trung bình.
- Lỗi database hoặc lỗi migration: Ảnh hưởng cao, xác suất thấp.
- Lộ API key hoặc thông tin nhạy cảm: Ảnh hưởng cao, xác suất thấp.
- Chi phí AWS/API tăng ngoài dự kiến: Ảnh hưởng trung bình, xác suất trung bình.
- Backend worker xử lý lỗi hoặc message bị fail: Ảnh hưởng trung bình, xác suất trung bình.

#### Chiến lược giảm thiểu

- Chỉ cho phép upload các định dạng được hỗ trợ như PDF, PNG, JPG và JPEG.
- Dùng SQS và retry mechanism để xử lý tài liệu ổn định hơn.
- Tích hợp fallback giữa Gemini và OpenAI khi một provider gặp lỗi.
- Dùng Secrets Manager để lưu API key và secret.
- Dùng IAM Role thay vì hardcode AWS access key.
- Dùng CloudWatch để theo dõi log, lỗi và trạng thái xử lý.
- Dùng AWS WAF để giảm rủi ro từ request độc hại.
- Dùng AWS Budgets để kiểm soát chi phí.

#### Kế hoạch dự phòng

- Nếu Gemini API lỗi, hệ thống chuyển sang OpenAI API.
- Nếu OpenAI API lỗi, hệ thống chuyển sang Gemini API.
- Nếu OCR thất bại, hệ thống cập nhật trạng thái FAILED và gửi thông báo cho người dùng.
- Nếu worker lỗi, message có thể retry hoặc chuyển vào dead-letter queue.
- Nếu hệ thống cloud tạm thời không ổn định, tài liệu vẫn được lưu ở S3 để xử lý lại sau.

---

### 8. Kết quả kỳ vọng
Sau khi hoàn thành, DocuMind AI có thể hỗ trợ người dùng tải tài liệu lên, trích xuất nội dung bằng OCR, phân tích tài liệu bằng AI, hỏi đáp theo ngữ cảnh tài liệu và theo dõi trạng thái xử lý trên dashboard.

Về mặt kỹ thuật, dự án thể hiện khả năng xây dựng một hệ thống cloud-native có nhiều thành phần thực tế như storage, queue, OCR, AI provider, database, monitoring, security và admin management. Đây là nền tảng có thể mở rộng để áp dụng vào các bài toán doanh nghiệp như quản lý hóa đơn, phân tích hợp đồng, xử lý biểu mẫu, tự động hóa tài liệu nội bộ hoặc xây dựng hệ thống quản trị tri thức.

Kết quả kỳ vọng bao gồm:

- Giảm thao tác đọc và nhập liệu thủ công.
- Tăng tốc độ xử lý tài liệu.
- Cung cấp dữ liệu có cấu trúc từ tài liệu phi cấu trúc.
- Hỗ trợ người dùng hỏi đáp trực tiếp với tài liệu.
- Cho phép Admin giám sát người dùng, tài liệu, lỗi xử lý và trạng thái hệ thống.
- Tăng tính bảo mật và khả năng vận hành thông qua WAF, CloudWatch, Secrets Manager và IAM Role.