---

title : "CloudWatch Logs"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.12.1. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ cấu hình **Amazon CloudWatch Logs** cho hệ thống **DocuMind AI**.

CloudWatch Logs giúp backend và worker ghi lại các sự kiện quan trọng trong quá trình vận hành như upload tài liệu, gửi message vào SQS, xử lý OCR bằng Amazon Textract, phân tích AI bằng Gemini/OpenAI, lỗi worker, lỗi quyền IAM và các sự kiện bảo mật.

Việc có log tập trung giúp developer và Admin dễ kiểm tra lỗi, theo dõi pipeline xử lý tài liệu và đánh giá trạng thái hoạt động của hệ thống.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo CloudWatch Log Group cho DocuMind AI.
* Ghi log từ backend và worker lên CloudWatch.
* Phân loại log theo module xử lý.
* Kiểm tra log upload, SQS, OCR và AI Analysis.
* Tạo nền tảng cho monitoring và alert.
* Hạn chế log dữ liệu nhạy cảm.

---

#### Vai trò của CloudWatch Logs trong DocuMind AI

Trong hệ thống **DocuMind AI**, CloudWatch Logs được dùng để theo dõi:

* Backend API start/stop.
* Document Upload API.
* S3 upload result.
* SQS send/receive message.
* OCR Worker processing.
* Amazon Textract OCR result.
* AI Gateway provider selection.
* Gemini/OpenAI response status.
* Notification creation.
* Audit Log creation.
* Security event và forbidden access.
* Lỗi hệ thống trong quá trình xử lý tài liệu.

---

#### Kiến trúc CloudWatch Logs

Luồng ghi log:

```text id="9y4qoc"
Backend API / Worker
  |
  v
Logger Service
  |
  v
Amazon CloudWatch Logs
  |
  v
Log Group: /docmind/application
```

![cloudwatch-logs](/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.1-cloudwatch-logs/cloudwatch-logs.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình CloudWatch Log Group `/docmind/application` có log từ backend hoặc worker.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.1-cloudwatch-logs/cloudwatch-logs.png`

---

#### Log Group đề xuất

Log group cho ứng dụng:

```text id="v9x1po"
/docmind/application
```

Có thể tách thêm nếu muốn:

```text id="z7ba8h"
/docmind/backend
/docmind/worker
/docmind/security
```

Trong workshop này, có thể dùng một log group chính:

```text id="x3ft4h"
/docmind/application
```

---

#### Bước 1: Truy cập CloudWatch

Đăng nhập vào AWS Console, tìm dịch vụ:

```text id="f2kzdt"
CloudWatch
```

Chọn:

```text id="nl5rpd"
Logs
```

Sau đó chọn:

```text id="gs14ve"
Log groups
```

---

#### Bước 2: Tạo Log Group

Nhấn:

```text id="x3bmiq"
Create log group
```

Nhập tên log group:

```text id="dglyo3"
/docmind/application
```

Chọn retention phù hợp.

Đối với workshop/demo:

```text id="vluk2o"
Retention: 7 days
```

Đối với production:

```text id="hf3xti"
Retention: 30 days hoặc 90 days
```

Sau đó nhấn:

```text id="khamn3"
Create
```

---

#### Bước 3: Cấu hình biến môi trường

Thêm vào file `.env` backend:

```env id="ib3x4r"
AWS_REGION="ap-southeast-1"
CLOUDWATCH_LOG_GROUP="/docmind/application"
CLOUDWATCH_LOG_STREAM="backend-dev"
```

Nếu worker dùng log stream riêng:

```env id="p82d8i"
CLOUDWATCH_WORKER_LOG_STREAM="worker-dev"
```

---

#### Bước 4: Quyền IAM cần thiết

Backend/worker cần quyền:

```text id="foa1tp"
logs:CreateLogGroup
logs:CreateLogStream
logs:PutLogEvents
logs:DescribeLogStreams
```

Các quyền này đã được chuẩn bị ở phần **5.11 - IAM Role and Policy**.

---

#### Bước 5: Log các sự kiện backend

Backend nên ghi log cho các mốc:

```text id="4m5vz2"
[APP_STARTED]
[DATABASE_CONNECTED]
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_SUCCESS]
[DOCUMENT_STATUS_REQUESTED]
[USER_NOTIFICATION_CREATED]
[AUDIT_LOG_CREATED]
```

Ví dụ log:

```json id="mm4btk"
{
  "level": "info",
  "event": "DOCUMENT_UPLOAD_STARTED",
  "message": "User started uploading document.",
  "userId": "user_001",
  "timestamp": "2026-06-10T10:00:00.000Z"
}
```

---

#### Bước 6: Log các sự kiện worker

Worker nên ghi log:

```text id="xvm1il"
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
[TEXTRACT_OCR_STARTED]
[TEXTRACT_OCR_COMPLETED]
[AI_ANALYSIS_STARTED]
[AI_ANALYSIS_COMPLETED]
[DOCUMENT_STATUS_COMPLETED]
[SQS_MESSAGE_DELETED]
```

Nếu lỗi:

```text id="fojqul"
[DOCUMENT_PROCESSING_FAILED]
[TEXTRACT_OCR_FAILED]
[AI_PROVIDER_FAILED]
[AI_PROVIDER_FALLBACK_USED]
[SQS_MESSAGE_RETRY_PENDING]
```

---

#### Bước 7: Không log dữ liệu nhạy cảm

Không nên log các thông tin sau:

```text id="0es8lv"
OPENAI_API_KEY
GEMINI_API_KEY
JWT_SECRET
DATABASE_URL có password
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
Nội dung đầy đủ của tài liệu nhạy cảm
JWT token
Password
```

Chỉ nên log metadata cần thiết như:

```text id="fpk9zp"
documentId
userId
fileName
status
provider
model
errorCode
```

---

#### Bước 8: Kiểm tra log trên CloudWatch

Sau khi chạy backend/worker, vào:

```text id="zjshpf"
CloudWatch
→ Logs
→ Log groups
→ /docmind/application
```

Mở log stream:

```text id="yv7gdl"
backend-dev
worker-dev
```

Kiểm tra có log mới.

---

#### Bước 9: Test log bằng upload tài liệu

Các bước test:

1. Start backend.
2. Start worker.
3. Upload file PDF từ frontend hoặc Postman.
4. Kiểm tra log backend có `DOCUMENT_UPLOAD_STARTED`.
5. Kiểm tra log S3 có `S3_UPLOAD_SUCCESS`.
6. Kiểm tra log SQS có `SQS_SEND_MESSAGE_SUCCESS`.
7. Kiểm tra log worker có `TEXTRACT_OCR_COMPLETED`.
8. Kiểm tra log AI có `AI_ANALYSIS_COMPLETED`.

---

#### Các lỗi thường gặp

| Lỗi                     | Nguyên nhân                        | Cách xử lý                       |
| ----------------------- | ---------------------------------- | -------------------------------- |
| Không thấy log group    | Chưa tạo log group hoặc sai region | Kiểm tra region CloudWatch       |
| Không thấy log stream   | Backend chưa tạo stream            | Kiểm tra logger config           |
| `AccessDeniedException` | Thiếu quyền CloudWatch Logs        | Kiểm tra IAM Policy              |
| Log không cập nhật      | Backend chưa gửi log               | Kiểm tra logger service          |
| Log bị lộ secret        | Log quá nhiều dữ liệu              | Mask hoặc loại bỏ field nhạy cảm |
| Sai timestamp           | Server timezone sai                | Dùng ISO timestamp UTC           |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo CloudWatch Log Group.
* Backend ghi được log lên CloudWatch.
* Worker ghi được log lên CloudWatch.
* Log upload, SQS, OCR và AI Analysis xuất hiện.
* Log lỗi được ghi rõ ràng.
* Không có secret hoặc token trong log.
* Admin có thể dùng log để debug pipeline.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có hệ thống log tập trung bằng CloudWatch Logs. Backend và worker có thể ghi lại các sự kiện quan trọng, giúp việc kiểm tra lỗi, monitoring và vận hành hệ thống dễ dàng hơn.
