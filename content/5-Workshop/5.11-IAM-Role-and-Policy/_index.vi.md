---

title : "IAM Role và Policy"
date : 2026-06-10
weight : 11
chapter : false
pre : " <b> 5.11. </b> "
------------------------

#### Cấu hình IAM Role và Policy

Trong phần này, chúng ta sẽ cấu hình **IAM Role** và **IAM Policy** cho hệ thống **DocuMind AI**.

IAM là viết tắt của **Identity and Access Management**. Đây là dịch vụ của AWS dùng để quản lý quyền truy cập vào các tài nguyên AWS. Trong dự án DocuMind AI, backend và worker cần truy cập nhiều dịch vụ AWS như **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Amazon CloudWatch** và **AWS Secrets Manager**.

Thay vì hardcode `AWS_ACCESS_KEY_ID` và `AWS_SECRET_ACCESS_KEY` trong source code hoặc file `.env` production, hệ thống nên sử dụng **IAM Role**. Khi backend/worker chạy trên EC2 hoặc môi trường AWS có gắn role phù hợp, AWS SDK có thể tự lấy quyền tạm thời từ IAM Role để gọi các dịch vụ AWS một cách an toàn hơn.

---

#### Vai trò của IAM trong DocuMind AI

Trong hệ thống **DocuMind AI**, IAM Role và Policy được sử dụng để cấp quyền cho backend/worker thực hiện các tác vụ sau:

* Upload tài liệu lên Amazon S3.
* Đọc tài liệu từ Amazon S3 để xử lý OCR.
* Gửi message xử lý tài liệu vào Amazon SQS.
* Nhận và xóa message từ Amazon SQS.
* Gọi Amazon Textract để trích xuất văn bản từ tài liệu.
* Ghi log backend/worker lên CloudWatch.
* Đọc secret từ AWS Secrets Manager.
* Giảm rủi ro lộ AWS credentials trong production.

---

#### Kiến trúc IAM Role và Policy

Luồng quyền trong hệ thống:

```text
Backend / Worker / EC2
  |
  v
IAM Role
  |
  |-- S3 Policy
  |-- SQS + Textract Policy
  |-- CloudWatch + Secrets Manager Policy
  |
  v
AWS Services
```

![iam-role-policy-flow](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/iam-role-policy-flow.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ sơ đồ minh họa Backend/Worker sử dụng IAM Role để truy cập Amazon S3, Amazon SQS, Amazon Textract, CloudWatch Logs và AWS Secrets Manager.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/iam-role-policy-flow.png`

---

#### Vì sao cần IAM Role thay vì Access Key?

Trong môi trường local, bạn có thể dùng AWS CLI hoặc AWS credentials để test nhanh. Tuy nhiên, khi deploy production, không nên lưu trực tiếp AWS access key trong `.env`.

Không nên dùng trong production:

```env
AWS_ACCESS_KEY_ID="your-access-key"
AWS_SECRET_ACCESS_KEY="your-secret-key"
```

Lý do:

* Dễ bị lộ nếu commit nhầm lên GitHub.
* Khó quản lý khi nhiều môi trường cùng dùng chung key.
* Khó rotate key định kỳ.
* Không phù hợp với nguyên tắc bảo mật least privilege.
* Nếu key bị lộ, attacker có thể truy cập tài nguyên AWS.

Thay vào đó, nên dùng:

```text
IAM Role attached to EC2 / Backend Server
```

Khi đó, backend chỉ cần cấu hình region, bucket name và queue URL. AWS SDK sẽ tự lấy credentials từ IAM Role.

---

#### Nguyên tắc Least Privilege

Khi tạo IAM Policy, nên áp dụng nguyên tắc **least privilege**, nghĩa là chỉ cấp đúng quyền cần thiết cho hệ thống.

Ví dụ:

Backend cần upload file lên S3 thì cấp:

```text
s3:PutObject
```

Worker cần đọc file từ S3 thì cấp:

```text
s3:GetObject
```

Worker cần nhận message từ SQS thì cấp:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
```

Không nên cấp quyền quá rộng như:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

hoặc:

```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

Các policy quá rộng có thể gây rủi ro bảo mật nếu backend hoặc server bị tấn công.

---

#### Nội dung chi tiết

Trong phần **5.11 - IAM Role and Policy**, chúng ta sẽ thực hiện các bước sau:

* [Tạo IAM Role](5.11.1-create-iam-role/)
* [Tạo S3 Policy](5.11.2-create-s3-policy/)
* [Tạo SQS và Textract Policy](5.11.3-create-sqs-textract-policy/)
* [Tạo CloudWatch và Secrets Manager Policy](5.11.4-create-cloudwatch-secrets-policy/)
* [Gắn IAM Role vào Backend](5.11.5-attach-role-to-backend/)
* [Kiểm tra IAM Permission](5.11.6-test-iam-permission/)

---

#### Các Policy cần tạo

Trong workshop này, chúng ta sẽ chia quyền thành nhiều policy riêng để dễ quản lý.

| Policy                            | Mục đích                                                     |
| --------------------------------- | ------------------------------------------------------------ |
| `DocuMindS3AccessPolicy`          | Cho phép backend/worker upload và đọc tài liệu trong S3      |
| `DocuMindSQSTextractPolicy`       | Cho phép gửi/nhận message SQS và gọi Amazon Textract         |
| `DocuMindCloudWatchSecretsPolicy` | Cho phép ghi log CloudWatch và đọc secret từ Secrets Manager |

Các policy này sẽ được attach vào IAM Role:

```text
DocuMindBackendWorkerRole
```

---

#### Quyền S3 cần thiết

Backend và worker cần quyền với Amazon S3:

```text
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

Ý nghĩa:

| Quyền             | Mục đích                               |
| ----------------- | -------------------------------------- |
| `s3:PutObject`    | Backend upload tài liệu lên S3         |
| `s3:GetObject`    | Worker đọc tài liệu để OCR             |
| `s3:DeleteObject` | Xóa object khi user/admin xóa tài liệu |
| `s3:ListBucket`   | Kiểm tra object trong bucket           |

Resource nên giới hạn theo bucket của dự án:

```text
arn:aws:s3:::documind-document-storage
arn:aws:s3:::documind-document-storage/*
```

---

#### Quyền SQS cần thiết

Backend cần quyền gửi message:

```text
sqs:SendMessage
```

Worker cần quyền xử lý message:

```text
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
sqs:ChangeMessageVisibility
```

Nếu có Dead Letter Queue, có thể cấp quyền đọc trạng thái DLQ để Admin hoặc worker kiểm tra lỗi.

---

#### Quyền Textract cần thiết

Worker cần quyền gọi Amazon Textract để OCR tài liệu.

Quyền cơ bản:

```text
textract:DetectDocumentText
textract:AnalyzeDocument
```

Nếu sử dụng Textract dạng asynchronous job, có thể cần thêm:

```text
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

---

#### Quyền CloudWatch cần thiết

Backend và worker cần ghi log lên CloudWatch.

Quyền cần thiết:

```text
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
logs:DescribeLogStreams
```

Log group đề xuất:

```text
/docmind/application
```

Các log nên ghi:

```text
[DOCUMENT_UPLOAD_STARTED]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[SQS_WORKER_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_PROCESSING_FAILED]
```

Không nên log API key, JWT token, database password hoặc toàn bộ nội dung tài liệu nhạy cảm.

---

#### Quyền Secrets Manager cần thiết

Nếu production dùng AWS Secrets Manager, backend cần quyền đọc secret:

```text
secretsmanager:GetSecretValue
```

Secret có thể lưu:

```text
DATABASE_URL
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
AWS_SQS_QUEUE_URL
```

Secret name đề xuất:

```text
docmind/backend
docmind/production
docmind/ai-keys
```

---

#### Biến môi trường sau khi dùng IAM Role

Khi dùng IAM Role, production không cần lưu access key.

Có thể giữ các biến cấu hình:

```env
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
```

Nếu dùng Secrets Manager:

```env
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
```

Không nên để trong production:

```env
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

---

#### Kiểm tra IAM Permission

Sau khi tạo role và attach policy, cần kiểm tra:

```bash
aws sts get-caller-identity
```

Kết quả mong đợi nếu chạy trên EC2 có gắn IAM Role:

```text
arn:aws:sts::123456789012:assumed-role/DocuMindBackendWorkerRole/i-xxxxxxxx
```

Sau đó test từng dịch vụ:

```bash
aws s3 ls s3://documind-document-storage
```

```bash
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_test","action":"PROCESS_DOCUMENT"}'
```

```bash
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

---

#### Liên kết với pipeline hệ thống

IAM Role được sử dụng xuyên suốt pipeline:

```text
Document Upload API
  |
  |-- uses S3 Policy
  v
Amazon S3

Backend sends job
  |
  |-- uses SQS Policy
  v
Amazon SQS

Worker processing
  |
  |-- uses SQS Policy
  |-- uses S3 Policy
  |-- uses Textract Policy
  v
OCR Result

Backend / Worker logs
  |
  |-- uses CloudWatch Policy
  v
CloudWatch Logs

Production secrets
  |
  |-- uses Secrets Manager Policy
  v
Secrets Manager
```

---

#### Các lỗi thường gặp

| Lỗi                              | Nguyên nhân                           | Cách xử lý                 |
| -------------------------------- | ------------------------------------- | -------------------------- |
| `Unable to locate credentials`   | Backend/EC2 chưa có IAM Role          | Gắn IAM Role vào EC2       |
| `AccessDenied` khi upload S3     | Thiếu `s3:PutObject`                  | Kiểm tra S3 Policy         |
| `AccessDenied` khi đọc S3        | Thiếu `s3:GetObject`                  | Kiểm tra S3 Policy         |
| `QueueDoesNotExist`              | Sai Queue URL hoặc region             | Kiểm tra `.env`            |
| `AccessDenied` khi gửi SQS       | Thiếu `sqs:SendMessage`               | Kiểm tra SQS Policy        |
| `AccessDeniedException` Textract | Thiếu quyền Textract                  | Kiểm tra Textract Policy   |
| Không có CloudWatch log          | Thiếu quyền logs                      | Kiểm tra CloudWatch Policy |
| Không đọc được secret            | Thiếu `secretsmanager:GetSecretValue` | Kiểm tra Secrets Policy    |
| Region mismatch                  | Các service khác region               | Đồng bộ `AWS_REGION`       |

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được:

* Đã tạo IAM Role cho backend/worker.
* Đã tạo S3 Policy.
* Đã tạo SQS và Textract Policy.
* Đã tạo CloudWatch và Secrets Manager Policy.
* Đã attach các policy vào IAM Role.
* Nếu dùng EC2, IAM Role đã được gắn vào instance.
* Backend/worker không cần hardcode AWS Access Key trong production.
* Backend có thể upload file lên S3.
* Worker có thể nhận message từ SQS.
* Worker có thể gọi Amazon Textract.
* Backend/worker có thể ghi log CloudWatch.
* Backend có thể đọc secret từ Secrets Manager nếu được cấu hình.

---

#### Kết quả kỳ vọng

Sau phần này, DocuMind AI đã có lớp phân quyền AWS an toàn hơn. Backend và worker có thể truy cập các dịch vụ AWS cần thiết thông qua IAM Role và IAM Policy, giúp hệ thống giảm rủi ro lộ credentials và sẵn sàng hơn cho môi trường production.
