---

title : "Kiểm tra gửi và nhận SQS Message"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.4.3. </b> "
-------------------------

#### Tổng quan

Sau khi đã tạo **SQS Processing Queue** và **Dead Letter Queue**, bước tiếp theo là kiểm tra việc gửi và nhận message trong hệ thống **DocuMind AI**.

Ở bước này, chúng ta sẽ test xem backend có thể gửi message vào SQS sau khi upload tài liệu hay chưa, đồng thời kiểm tra worker có thể nhận message từ queue để chuẩn bị xử lý OCR và AI Analysis hay không.

Việc test SQS giúp đảm bảo pipeline bất đồng bộ hoạt động đúng trước khi tích hợp sâu với **Amazon Textract**, **Gemini API** và **OpenAI ChatGPT API**.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Gửi thử một message vào SQS.
* Kiểm tra message xuất hiện trong queue.
* Worker hoặc AWS Console có thể nhận message.
* Hiểu cấu trúc message xử lý tài liệu.
* Kiểm tra lỗi quyền IAM hoặc sai Queue URL.
* Xác nhận hệ thống sẵn sàng chuyển sang bước Textract OCR.

---

#### Luồng kiểm tra message

Luồng gửi và nhận message trong DocuMind AI:

```text
Backend API / AWS Console
  |
  v
Amazon SQS Processing Queue
  |
  v
Worker poll message
  |
  v
Parse documentId + s3Key
  |
  v
Chuẩn bị xử lý OCR
```

![test-sqs-message](/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.3-test-sqs-message/test-sqs-message.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình message được gửi vào SQS và màn hình worker hoặc AWS Console nhận được message.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/5.4.3-test-sqs-message/test-sqs-message.png`

---

#### Điều kiện trước khi kiểm tra

Trước khi test, hãy đảm bảo:

* Đã tạo main SQS queue.
* Đã có Queue URL.
* Đã tạo DLQ nếu dùng.
* Queue URL đã được thêm vào `.env`.
* Backend hoặc AWS CLI có quyền `sqs:SendMessage`.
* Worker có quyền `sqs:ReceiveMessage` và `sqs:DeleteMessage`.
* AWS region trong `.env` đúng với region của queue.

Ví dụ `.env`:

```env
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-dlq"
```

---

#### Bước 1: Gửi message test bằng AWS Console

Vào AWS Console:

```text
Amazon SQS
```

Chọn queue:

```text
docmind-document-processing-queue
```

Chọn:

```text
Send and receive messages
```

Ở phần **Message body**, nhập message mẫu:

```json
{
  "documentId": "doc_test_001",
  "userId": "user_test_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_test_001/doc_test_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Sau đó nhấn:

```text
Send message
```

Nếu gửi thành công, AWS sẽ hiển thị thông báo message đã được gửi vào queue.

---

#### Bước 2: Kiểm tra message trong queue

Sau khi gửi message, quay lại trang chi tiết queue và kiểm tra:

```text
Available messages
```

Nếu số lượng message tăng lên, nghĩa là message đã nằm trong queue và đang chờ worker xử lý.

---

#### Bước 3: Nhận message bằng AWS Console

Trong trang:

```text
Send and receive messages
```

Nhấn:

```text
Poll for messages
```

Nếu thành công, bạn sẽ thấy message vừa gửi xuất hiện.

Kiểm tra nội dung message có đủ:

* `documentId`
* `userId`
* `s3Bucket`
* `s3Key`
* `action`
* `status`

---

#### Bước 4: Gửi message bằng AWS CLI

Bạn cũng có thể gửi message bằng AWS CLI:

```bash
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --message-body '{"documentId":"doc_test_001","userId":"user_test_001","s3Bucket":"documind-document-storage","s3Key":"uploads/user_test_001/doc_test_001/invoice-demo.pdf","action":"PROCESS_DOCUMENT","status":"QUEUED"}'
```

Nếu thành công, AWS CLI sẽ trả về `MessageId`.

---

#### Bước 5: Nhận message bằng AWS CLI

Kiểm tra nhận message:

```bash
aws sqs receive-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --max-number-of-messages 1 \
  --wait-time-seconds 10
```

Nếu nhận được message, CLI sẽ trả về nội dung message cùng `ReceiptHandle`.

---

#### Bước 6: Xóa message sau khi xử lý thành công

Trong thực tế, worker chỉ nên xóa message khi xử lý thành công.

Nếu muốn test xóa message bằng AWS CLI, dùng `ReceiptHandle` từ bước nhận message:

```bash
aws sqs delete-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/123456789012/docmind-document-processing-queue" \
  --receipt-handle "your-receipt-handle"
```

Nếu worker xử lý lỗi, không nên xóa message. Khi hết visibility timeout, message sẽ quay lại queue để retry.

---

#### Bước 7: Test backend gửi message sau upload

Sau khi API upload tài liệu lên S3 thành công, backend nên gửi message vào SQS.

API upload có thể trả response:

```json
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED"
  }
}
```

Log backend mong đợi:

```text
[DOCUMENT_UPLOAD_SUCCESS]
[SQS_SEND_MESSAGE_STARTED]
[SQS_SEND_MESSAGE_SUCCESS]
[DOCUMENT_STATUS_UPDATED_TO_QUEUED]
```

---

#### Bước 8: Test worker nhận message

Khi worker chạy, log mong đợi:

```text
[SQS_WORKER_STARTED]
[SQS_MESSAGE_RECEIVED]
[DOCUMENT_PROCESSING_STARTED]
```

Ở bước này, worker chỉ cần nhận và parse message thành công. Việc gọi Textract OCR sẽ được thực hiện ở phần 5.5.

Worker cần đọc được các field:

```text
documentId
userId
s3Bucket
s3Key
action
```

Nếu message thiếu field quan trọng, worker nên log lỗi và không xử lý tiếp.

---

#### Bước 9: Test message lỗi và DLQ

Để test DLQ, có thể gửi message sai, ví dụ thiếu `s3Key`:

```json
{
  "documentId": "doc_error_001",
  "userId": "user_test_001",
  "action": "PROCESS_DOCUMENT"
}
```

Worker nên xử lý lỗi, không gọi `DeleteMessage`. Sau khi retry vượt quá `Maximum receives`, message sẽ được chuyển sang DLQ.

Kiểm tra DLQ:

```text
docmind-document-processing-dlq
```

Nếu thấy message xuất hiện trong DLQ, Redrive policy đã hoạt động.

---

#### Các lỗi thường gặp

| Lỗi                     | Nguyên nhân                                   | Cách xử lý                         |
| ----------------------- | --------------------------------------------- | ---------------------------------- |
| `AccessDenied`          | Thiếu quyền SQS                               | Kiểm tra IAM Policy                |
| `InvalidAddress`        | Sai Queue URL                                 | Copy lại Queue URL                 |
| `QueueDoesNotExist`     | Queue sai region hoặc bị xóa                  | Kiểm tra AWS region                |
| Không nhận được message | Message đang invisible                        | Chờ hết visibility timeout         |
| Worker nhận message lặp | Không delete message sau khi xử lý thành công | Gọi `DeleteMessage` khi thành công |
| Message không vào DLQ   | Chưa cấu hình Redrive policy                  | Kiểm tra DLQ và Maximum receives   |
| JSON parse error        | Message body không đúng JSON                  | Kiểm tra format message            |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Gửi message test vào SQS thành công.
* Queue hiển thị số lượng message.
* AWS Console hoặc CLI nhận được message.
* Message có đầy đủ `documentId`, `userId`, `s3Bucket`, `s3Key`.
* Backend có thể gửi message sau khi upload tài liệu.
* Worker có thể poll message từ SQS.
* Message lỗi có thể retry hoặc chuyển sang DLQ.
* Hệ thống sẵn sàng chuyển sang phần 5.5 - Integrate Textract OCR.

---

#### Kết quả kỳ vọng

Sau bước này, pipeline bất đồng bộ của DocuMind AI đã hoạt động ở mức cơ bản. Backend có thể đưa tài liệu vào hàng đợi xử lý, worker có thể nhận message và hệ thống đã sẵn sàng tích hợp Amazon Textract để OCR tài liệu trong phần tiếp theo.
