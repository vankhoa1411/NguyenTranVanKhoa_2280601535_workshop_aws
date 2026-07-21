---

title : "Tạo SQS Processing Queue"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
-----------------------

#### Tạo Amazon SQS Processing Queue

Trong phần này, chúng ta sẽ thiết lập **Amazon SQS** để xây dựng cơ chế xử lý tài liệu bất đồng bộ cho hệ thống **DocuMind AI**.

Sau khi người dùng upload tài liệu thành công lên **Amazon S3**, backend sẽ không xử lý OCR và AI ngay lập tức trong cùng một request. Thay vào đó, backend sẽ tạo một message chứa thông tin tài liệu và gửi vào **Amazon SQS Queue**. Worker sẽ nhận message từ queue, sau đó thực hiện các bước xử lý tiếp theo như gọi **Amazon Textract OCR** và gửi nội dung sang **Gemini/OpenAI AI Gateway**.

Cách thiết kế này giúp hệ thống hoạt động ổn định hơn, tránh tình trạng request upload bị chờ quá lâu, đồng thời tách riêng phần upload tài liệu và phần xử lý tài liệu nền.

---

#### Vai trò của Amazon SQS trong DocuMind AI

Trong dự án **DocuMind AI**, Amazon SQS được sử dụng để:

* Nhận job xử lý tài liệu sau khi upload lên S3.
* Kết nối bất đồng bộ giữa Backend API và Worker.
* Giúp hệ thống tránh quá tải khi nhiều người dùng upload tài liệu cùng lúc.
* Cho phép worker xử lý tài liệu theo hàng đợi.
* Hỗ trợ retry khi quá trình OCR hoặc AI Analysis gặp lỗi.
* Kết hợp với Dead Letter Queue để lưu các message xử lý thất bại nhiều lần.
* Tăng độ ổn định cho pipeline xử lý tài liệu.

Amazon SQS không trực tiếp OCR hoặc phân tích AI. Nó chỉ đóng vai trò là hàng đợi trung gian để truyền message từ backend sang worker.

---

#### Kiến trúc xử lý hàng đợi

Luồng xử lý trong phần này:

```text id="9rtww8"
User
  |
  v
React Frontend
  |
  v
Backend API Node.js/Express
  |
  v
Amazon S3 lưu tài liệu
  |
  v
Amazon SQS nhận message xử lý
  |
  v
Worker poll message từ SQS
```

![sqs-processing-queue](/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/sqs-processing-queue.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Vẽ hoặc chụp sơ đồ minh họa luồng xử lý gồm các thành phần: React Frontend, Backend API, Amazon S3, Amazon SQS Queue và Worker.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.4-Create-SQS-Processing-Queue/sqs-processing-queue.png`

---

#### Message xử lý tài liệu

Sau khi upload tài liệu thành công lên S3, backend sẽ gửi message vào SQS với các thông tin cần thiết để worker xử lý.

Ví dụ message:

```json id="f37bll"
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Trong đó:

| Trường       | Ý nghĩa                             |
| ------------ | ----------------------------------- |
| `documentId` | ID tài liệu đã lưu trong PostgreSQL |
| `userId`     | ID người dùng upload tài liệu       |
| `s3Bucket`   | Tên S3 bucket chứa tài liệu         |
| `s3Key`      | Đường dẫn object trên S3            |
| `action`     | Loại tác vụ cần xử lý               |
| `status`     | Trạng thái hiện tại của tài liệu    |

---

#### Vì sao cần xử lý bất đồng bộ?

Nếu backend xử lý toàn bộ quy trình trong một request:

```text id="5ff0uc"
Upload file
→ OCR bằng Textract
→ Gọi Gemini/OpenAI
→ Lưu kết quả
→ Trả response
```

thì người dùng có thể phải chờ rất lâu, đặc biệt khi file lớn hoặc AI provider phản hồi chậm.

Với SQS, quy trình được tách thành:

```text id="s7wqsb"
Upload file
→ Lưu S3
→ Gửi job vào SQS
→ Trả response ngay cho người dùng
→ Worker xử lý nền
```

Nhờ đó, giao diện có thể hiển thị trạng thái như:

```text id="2j86hv"
UPLOADED
QUEUED
PROCESSING
OCR_COMPLETED
AI_ANALYZING
COMPLETED
FAILED
```

Người dùng có thể theo dõi tiến trình xử lý trên Dashboard hoặc nhận thông báo khi tài liệu hoàn tất.

---

#### Nội dung chi tiết

Trong phần **5.4 - Create SQS Processing Queue**, chúng ta sẽ thực hiện các bước sau:

* [Tạo SQS Queue](5.4.1-create-sqs-queue/)
* [Tạo Dead Letter Queue](5.4.2-create-dead-letter-queue/)
* [Kiểm tra gửi và nhận SQS Message](5.4.3-test-sqs-message/)

---

#### Cấu hình SQS khuyến nghị

Khi tạo SQS queue cho DocuMind AI, có thể sử dụng cấu hình sau:

| Thành phần                | Cấu hình đề xuất               |
| ------------------------- | ------------------------------ |
| Queue type                | Standard                       |
| Visibility timeout        | 300 seconds                    |
| Message retention period  | 4 days                         |
| Delivery delay            | 0 seconds                      |
| Receive message wait time | 20 seconds                     |
| Encryption                | SSE-SQS                        |
| Dead Letter Queue         | Bật nếu muốn xử lý lỗi tốt hơn |

Với workflow xử lý OCR và AI, **Standard Queue** là lựa chọn phù hợp vì hệ thống không yêu cầu thứ tự tuyệt đối giữa các tài liệu. Mỗi tài liệu có `documentId` riêng nên worker có thể xử lý độc lập.

---

#### Biến môi trường liên quan

Backend cần có biến môi trường chứa Queue URL:

```env id="xwf8mo"
AWS_REGION="ap-southeast-1"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
AWS_SQS_DLQ_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-dlq"
```

Nếu không sử dụng Dead Letter Queue trong demo, có thể tạm thời chưa cần biến:

```env id="nbl0pl"
AWS_SQS_DLQ_URL
```

---

#### Quyền IAM cần thiết

Backend cần quyền gửi message vào SQS:

```text id="0m3jkq"
sqs:SendMessage
```

Worker cần quyền nhận và xóa message:

```text id="ze71vu"
sqs:ReceiveMessage
sqs:DeleteMessage
sqs:GetQueueAttributes
```

Các quyền này sẽ được cấu hình rõ hơn trong phần **IAM Role and Policy** của workshop.

---

#### Kết quả sau khi hoàn thành

Sau khi hoàn thành phần này, hệ thống cần đạt được các kết quả sau:

* Đã tạo SQS Standard Queue cho DocuMind AI.
* Đã tạo Dead Letter Queue nếu cần.
* Backend có thể gửi message vào SQS sau khi upload tài liệu.
* Worker có thể nhận message từ SQS.
* Message chứa đầy đủ `documentId`, `userId`, `s3Bucket` và `s3Key`.
* Hệ thống sẵn sàng chuyển sang bước xử lý OCR bằng Amazon Textract.

---

#### Liên kết với bước tiếp theo

Sau khi message được gửi vào Amazon SQS, worker sẽ nhận message và bắt đầu xử lý tài liệu. Ở phần tiếp theo, chúng ta sẽ tích hợp **Amazon Textract OCR** để trích xuất văn bản từ tài liệu đã lưu trên S3.

Luồng xử lý tiếp theo:

```text id="splxg6"
Amazon SQS
  |
  v
Worker
  |
  v
Amazon S3 lấy tài liệu
  |
  v
Amazon Textract OCR
  |
  v
PostgreSQL + Prisma lưu OCR Result
```

Như vậy, phần 5.4 đóng vai trò là lớp điều phối tác vụ nền cho toàn bộ pipeline xử lý tài liệu của DocuMind AI.
