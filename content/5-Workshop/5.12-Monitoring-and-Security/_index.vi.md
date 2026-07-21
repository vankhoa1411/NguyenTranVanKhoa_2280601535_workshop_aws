---

title : "Monitoring và Security"
date : 2026-06-10
weight : 12
chapter : false
pre : " <b> 5.12. </b> "
------------------------

#### Xây dựng Monitoring và Security

Trong phần này, chúng ta sẽ cấu hình các thành phần **Monitoring** và **Security** cho hệ thống **DocuMind AI**.

Sau khi hệ thống đã có pipeline xử lý tài liệu gồm Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, PostgreSQL + Prisma, Backend API, Worker, Notification và Admin Panel, bước tiếp theo là bổ sung khả năng giám sát và bảo vệ hệ thống.

Monitoring giúp developer và Admin theo dõi hoạt động của backend, worker, queue, AI provider và các lỗi xử lý tài liệu. Security giúp bảo vệ API, quản lý secret an toàn, kiểm tra truy cập trái quyền và giảm rủi ro từ các request độc hại.

---

#### Vai trò của Monitoring trong DocuMind AI

Trong hệ thống **DocuMind AI**, Monitoring được sử dụng để:

* Theo dõi log backend.
* Theo dõi log worker.
* Kiểm tra quá trình upload tài liệu.
* Kiểm tra message gửi vào Amazon SQS.
* Theo dõi quá trình OCR bằng Amazon Textract.
* Theo dõi quá trình AI Analysis bằng Gemini/OpenAI.
* Ghi nhận lỗi xử lý tài liệu.
* Theo dõi AI provider fallback.
* Hỗ trợ Admin kiểm tra lỗi hệ thống.
* Chuẩn bị dữ liệu cho Admin Monitoring Dashboard.

Các log quan trọng có thể được ghi vào **Amazon CloudWatch Logs** để dễ dàng tìm kiếm và kiểm tra khi hệ thống gặp lỗi.

---

#### Vai trò của Security trong DocuMind AI

Security giúp hệ thống bảo vệ dữ liệu và kiểm soát truy cập tốt hơn.

Trong DocuMind AI, Security tập trung vào:

* Không hardcode API key trong source code.
* Lưu secret bằng AWS Secrets Manager.
* Không public tài liệu trong Amazon S3.
* Chỉ user sở hữu tài liệu mới được xem tài liệu đó.
* Admin API bắt buộc kiểm tra role `ADMIN`.
* Ghi Audit Log cho các hành động quan trọng.
* Ghi nhận truy cập trái quyền.
* Dùng AWS WAF để giảm rủi ro từ SQL Injection, XSS hoặc request bất thường.
* Không log password, JWT token, API key hoặc nội dung tài liệu nhạy cảm.

---

#### Kiến trúc Monitoring và Security

Luồng tổng quan:

```text id="kotk24"
Frontend / User Request
  |
  v
AWS WAF
  |
  v
Backend API / Worker
  |
  |-- Write logs --> CloudWatch Logs
  |-- Read secrets --> AWS Secrets Manager
  |-- Create Audit Log --> PostgreSQL + Prisma
  |-- Create Admin Notification --> PostgreSQL + Prisma
  |
  v
Admin Panel / Monitoring Dashboard
```

![monitoring-security-flow](/images/5-Workshop/5.12-Monitoring-and-Security/monitoring-security-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa AWS WAF bảo vệ request đi vào hệ thống, Backend/Worker ghi log lên CloudWatch, đọc secret từ Secrets Manager và lưu Audit Log/Admin Notification vào PostgreSQL + Prisma.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/monitoring-security-flow.png`

---

#### Nội dung chi tiết

Trong phần **5.12 - Monitoring and Security**, chúng ta sẽ thực hiện các bước sau:

* [CloudWatch Logs](5.12.1-cloudwatch-logs/)
* [Secrets Manager](5.12.2-secrets-manager/)
* [WAF Protection](5.12.3-waf-protection/)
* [Kiểm tra Security Logs](5.12.4-test-security-logs/)

---

#### CloudWatch Logs

**Amazon CloudWatch Logs** được dùng để lưu log từ backend và worker.

Log group đề xuất:

```text id="4967r7"
/docmind/application
```

Các log quan trọng:

```text id="zek0rw"
[APP_STARTED]
[DATABASE_CONNECTED]
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_PROCESSING_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[SECURITY_FORBIDDEN_ACCESS]
```

CloudWatch giúp developer và Admin dễ kiểm tra lỗi khi pipeline xử lý tài liệu không hoạt động đúng.

---

#### Secrets Manager

**AWS Secrets Manager** được dùng để lưu thông tin nhạy cảm của hệ thống.

Các secret có thể lưu:

```text id="4t8fyv"
DATABASE_URL
JWT_SECRET
OPENAI_API_KEY
GEMINI_API_KEY
AI_PROVIDER
AWS_SQS_QUEUE_URL
```

Secret name đề xuất:

```text id="74ey09"
docmind/backend
```

Khi chạy production, backend chỉ cần cấu hình:

```env id="0enioh"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

Không nên lưu trực tiếp các key nhạy cảm trong `.env` production.

---

#### AWS WAF

**AWS WAF** giúp bảo vệ ứng dụng web khỏi các request bất thường hoặc độc hại.

WAF có thể được gắn vào:

```text id="bffnkx"
Amazon CloudFront
Application Load Balancer
API Gateway
```

Trong DocuMind AI, WAF có thể giúp giảm rủi ro từ:

```text id="k2yjp2"
SQL Injection
Cross-Site Scripting
Bad bot traffic
Known bad inputs
Suspicious request patterns
```

Các rule group đề xuất:

```text id="jew7uq"
AWSManagedRulesCommonRuleSet
AWSManagedRulesKnownBadInputsRuleSet
AWSManagedRulesSQLiRuleSet
AmazonIpReputationList
```

Khi mới cấu hình, nên để rule ở chế độ `Count` trước, sau đó kiểm tra request hợp lệ rồi mới chuyển sang `Block`.

---

#### Security Logs

Security Logs dùng để ghi nhận các sự kiện bảo mật quan trọng như:

```text id="wl8s4p"
LOGIN_FAILED
UNAUTHORIZED_REQUEST
SECURITY_FORBIDDEN_ACCESS
UNSUPPORTED_FILE_UPLOAD
FILE_SIZE_EXCEEDED
AI_PROVIDER_FALLBACK_USED
DOCUMENT_PROCESSING_FAILED
IAM_ACCESS_DENIED
WAF_REQUEST_COUNTED
WAF_REQUEST_BLOCKED
```

Các security event này nên được ghi vào:

* CloudWatch Logs.
* AuditLog trong PostgreSQL.
* Admin Notification nếu là lỗi quan trọng.

Ví dụ khi user truy cập tài liệu không thuộc quyền:

```text id="8q4oap"
[SECURITY_FORBIDDEN_ACCESS]
```

Audit Log nên lưu:

```json id="ygyh6k"
{
  "action": "SECURITY_FORBIDDEN_ACCESS",
  "target": "Document",
  "severity": "warning",
  "metadata": {
    "documentId": "doc_001"
  }
}
```

---

#### Các thành phần liên quan

| Thành phần         | Vai trò                                        |
| ------------------ | ---------------------------------------------- |
| CloudWatch Logs    | Lưu log backend, worker và security event      |
| Secrets Manager    | Lưu API key, database URL và secret            |
| AWS WAF            | Bảo vệ request đi vào ứng dụng                 |
| AuditLog           | Lưu lịch sử hành động quan trọng               |
| Admin Notification | Cảnh báo Admin khi có lỗi hoặc sự kiện bảo mật |
| IAM Role           | Cấp quyền an toàn cho backend/worker           |
| Admin Panel        | Hiển thị log, alert và trạng thái hệ thống     |

---

#### Quy tắc không log dữ liệu nhạy cảm

Khi xây dựng monitoring, cần đảm bảo không log các thông tin sau:

```text id="fu1xam"
password
JWT token
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
DATABASE_URL có password
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
full document content
```

Chỉ nên log metadata cần thiết:

```text id="9vr2jw"
documentId
userId
fileName
status
provider
model
errorCode
requestId
```

Ví dụ log an toàn:

```json id="jv6z99"
{
  "level": "info",
  "event": "AI_ANALYSIS_COMPLETED",
  "documentId": "doc_001",
  "provider": "openai",
  "model": "gpt-4.1-mini",
  "timestamp": "2026-06-10T10:00:00.000Z"
}
```

---

#### Monitoring trong Admin Panel

Admin Panel có thể hiển thị các thông tin monitoring:

```text id="g4r0bv"
Total Requests
Upload Success
Processing Failed
AI Provider Fallback Count
Unread Admin Alerts
WAF Counted Requests
WAF Blocked Requests
Recent Security Events
```

Nếu chưa tích hợp dashboard realtime, Admin có thể kiểm tra thông qua:

* CloudWatch Logs.
* Admin Notifications.
* Audit Logs.
* Processing Errors.

---

#### Test end-to-end Monitoring và Security

Các bước kiểm tra:

1. Start backend.
2. Start worker.
3. Upload tài liệu hợp lệ.
4. Kiểm tra log upload trong CloudWatch.
5. Kiểm tra log worker xử lý SQS.
6. Kiểm tra log Textract OCR.
7. Kiểm tra log AI Analysis.
8. Test login sai mật khẩu.
9. Test truy cập document không thuộc quyền.
10. Test upload file sai định dạng.
11. Test AI provider fallback.
12. Test WAF với SQLi/XSS payload.
13. Kiểm tra Admin Notification và Audit Log.
14. Đảm bảo log không chứa secret hoặc token.

---

#### Các lỗi thường gặp

| Lỗi                               | Nguyên nhân                                    | Cách xử lý                              |
| --------------------------------- | ---------------------------------------------- | --------------------------------------- |
| Không thấy CloudWatch log         | Chưa cấu hình logger hoặc thiếu IAM permission | Kiểm tra CloudWatch policy              |
| Không đọc được secret             | Sai secret name hoặc thiếu quyền               | Kiểm tra Secrets Manager và IAM         |
| WAF không hoạt động               | Chưa attach vào CloudFront/ALB                 | Kiểm tra Associated resources           |
| Security event không được ghi log | Middleware chưa gọi logger/audit service       | Kiểm tra auth/authorization middleware  |
| Admin không thấy alert            | Chưa tạo Admin Notification                    | Kiểm tra notification service           |
| Log chứa API key/token            | Logger ghi toàn bộ request/env                 | Mask hoặc loại bỏ dữ liệu nhạy cảm      |
| WAF chặn nhầm request hợp lệ      | Rule quá strict                                | Chuyển về Count mode hoặc tạo exception |

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được:

* CloudWatch Log Group đã được tạo.
* Backend và worker ghi được log lên CloudWatch.
* Secrets Manager lưu được các secret quan trọng.
* Backend có thể đọc secret khi chạy production.
* AWS WAF được cấu hình để bảo vệ request.
* WAF metrics được bật.
* Security events được ghi nhận.
* Audit Log có các sự kiện bảo mật.
* Admin Notification được tạo khi có lỗi quan trọng.
* Không có secret, token hoặc password trong log.

---

#### Kết quả kỳ vọng

Sau phần này, DocuMind AI đã có lớp monitoring và security cơ bản để phục vụ vận hành thực tế. Hệ thống có thể ghi log tập trung, quản lý secret an toàn, phát hiện request bất thường bằng WAF và cung cấp dữ liệu cho Admin kiểm tra lỗi hoặc sự kiện bảo mật.
