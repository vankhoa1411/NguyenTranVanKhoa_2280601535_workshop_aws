---

title : "Kiểm tra Full Workflow"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.13.3. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ kiểm tra toàn bộ **full workflow** của hệ thống **DocuMind AI** sau khi đã deploy backend và cấu hình production.

Full workflow bao gồm quá trình người dùng đăng nhập, upload tài liệu, backend lưu file lên Amazon S3, gửi message vào Amazon SQS, worker xử lý OCR bằng Amazon Textract, AI Gateway phân tích bằng Gemini/OpenAI, lưu kết quả vào PostgreSQL và frontend hiển thị kết quả trên Dashboard.

Đây là bước xác nhận toàn bộ hệ thống hoạt động end-to-end.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Kiểm tra authentication hoạt động.
* Kiểm tra upload tài liệu từ frontend hoặc Postman.
* Xác nhận file được lưu vào Amazon S3.
* Xác nhận message được gửi vào Amazon SQS.
* Xác nhận worker nhận message.
* Xác nhận Amazon Textract OCR thành công.
* Xác nhận Gemini/OpenAI phân tích tài liệu.
* Xác nhận kết quả được lưu vào PostgreSQL.
* Xác nhận Dashboard hiển thị trạng thái và kết quả.
* Kiểm tra notification, audit log và security log nếu có.

---

#### Kiến trúc full workflow

```text id="full-workflow"
User
  |
  v
Frontend Dashboard
  |
  v
Backend API on EC2
  |
  |-- Upload file --> Amazon S3
  |-- Save metadata --> PostgreSQL + Prisma
  |-- Send job --> Amazon SQS
                         |
                         v
                    SQS Worker
                         |
                         v
                  Amazon Textract OCR
                         |
                         v
                    AI Gateway
                         |
          |--------------|--------------|
          v                             v
      OpenAI API                  Gemini API
          |--------------|--------------|
                         v
               Save AIAnalysis
                         |
                         v
                  Frontend Dashboard
```

![test-full-workflow](/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.3-test-full-workflow/test-full-workflow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình Dashboard sau khi tài liệu xử lý thành công, S3 object đã upload, SQS message đã được xử lý và log worker hiển thị OCR/AI completed.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.3-test-full-workflow/test-full-workflow.png`

---

#### Điều kiện trước khi test

Trước khi test full workflow, hãy đảm bảo:

* Backend API đang chạy trên EC2.
* Worker đang chạy trên EC2 hoặc server riêng.
* PostgreSQL đang hoạt động.
* Prisma migration đã chạy.
* S3 bucket đã được tạo.
* SQS queue đã được tạo.
* IAM Role đã gắn đúng policy.
* Textract có quyền đọc file từ S3.
* Gemini/OpenAI API key hoạt động.
* Frontend đã trỏ đúng backend URL.
* CloudWatch Logs đã cấu hình nếu dùng.

---

#### Bước 1: Kiểm tra backend health

Gọi:

```text id="health-url"
GET http://your-ec2-public-ip:3000/api/health
```

Response mong đợi:

```json id="health-ok"
{
  "success": true,
  "message": "DocuMind AI backend is running"
}
```

Nếu có database health:

```text id="db-health-url"
GET http://your-ec2-public-ip:3000/api/health/database
```

Response mong đợi:

```json id="db-health-ok"
{
  "success": true,
  "message": "Database connection is healthy"
}
```

---

#### Bước 2: Kiểm tra đăng nhập

Gọi API login hoặc đăng nhập từ frontend:

```text id="login-api"
POST /api/auth/login
```

Body:

```json id="login-body"
{
  "email": "demo@docmind.ai",
  "password": "your-password"
}
```

Response mong đợi:

```json id="login-response"
{
  "success": true,
  "accessToken": "jwt-token",
  "user": {
    "email": "demo@docmind.ai",
    "role": "USER"
  }
}
```

---

#### Bước 3: Upload tài liệu

Từ frontend hoặc Postman:

```text id="upload-api"
POST /api/documents/upload
```

Body dạng:

```text id="upload-form"
multipart/form-data
file: invoice-demo.pdf
```

Header:

```text id="auth-header"
Authorization: Bearer your_jwt_token
```

Response mong đợi:

```json id="upload-response"
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED"
  }
}
```

---

#### Bước 4: Kiểm tra file trên S3

Vào AWS Console:

```text id="s3-check"
Amazon S3
→ documind-document-storage
→ uploads/
```

Kiểm tra file đã xuất hiện.

Hoặc dùng CLI:

```bash id="s3-ls-upload"
aws s3 ls s3://documind-document-storage/uploads/ --recursive
```

---

#### Bước 5: Kiểm tra message trong SQS

Vào:

```text id="sqs-check"
Amazon SQS
→ docmind-document-processing-queue
```

Kiểm tra message count.

Nếu worker đang chạy, message có thể được xử lý rất nhanh và không còn trong queue. Khi đó kiểm tra log worker.

---

#### Bước 6: Kiểm tra worker log

Trên EC2:

```bash id="pm2-worker-log"
pm2 logs documind-worker
```

Log mong đợi:

```text id="worker-log-expected"
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[OCR_RESULT_SAVED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[AI_ANALYSIS_SAVED]
[DOCUMENT_STATUS_COMPLETED]
[SQS_MESSAGE_DELETED]
```

---

#### Bước 7: Kiểm tra PostgreSQL

Mở Prisma Studio nếu đang test local/server:

```bash id="prisma-studio"
npx prisma studio
```

Kiểm tra bảng:

```text id="tables-check"
Document
OCRResult
AIAnalysis
Notification
AuditLog
```

Trạng thái document mong đợi:

```text id="completed-status"
COMPLETED
```

---

#### Bước 8: Kiểm tra Dashboard

Mở frontend Dashboard:

1. Đăng nhập.
2. Vào Document Dashboard.
3. Kiểm tra tài liệu vừa upload.
4. Kiểm tra status là `COMPLETED`.
5. Mở Document Detail.
6. Xem OCR text.
7. Xem AI Analysis.
8. Test AI Chat/RAG nếu có.

---

#### Bước 9: Kiểm tra Notification

Kiểm tra notification center:

```text id="notification-check"
Document uploaded successfully.
AI analysis completed.
```

Nếu xử lý lỗi, cần có notification:

```text id="notification-failed"
Document processing failed.
```

---

#### Bước 10: Kiểm tra Audit Log

Admin mở Audit Log và kiểm tra các action:

```text id="audit-actions"
USER_LOGIN
DOCUMENT_UPLOADED
OCR_COMPLETED
AI_ANALYSIS_COMPLETED
```

Nếu có lỗi:

```text id="audit-error-actions"
DOCUMENT_PROCESSING_FAILED
AI_PROVIDER_FALLBACK_USED
SECURITY_FORBIDDEN_ACCESS
```

---

#### Bước 11: Kiểm tra CloudWatch Logs

Vào:

```text id="cw-check"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Kiểm tra log backend và worker.

---

#### Bước 12: Test AI Chat/RAG

Gọi:

```text id="chat-api"
POST /api/ai/chat
```

Body:

```json id="chat-body"
{
  "documentId": "doc_001",
  "question": "Summarize this document.",
  "provider": "openai"
}
```

Response mong đợi:

```json id="chat-response"
{
  "success": true,
  "answer": "This document is an invoice...",
  "provider": "openai",
  "model": "gpt-4.1-mini"
}
```

---

#### Bảng kiểm tra full workflow

| Bước           | Kết quả mong đợi                  |
| -------------- | --------------------------------- |
| Backend health | API trả success                   |
| Login          | Nhận JWT token                    |
| Upload         | File upload thành công            |
| S3             | Object xuất hiện trong bucket     |
| SQS            | Message được gửi hoặc worker nhận |
| Worker         | Log xử lý đầy đủ                  |
| Textract       | OCR text được tạo                 |
| AI Gateway     | Gemini/OpenAI trả kết quả         |
| PostgreSQL     | Document/OCR/AIAnalysis được lưu  |
| Dashboard      | Hiển thị status và result         |
| Notification   | Có thông báo thành công           |
| Audit Log      | Có log hành động                  |

---

#### Các lỗi thường gặp

| Lỗi                       | Nguyên nhân                         | Cách xử lý                     |
| ------------------------- | ----------------------------------- | ------------------------------ |
| Backend health fail       | Server chưa chạy hoặc port chưa mở  | Kiểm tra PM2 và Security Group |
| Login fail                | Sai user/password/JWT config        | Kiểm tra Auth API              |
| Upload fail               | CORS, file type hoặc S3 permission  | Kiểm tra backend và IAM        |
| S3 không có file          | Upload service lỗi                  | Kiểm tra S3 log                |
| SQS không có message      | Backend chưa gửi message            | Kiểm tra SQS service           |
| Worker không nhận message | Worker chưa chạy hoặc Queue URL sai | Kiểm tra PM2 và `.env`         |
| Textract lỗi              | File sai định dạng hoặc thiếu quyền | Kiểm tra file và IAM           |
| AI lỗi                    | API key sai hoặc quota              | Kiểm tra Gemini/OpenAI config  |
| Status không đổi          | Worker chưa update database         | Kiểm tra Prisma update         |
| Dashboard không cập nhật  | Frontend chưa polling               | Kiểm tra status API            |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Backend API hoạt động trên EC2.
* Worker chạy ổn định.
* User đăng nhập được.
* Upload tài liệu thành công.
* File xuất hiện trong S3.
* Message được gửi vào SQS.
* Worker nhận và xử lý message.
* Textract OCR thành công.
* AI Gateway phân tích bằng Gemini/OpenAI thành công.
* Kết quả lưu vào PostgreSQL.
* Dashboard hiển thị kết quả.
* Notification và Audit Log hoạt động.
* CloudWatch có log cần thiết.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã được kiểm tra end-to-end. Hệ thống có thể xử lý tài liệu từ lúc người dùng upload đến khi hiển thị kết quả OCR, AI Analysis và AI Chat trên Dashboard.
