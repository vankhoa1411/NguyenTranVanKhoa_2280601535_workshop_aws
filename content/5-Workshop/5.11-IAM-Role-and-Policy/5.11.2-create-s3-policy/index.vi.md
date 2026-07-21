---

title : "Tạo S3 Policy"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.11.2. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tạo **IAM Policy cho Amazon S3** để backend và worker của **DocuMind AI** có quyền upload, đọc và quản lý tài liệu trong S3 bucket.

Amazon S3 là nơi lưu trữ tài liệu gốc do người dùng upload. Vì vậy, backend cần quyền `PutObject` để upload file, còn worker cần quyền `GetObject` để đọc file và gửi sang Amazon Textract xử lý OCR.

Policy này chỉ nên cấp quyền trên bucket của dự án, không nên cấp quyền quá rộng cho toàn bộ S3.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được IAM Policy cho Amazon S3.
* Cấp quyền backend upload file lên S3.
* Cấp quyền worker đọc file từ S3.
* Giới hạn quyền theo bucket của DocuMind AI.
* Chuẩn bị policy để attach vào IAM Role ở bước sau.

---

#### Vai trò của S3 Policy trong DocuMind AI

S3 Policy cho phép backend/worker thực hiện:

* Upload tài liệu lên S3.
* Đọc tài liệu từ S3 để OCR.
* Xóa object nếu người dùng xóa tài liệu.
* List bucket để kiểm tra object khi cần.
* Không public tài liệu ra internet.

Luồng quyền:

```text id="rx4dh0"
IAM Role
  |
  v
S3 Policy
  |
  v
Amazon S3 Bucket
  |
  v
documents stored as s3Key
```

![s3-policy](/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.2-create-s3-policy/s3-policy.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình IAM Policy dành cho S3 hoặc policy JSON trong AWS Console.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.2-create-s3-policy/s3-policy.png`

---

#### Quyền S3 cần thiết

Các quyền tối thiểu thường dùng:

```text id="kjqbh7"
s3:PutObject
s3:GetObject
s3:DeleteObject
s3:ListBucket
```

Ý nghĩa:

| Quyền             | Mục đích                             |
| ----------------- | ------------------------------------ |
| `s3:PutObject`    | Backend upload file lên S3           |
| `s3:GetObject`    | Worker đọc file từ S3                |
| `s3:DeleteObject` | Xóa file nếu user/admin xóa tài liệu |
| `s3:ListBucket`   | Kiểm tra object trong bucket         |

---

#### Bước 1: Truy cập IAM Policy

Vào AWS Console, tìm:

```text id="6xlwbs"
IAM
```

Chọn:

```text id="67uxve"
Policies
```

Sau đó nhấn:

```text id="ouiznb"
Create policy
```

---

#### Bước 2: Chọn JSON Editor

Trong màn hình tạo policy, chọn tab:

```text id="ul7gey"
JSON
```

Sau đó dán policy mẫu bên dưới.

---

#### Bước 3: Policy JSON cho S3

Thay `documind-document-storage` bằng tên bucket thật của bạn.

```json id="kw34gm"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DocuMindS3ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::documind-document-storage/*"
    },
    {
      "Sid": "DocuMindS3ListBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::documind-document-storage"
    }
  ]
}
```

---

#### Bước 4: Đặt tên policy

Đặt tên policy:

```text id="x7h0dj"
DocuMindS3AccessPolicy
```

Hoặc:

```text id="v2cx91"
DocMind-S3-Document-Storage-Policy
```

Description đề xuất:

```text id="95cz91"
Allows DocuMind AI backend and worker to upload, read, delete and list documents in the project S3 bucket.
```

---

#### Bước 5: Tạo policy

Kiểm tra lại:

| Thành phần     | Giá trị                            |
| -------------- | ---------------------------------- |
| Service        | Amazon S3                          |
| Bucket         | Bucket của DocuMind AI             |
| Object actions | PutObject, GetObject, DeleteObject |
| Bucket action  | ListBucket                         |

Sau đó nhấn:

```text id="5jacb1"
Create policy
```

---

#### Bước 6: Kiểm tra policy sau khi tạo

Sau khi tạo, mở policy và kiểm tra:

* Policy name đúng.
* Resource trỏ đúng bucket.
* Không dùng wildcard quá rộng như `arn:aws:s3:::*`.
* Không có quyền public.
* Không có quyền ngoài nhu cầu workshop.

Không nên dùng policy quá rộng như:

```json id="23pp3h"
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
```

Policy trên quá rộng và không phù hợp với nguyên tắc least privilege.

---

#### Bước 7: Cập nhật bucket name trong `.env`

Đảm bảo `.env` backend có đúng bucket:

```env id="cm9k82"
AWS_S3_BUCKET_NAME="documind-document-storage"
```

Nếu bucket của bạn tên khác, policy và `.env` phải cùng tên bucket.

---

#### Test nhanh quyền S3

Sau khi attach policy vào role, có thể test:

```bash id="x4b5nu"
aws s3 ls s3://documind-document-storage
```

Test upload file:

```bash id="rxi3t7"
aws s3 cp ./invoice-demo.pdf s3://documind-document-storage/test/invoice-demo.pdf
```

Test đọc file:

```bash id="sotbka"
aws s3 cp s3://documind-document-storage/test/invoice-demo.pdf ./downloaded-invoice-demo.pdf
```

---

#### Các lỗi thường gặp

| Lỗi                        | Nguyên nhân           | Cách xử lý                                  |
| -------------------------- | --------------------- | ------------------------------------------- |
| `AccessDenied`             | Thiếu quyền S3        | Kiểm tra policy đã attach vào role chưa     |
| `NoSuchBucket`             | Sai tên bucket        | Kiểm tra bucket name trong policy và `.env` |
| `PermanentRedirect`        | Sai region bucket     | Kiểm tra `AWS_REGION`                       |
| Upload không thành công    | Thiếu `s3:PutObject`  | Thêm quyền PutObject                        |
| Worker không đọc được file | Thiếu `s3:GetObject`  | Thêm quyền GetObject                        |
| Không list được bucket     | Thiếu `s3:ListBucket` | Thêm quyền ListBucket                       |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo S3 IAM Policy.
* Policy chỉ cấp quyền cho bucket DocuMind AI.
* Policy có `s3:PutObject`.
* Policy có `s3:GetObject`.
* Policy có `s3:DeleteObject` nếu cần xóa file.
* Policy có `s3:ListBucket`.
* Policy không cấp quyền quá rộng.
* Policy sẵn sàng attach vào IAM Role.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có IAM Policy cho Amazon S3. Backend có thể upload tài liệu lên S3 và worker có thể đọc tài liệu để xử lý OCR bằng Amazon Textract.
