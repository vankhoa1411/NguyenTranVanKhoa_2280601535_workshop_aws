---

title : "Tạo SQS và Textract Policy"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.11.3. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tạo **IAM Policy cho Amazon SQS và Amazon Textract** cho hệ thống **DocuMind AI**.

Backend cần quyền gửi message vào **Amazon SQS** sau khi upload tài liệu. Worker cần quyền nhận message, xóa message khỏi queue sau khi xử lý thành công và gọi **Amazon Textract** để OCR tài liệu.

Policy này giúp pipeline bất đồng bộ hoạt động đầy đủ từ upload tài liệu đến OCR processing.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được IAM Policy cho Amazon SQS.
* Cấp quyền backend gửi message vào SQS.
* Cấp quyền worker nhận và xóa message từ SQS.
* Cấp quyền worker gọi Amazon Textract.
* Giới hạn quyền theo queue của DocuMind AI nếu có thể.
* Chuẩn bị policy để attach vào IAM Role.

---

#### Vai trò của SQS và Textract Policy

Trong DocuMind AI:

* Backend dùng SQS để gửi job xử lý tài liệu.
* Worker dùng SQS để nhận job xử lý.
* Worker dùng Textract để OCR tài liệu PDF/PNG/JPG/JPEG.
* Sau khi xử lý thành công, worker xóa message khỏi SQS.
* Nếu xử lý lỗi, message có thể retry hoặc chuyển sang Dead Letter Queue.

Luồng quyền:

```text id="fv6hlj"
IAM Role
  |
  |-- SQS Policy --> Amazon SQS
  |
  |-- Textract Policy --> Amazon Textract
```

![sqs-textract-policy](/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.3-create-sqs-textract-policy/sqs-textract-policy.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình IAM Policy có quyền SQS và Textract, hoặc vẽ sơ đồ IAM Role truy cập Amazon SQS và Amazon Textract.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.3-create-sqs-textract-policy/sqs-textract-policy.png`

---

#### Quyền SQS cần thiết

Backend cần:

```text id="l2sp73"
sqs:SendMessage
```

Worker cần:

```text id="lk222a"
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
sqs:ChangeMessageVisibility
```

Nếu có Dead Letter Queue, Admin hoặc worker có thể cần đọc thông tin DLQ:

```text id="v9qy16"
sqs:GetQueueAttributes
sqs:ReceiveMessage
```

---

#### Quyền Textract cần thiết

Nếu dùng OCR đơn giản:

```text id="y67vdq"
textract:DetectDocumentText
textract:AnalyzeDocument
```

Nếu dùng Textract asynchronous job:

```text id="g0su31"
textract:StartDocumentTextDetection
textract:GetDocumentTextDetection
textract:StartDocumentAnalysis
textract:GetDocumentAnalysis
```

Trong workshop, có thể cấp cả hai nhóm quyền để linh hoạt test OCR.

---

#### Bước 1: Truy cập IAM Policy

Vào AWS Console:

```text id="42e6u7"
IAM
```

Chọn:

```text id="4724h2"
Policies
```

Nhấn:

```text id="lwe5j1"
Create policy
```

Chọn tab:

```text id="p2ycpr"
JSON
```

---

#### Bước 2: Lấy ARN của SQS Queue

Vào Amazon SQS, mở queue:

```text id="n86z5j"
docmind-document-processing-queue
```

Copy ARN, ví dụ:

```text id="dlmfpz"
arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-queue
```

Nếu có DLQ, copy thêm ARN:

```text id="tlxoj5"
arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-dlq
```

---

#### Bước 3: Policy JSON cho SQS và Textract

Thay `account-id`, region và queue name theo tài khoản AWS của bạn.

```json id="d1e0ui"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindSQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": [
        "arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-queue",
        "arn:aws:sqs:ap-southeast-1:123456789012:docmind-document-processing-dlq"
      ]
    },
    {
      "Sid": "DocuMindTextractAccess",
      "Effect": "Allow",
      "Action": [
        "textract:DetectDocumentText",
        "textract:AnalyzeDocument",
        "textract:StartDocumentTextDetection",
        "textract:GetDocumentTextDetection",
        "textract:StartDocumentAnalysis",
        "textract:GetDocumentAnalysis"
      ],
      "Resource": "*"
    }
  ]
}
```

> ⚠️ **Lưu ý:**
> Amazon Textract thường sử dụng `Resource: "*"` cho nhiều action. Với SQS, nên giới hạn theo ARN queue thật của dự án.

---

#### Bước 4: Đặt tên policy

Đặt tên:

```text id="rknhdg"
DocuMindSQSTextractPolicy
```

Description:

```text id="t32mfr"
Allows DocuMind AI backend and worker to send, receive and delete SQS messages and call Amazon Textract OCR.
```

Sau đó nhấn:

```text id="rc5bfm"
Create policy
```

---

#### Bước 5: Kiểm tra biến môi trường

Backend/worker cần có:

```env id="n93qvn"
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

Nếu không dùng DLQ trong demo, có thể chưa cần `AWS_SQS_DLQ_URL`.

---

#### Test nhanh quyền SQS

Sau khi attach policy vào role, test gửi message:

```bash id="82rhzg"
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_test_001","action":"PROCESS_DOCUMENT"}'
```

Test nhận message:

```bash id="77h1ye"
aws sqs receive-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --max-number-of-messages 1 \
  --wait-time-seconds 10
```

---

#### Test nhanh quyền Textract

Sau khi có file PDF hoặc ảnh trong S3, worker có thể gọi Textract. Nếu thiếu quyền, thường gặp lỗi:

```text id="rnewye"
AccessDeniedException
```

Nếu có quyền, Textract sẽ trả về blocks OCR như:

```text id="03bc90"
PAGE
LINE
WORD
```

---

#### Các lỗi thường gặp

| Lỗi                              | Nguyên nhân                         | Cách xử lý                     |
| -------------------------------- | ----------------------------------- | ------------------------------ |
| `AccessDenied` khi gửi SQS       | Thiếu `sqs:SendMessage`             | Kiểm tra policy                |
| Worker không nhận message        | Thiếu `sqs:ReceiveMessage`          | Thêm quyền ReceiveMessage      |
| Worker xử lý lặp                 | Thiếu hoặc chưa gọi `DeleteMessage` | Kiểm tra quyền và logic worker |
| `AccessDeniedException` Textract | Thiếu quyền Textract                | Thêm quyền Textract            |
| `QueueDoesNotExist`              | Sai Queue URL hoặc region           | Kiểm tra `.env`                |
| `InvalidS3ObjectException`       | Textract không đọc được S3 object   | Kiểm tra S3 policy và s3Key    |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo policy cho SQS.
* Policy có `sqs:SendMessage`.
* Policy có `sqs:ReceiveMessage`.
* Policy có `sqs:DeleteMessage`.
* Policy có `sqs:GetQueueAttributes`.
* Policy có quyền gọi Textract.
* SQS resource trỏ đúng ARN queue.
* Policy sẵn sàng attach vào IAM Role.

---

#### Kết quả kỳ vọng

Sau bước này, backend có thể gửi message vào SQS và worker có thể nhận message để gọi Amazon Textract OCR. Đây là policy quan trọng giúp pipeline xử lý tài liệu bất đồng bộ hoạt động hoàn chỉnh.
