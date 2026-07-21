---

title : "Xây dựng Document Upload API"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.8.1. </b> "
-------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ xây dựng **Document Upload API** cho hệ thống **DocuMind AI**.

API này cho phép người dùng tải tài liệu lên hệ thống. Backend sẽ nhận file, kiểm tra định dạng và dung lượng, upload file lên **Amazon S3**, lưu metadata vào **PostgreSQL thông qua Prisma**, sau đó gửi message vào **Amazon SQS** để worker xử lý OCR và AI Analysis ở các bước tiếp theo.

Đây là API quan trọng nhất trong pipeline vì toàn bộ quá trình xử lý tài liệu bắt đầu từ bước upload.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được API upload tài liệu.
* Kiểm tra file type và file size trước khi upload.
* Upload file lên Amazon S3.
* Lưu metadata tài liệu vào PostgreSQL.
* Tạo `s3Bucket` và `s3Key` cho tài liệu.
* Gửi job xử lý tài liệu vào Amazon SQS.
* Trả response rõ ràng cho frontend.

---

#### Luồng Document Upload API

Luồng xử lý API upload trong DocuMind AI:

```text id="1e340w"
React Frontend / Postman
  |
  v
POST /api/documents/upload
  |
  v
Validate file + user
  |
  v
Upload file to Amazon S3
  |
  v
Save document metadata to PostgreSQL
  |
  v
Send processing message to Amazon SQS
  |
  v
Return response to user
```

![document-upload-api](/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.1-document-upload-api/document-upload-api.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình Postman hoặc frontend upload tài liệu thành công, kèm log backend hiển thị file đã upload lên S3 và message đã gửi vào SQS.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.8-Backend-API-and-Worker/5.8.1-document-upload-api/document-upload-api.png`

---

#### Endpoint đề xuất

API upload tài liệu:

```text id="jlm42d"
POST /api/documents/upload
```

Request dạng:

```text id="i1yk76"
multipart/form-data
```

Field upload:

| Key           | Type | Mô tả                     |
| ------------- | ---- | ------------------------- |
| `file`        | File | Tài liệu PDF/PNG/JPG/JPEG |
| `title`       | Text | Tên tài liệu tùy chọn     |
| `description` | Text | Mô tả tùy chọn            |

Nếu API yêu cầu đăng nhập, cần gửi JWT token:

```text id="jaqfiv"
Authorization: Bearer your_jwt_token
```

---

#### Định dạng file được phép

Backend chỉ nên cho phép các định dạng phù hợp với Amazon Textract:

```text id="pxe6br"
PDF
PNG
JPG
JPEG
```

MIME type hợp lệ:

```text id="4idhln"
application/pdf
image/png
image/jpeg
```

Không nên cho phép:

```text id="757wrh"
DOCX
XLSX
PPTX
EXE
SH
BAT
JS
```

Nếu cần xử lý DOCX, XLSX hoặc PPTX, nên chuyển đổi sang PDF trước khi upload vào pipeline OCR.

---

#### Giới hạn dung lượng file

Đối với workshop/demo, có thể giới hạn:

```text id="6bz1qx"
Max file size: 10MB
```

Nếu muốn xử lý tài liệu lớn hơn, có thể tăng lên:

```text id="jzrn1k"
Max file size: 20MB
```

Response khi file quá lớn:

```json id="5i1531"
{
  "success": false,
  "message": "File size exceeds the allowed limit."
}
```

---

#### Cấu trúc file backend đề xuất

Có thể tổ chức source code như sau:

```text id="tokybt"
src/
├── controllers/
│   └── document.controller.ts
├── routes/
│   └── document.routes.ts
├── services/
│   ├── document.service.ts
│   ├── s3.service.ts
│   └── sqs.service.ts
├── middlewares/
│   ├── auth.middleware.ts
│   └── upload.middleware.ts
└── prisma/
    └── client.ts
```

Vai trò từng file:

| File                     | Vai trò                           |
| ------------------------ | --------------------------------- |
| `document.controller.ts` | Xử lý request/response upload     |
| `document.routes.ts`     | Khai báo route upload             |
| `document.service.ts`    | Lưu metadata và cập nhật document |
| `s3.service.ts`          | Upload file lên Amazon S3         |
| `sqs.service.ts`         | Gửi message vào Amazon SQS        |
| `upload.middleware.ts`   | Xử lý multipart/form-data         |
| `auth.middleware.ts`     | Kiểm tra user đã đăng nhập        |

---

#### Cấu hình multer upload

Nếu dùng `multer`, cài đặt:

```bash id="h25yo8"
npm install multer
npm install --save-dev @types/multer
```

Ví dụ middleware:

```ts id="swzf9y"
import multer from "multer";

const storage = multer.memoryStorage();

export const upload = multer({
  storage,
  limits: {
    fileSize: 10 * 1024 * 1024
  },
  fileFilter: (req, file, cb) => {
    const allowedTypes = ["application/pdf", "image/png", "image/jpeg"];

    if (!allowedTypes.includes(file.mimetype)) {
      return cb(new Error("Unsupported file type"));
    }

    cb(null, true);
  }
});
```

---

#### Upload file lên S3

Ví dụ service upload S3:

```ts id="n328gn"
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const s3Client = new S3Client({
  region: process.env.AWS_REGION
});

export async function uploadFileToS3(params: {
  buffer: Buffer;
  fileName: string;
  mimeType: string;
  userId: string;
  documentId: string;
}) {
  const bucket = process.env.AWS_S3_BUCKET_NAME!;

  const s3Key = `uploads/${params.userId}/${params.documentId}/${params.fileName}`;

  await s3Client.send(
    new PutObjectCommand({
      Bucket: bucket,
      Key: s3Key,
      Body: params.buffer,
      ContentType: params.mimeType
    })
  );

  return {
    s3Bucket: bucket,
    s3Key
  };
}
```

---

#### Lưu metadata vào PostgreSQL

Sau khi upload lên S3 thành công, lưu metadata vào bảng `Document`.

Ví dụ:

```ts id="dxi3cq"
const document = await prisma.document.create({
  data: {
    userId,
    fileName: file.originalname,
    fileType: file.mimetype,
    fileSize: file.size,
    s3Bucket,
    s3Key,
    status: "UPLOADED"
  }
});
```

Sau khi gửi message vào SQS, có thể cập nhật trạng thái:

```ts id="77zvuc"
await prisma.document.update({
  where: { id: document.id },
  data: { status: "QUEUED" }
});
```

---

#### Gửi message vào SQS

Message gửi vào queue:

```json id="x6bkcb"
{
  "documentId": "doc_001",
  "userId": "user_001",
  "s3Bucket": "documind-document-storage",
  "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
  "action": "PROCESS_DOCUMENT",
  "status": "QUEUED"
}
```

Ví dụ service gửi message:

```ts id="r2eqza"
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";

const sqsClient = new SQSClient({
  region: process.env.AWS_REGION
});

export async function sendDocumentProcessingMessage(payload: any) {
  await sqsClient.send(
    new SendMessageCommand({
      QueueUrl: process.env.AWS_SQS_QUEUE_URL!,
      MessageBody: JSON.stringify(payload)
    })
  );
}
```

---

#### Response API mong đợi

Khi upload thành công:

```json id="fn8ubf"
{
  "success": true,
  "message": "Document uploaded and queued successfully",
  "data": {
    "id": "doc_001",
    "fileName": "invoice-demo.pdf",
    "fileType": "application/pdf",
    "fileSize": 204800,
    "s3Bucket": "documind-document-storage",
    "s3Key": "uploads/user_001/doc_001/invoice-demo.pdf",
    "status": "QUEUED",
    "createdAt": "2026-06-10T10:00:00.000Z"
  }
}
```

Khi thiếu file:

```json id="2ojr5h"
{
  "success": false,
  "message": "No file uploaded."
}
```

Khi sai định dạng:

```json id="a7sm44"
{
  "success": false,
  "message": "Unsupported file type. Only PDF, PNG, JPG and JPEG are allowed."
}
```

---

#### Test bằng Postman

Tạo request:

```text id="x6369y"
Method: POST
URL: http://localhost:3000/api/documents/upload
```

Body chọn:

```text id="5sbjcu"
form-data
```

Thêm field:

| Key    | Type | Value            |
| ------ | ---- | ---------------- |
| `file` | File | invoice-demo.pdf |

Nếu có authentication, thêm:

```text id="brkhqd"
Authorization: Bearer your_jwt_token
```

---

#### Log backend mong đợi

Backend nên ghi log:

```text id="hff4l5"
[DOCUMENT_UPLOAD_STARTED]
[FILE_VALIDATION_SUCCESS]
[S3_UPLOAD_SUCCESS]
[DOCUMENT_METADATA_SAVED]
[SQS_SEND_MESSAGE_SUCCESS]
[DOCUMENT_STATUS_UPDATED_TO_QUEUED]
```

Nếu lỗi:

```text id="kec972"
[DOCUMENT_UPLOAD_FAILED]
[S3_UPLOAD_FAILED]
[SQS_SEND_MESSAGE_FAILED]
```

---

#### Các lỗi thường gặp

| Lỗi                     | Nguyên nhân                   | Cách xử lý                            |
| ----------------------- | ----------------------------- | ------------------------------------- |
| `No file uploaded`      | Request không có field `file` | Kiểm tra form-data                    |
| `Unsupported file type` | File không đúng định dạng     | Dùng PDF/PNG/JPG/JPEG                 |
| `AccessDenied`          | Thiếu quyền S3                | Kiểm tra IAM Policy                   |
| `NoSuchBucket`          | Sai bucket name               | Kiểm tra `.env`                       |
| `QueueDoesNotExist`     | Sai SQS Queue URL             | Kiểm tra `AWS_SQS_QUEUE_URL`          |
| `Unauthorized`          | Thiếu JWT token               | Đăng nhập và gửi Authorization header |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* API upload hoạt động.
* Backend validate được file type và file size.
* File được upload lên S3.
* Metadata được lưu vào PostgreSQL.
* Message được gửi vào SQS.
* Trạng thái document là `QUEUED`.
* Response API trả về rõ ràng.
* Log backend thể hiện đầy đủ quá trình upload.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có API upload tài liệu hoàn chỉnh. Người dùng có thể upload tài liệu, file được lưu trên Amazon S3, metadata được lưu vào PostgreSQL và job xử lý được đưa vào Amazon SQS để worker tiếp tục OCR và AI Analysis.
