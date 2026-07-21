---

title : "Deployment và Test"
date : 2026-06-10
weight : 13
chapter : false
pre : " <b> 5.13. </b> "
------------------------

#### Triển khai và kiểm thử hệ thống

Trong phần này, chúng ta sẽ thực hiện bước **Deployment và Test** cho hệ thống **DocuMind AI**.

Sau khi đã hoàn thành các phần chính như Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, PostgreSQL + Prisma, Backend API, Worker, Frontend Dashboard, Notification, Admin Panel, IAM Role, Monitoring và Security, bước cuối cùng là triển khai backend lên môi trường server và kiểm tra toàn bộ workflow.

Mục tiêu của phần này là đảm bảo hệ thống có thể chạy ổn định trên môi trường gần với production, thay vì chỉ chạy local.

---

#### Vai trò của Deployment trong DocuMind AI

Deployment giúp đưa hệ thống từ môi trường phát triển local lên môi trường server thật.

Trong DocuMind AI, backend và worker cần được deploy để:

* Nhận request từ frontend.
* Xử lý đăng nhập và phân quyền người dùng.
* Upload tài liệu lên Amazon S3.
* Gửi message xử lý tài liệu vào Amazon SQS.
* Chạy worker để nhận message từ SQS.
* Gọi Amazon Textract để OCR tài liệu.
* Gọi Gemini/OpenAI để phân tích tài liệu.
* Lưu kết quả vào PostgreSQL.
* Trả kết quả về Dashboard cho người dùng.

---

#### Vai trò của Testing trong DocuMind AI

Testing giúp kiểm tra toàn bộ hệ thống có hoạt động đúng hay không.

Các phần cần test gồm:

* Backend API health check.
* Authentication.
* Upload tài liệu.
* S3 object storage.
* SQS message queue.
* Worker processing.
* Textract OCR.
* AI Analysis bằng Gemini/OpenAI.
* PostgreSQL data persistence.
* Document Dashboard.
* Notification.
* Audit Log.
* Admin Panel.
* CloudWatch Logs.
* IAM Permission.
* Các lỗi thường gặp.

---

#### Kiến trúc Deployment và Test

Luồng tổng quan sau khi deploy:

```text id="deployment-test-flow"
Frontend Dashboard
  |
  v
Backend API on EC2
  |
  |-- Upload file --> Amazon S3
  |-- Send job --> Amazon SQS
  |-- Save metadata --> PostgreSQL + Prisma
  |
  v
SQS Worker on EC2
  |
  |-- Read file from S3
  |-- OCR using Amazon Textract
  |-- Analyze using Gemini/OpenAI
  |-- Save OCRResult + AIAnalysis
  |
  v
Frontend shows completed result
```

![deployment-test-flow](/static/images/5-Workshop/5.13-Deployment-and-Test/deployment-test-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa Frontend gọi Backend API trên EC2, backend upload tài liệu lên S3, gửi message vào SQS, worker xử lý Textract OCR, gọi Gemini/OpenAI và lưu kết quả vào PostgreSQL.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/deployment-test-flow.png`

---

#### Nội dung chi tiết

Trong phần **5.13 - Deployment and Test**, chúng ta sẽ thực hiện các bước sau:

* [Deploy Backend lên EC2](5.13.1-deploy-backend-ec2/)
* [Cấu hình môi trường Production](5.13.2-configure-env-production/)
* [Kiểm tra Full Workflow](5.13.3-test-full-workflow/)
* [Các lỗi thường gặp](5.13.4-common-errors/)

---

#### Thành phần được deploy

Trong workshop này, thành phần chính cần deploy là:

```text id="deploy-components"
Backend API
SQS Worker
```

Backend API chịu trách nhiệm:

* Auth API.
* Document Upload API.
* Document Status API.
* OCR Result API.
* AI Analysis API.
* Notification API.
* Admin API.

SQS Worker chịu trách nhiệm:

* Nhận message từ SQS.
* Đọc file từ S3.
* Gọi Textract OCR.
* Gọi AI Gateway.
* Lưu kết quả xử lý.
* Cập nhật document status.
* Tạo notification và audit log.

---

#### Công nghệ triển khai

Các công nghệ sử dụng trong phần deployment:

| Công nghệ       | Vai trò                           |
| --------------- | --------------------------------- |
| Amazon EC2      | Server chạy backend và worker     |
| Ubuntu Server   | Hệ điều hành cho EC2              |
| Node.js         | Runtime cho backend               |
| PM2             | Quản lý process backend và worker |
| Git             | Clone source code                 |
| PostgreSQL      | Database chính                    |
| Prisma          | Migration và database client      |
| AWS IAM Role    | Cấp quyền AWS cho EC2             |
| CloudWatch Logs | Theo dõi log production           |

---

#### Cấu hình production cần chuẩn bị

Các nhóm biến môi trường cần có:

```text id="production-config-groups"
App Config
Database Config
AWS Config
AI Provider Config
Auth Config
CORS Config
CloudWatch Config
Secrets Manager Config
```

Ví dụ file `.env.production`:

```env id="sample-env-production"
NODE_ENV="production"
PORT=3000

DATABASE_URL="postgresql://docmind:docmind_password@your-db-host:5432/docmind"

AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"

AI_PROVIDER="openai"

OPENAI_API_KEY="your-openai-api-key"
OPENAI_MODEL="gpt-4.1-mini"

GEMINI_API_KEY="your-gemini-api-key"
GEMINI_MODEL="gemini-1.5-flash"

JWT_SECRET="your-strong-jwt-secret"

CORS_ORIGIN="https://your-frontend-domain.com"

CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-production"
CLOUDWATCH_WORKER_LOG_STREAM="worker-production"
```

Nếu dùng AWS Secrets Manager, production có thể chỉ cần:

```env id="sample-secrets-env"
NODE_ENV="production"
PORT=3000

USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

---

#### Lưu ý về PostgreSQL

Trong workshop này, DocuMind AI sử dụng:

```text id="postgres-prisma"
PostgreSQL + Prisma
```

Không sử dụng Amazon RDS trong phần này.

PostgreSQL có thể chạy:

* Trên cùng EC2 với backend cho demo.
* Trên một server riêng.
* Trên dịch vụ managed database khác nếu sau này mở rộng.

Điều quan trọng là `DATABASE_URL` phải đúng và backend có thể kết nối được.

---

#### Lưu ý về IAM Role

Khi backend chạy trên EC2, nên gắn IAM Role đã tạo ở phần 5.11 vào EC2.

IAM Role cần có quyền:

```text id="iam-needed"
Amazon S3
Amazon SQS
Amazon Textract
CloudWatch Logs
Secrets Manager
```

Khi dùng IAM Role, production không cần hardcode:

```env id="bad-aws-keys"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Kiểm tra IAM Role bằng lệnh:

```bash id="sts-check"
aws sts get-caller-identity
```

Kết quả mong đợi có dạng:

```text id="assumed-role"
assumed-role/DocuMindBackendWorkerRole
```

---

#### Chạy backend và worker bằng PM2

Sau khi build backend, có thể chạy bằng PM2.

Backend API:

```bash id="pm2-backend"
pm2 start dist/server.js --name documind-backend
```

Worker:

```bash id="pm2-worker"
pm2 start dist/workers/document.worker.js --name documind-worker
```

Kiểm tra process:

```bash id="pm2-list"
pm2 list
```

Xem log:

```bash id="pm2-logs"
pm2 logs documind-backend
pm2 logs documind-worker
```

Lưu process để tự chạy lại sau reboot:

```bash id="pm2-save"
pm2 startup
pm2 save
```

---

#### Quy trình kiểm thử full workflow

Sau khi deploy, cần test toàn bộ workflow:

```text id="full-workflow-steps"
1. Backend health check
2. Login user
3. Upload document
4. Check S3 object
5. Check SQS message
6. Check worker logs
7. Check Textract OCR result
8. Check AI Analysis result
9. Check PostgreSQL data
10. Check Dashboard status
11. Check Notification
12. Check Audit Log
13. Check CloudWatch Logs
```

---

#### Status cần kiểm tra

Trong quá trình test, document status nên đi theo luồng:

```text id="status-flow"
PENDING
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
```

Nếu lỗi:

```text id="failed-status"
FAILED
```

Frontend Dashboard cần hiển thị đúng trạng thái này để người dùng biết tài liệu đang ở bước nào.

---

#### Health Check API

Backend nên có API health check:

```text id="health-api"
GET /api/health
```

Response mong đợi:

```json id="health-response"
{
  "success": true,
  "message": "DocuMind AI backend is running"
}
```

Nếu có kiểm tra database:

```text id="database-health-api"
GET /api/health/database
```

Response mong đợi:

```json id="database-health-response"
{
  "success": true,
  "message": "Database connection is healthy"
}
```

---

#### Kiểm tra dữ liệu trong PostgreSQL

Sau khi full workflow chạy thành công, các bảng cần có dữ liệu:

| Bảng           | Dữ liệu mong đợi                |
| -------------- | ------------------------------- |
| `User`         | Thông tin người dùng            |
| `Document`     | Metadata tài liệu và status     |
| `OCRResult`    | Text OCR từ Textract            |
| `AIAnalysis`   | Kết quả phân tích Gemini/OpenAI |
| `Notification` | Thông báo user/admin            |
| `AuditLog`     | Lịch sử hành động               |

Có thể dùng Prisma Studio để kiểm tra:

```bash id="prisma-studio"
npx prisma studio
```

---

#### Kiểm tra log production

Các log quan trọng cần thấy:

```text id="production-logs"
[APP_STARTED]
[DATABASE_CONNECTED]
[SQS_WORKER_STARTED]
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_MESSAGE_RECEIVED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_STATUS_COMPLETED]
```

Nếu có lỗi:

```text id="error-logs"
[DOCUMENT_PROCESSING_FAILED]
[TEXTRACT_OCR_FAILED]
[AI_PROVIDER_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[SECURITY_FORBIDDEN_ACCESS]
```

---

#### Các lỗi cần chuẩn bị xử lý

Trong quá trình deploy và test, có thể gặp lỗi ở các nhóm:

| Nhóm lỗi    | Ví dụ                                         |
| ----------- | --------------------------------------------- |
| EC2         | Không SSH được, port chưa mở                  |
| Node.js     | Build fail, thiếu dependency                  |
| PM2         | Backend/worker crash                          |
| PostgreSQL  | Sai `DATABASE_URL`, migration lỗi             |
| Prisma      | Chưa generate client, thiếu table             |
| S3          | `AccessDenied`, sai bucket                    |
| SQS         | Sai Queue URL, worker không nhận message      |
| Textract    | File không hỗ trợ, `InvalidS3ObjectException` |
| AI Provider | API key sai, quota, model sai                 |
| IAM         | Thiếu permission                              |
| CORS        | Frontend không gọi được backend               |
| CloudWatch  | Không ghi được log                            |

Các lỗi chi tiết sẽ được tổng hợp ở phần **5.13.4 - Các lỗi thường gặp**.

---

#### Checklist trước khi hoàn thành phần 5.13

Bạn đã hoàn thành phần này khi:

* Backend API đã chạy trên EC2.
* SQS Worker đã chạy trên EC2.
* `.env.production` hoặc Secrets Manager đã được cấu hình.
* PostgreSQL kết nối thành công.
* Prisma migration production đã chạy.
* EC2 đã gắn IAM Role đúng.
* Backend health check hoạt động.
* User đăng nhập được.
* Upload tài liệu thành công.
* File xuất hiện trong S3.
* Message được gửi vào SQS.
* Worker nhận và xử lý message.
* Textract OCR hoàn tất.
* Gemini/OpenAI phân tích thành công.
* Kết quả lưu vào PostgreSQL.
* Dashboard hiển thị status `COMPLETED`.
* Notification và Audit Log hoạt động.
* CloudWatch có log production.

---

#### Kết quả kỳ vọng

Sau phần này, DocuMind AI đã được deploy và kiểm thử end-to-end. Backend API và Worker có thể chạy trên EC2, kết nối đầy đủ với PostgreSQL, Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI và hiển thị kết quả xử lý tài liệu trên Dashboard.

Đây là bước xác nhận hệ thống đã sẵn sàng để demo, báo cáo hoặc tiếp tục mở rộng lên môi trường production hoàn chỉnh hơn.
