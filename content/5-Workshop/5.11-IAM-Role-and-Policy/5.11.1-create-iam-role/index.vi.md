---

title : "Tạo IAM Role"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.11.1. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tạo **IAM Role** cho hệ thống **DocuMind AI**.

IAM Role được sử dụng để cấp quyền cho backend hoặc worker truy cập các dịch vụ AWS như **Amazon S3**, **Amazon SQS**, **Amazon Textract**, **Amazon CloudWatch** và **AWS Secrets Manager** mà không cần hardcode `AWS_ACCESS_KEY_ID` hoặc `AWS_SECRET_ACCESS_KEY` trong source code.

Đây là bước quan trọng giúp hệ thống an toàn hơn khi deploy backend lên EC2 hoặc môi trường server production.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo được IAM Role cho backend/worker.
* Hiểu vai trò của IAM Role trong DocuMind AI.
* Biết cách chọn trusted entity phù hợp.
* Chuẩn bị role để gắn các IAM Policy ở các bước tiếp theo.
* Tránh hardcode AWS credentials trong production.

---

#### Vai trò của IAM Role trong DocuMind AI

Trong hệ thống **DocuMind AI**, IAM Role được dùng để cấp quyền cho backend/worker thực hiện các tác vụ:

* Upload tài liệu lên Amazon S3.
* Đọc tài liệu từ Amazon S3.
* Gửi message vào Amazon SQS.
* Nhận và xóa message từ Amazon SQS.
* Gọi Amazon Textract để OCR tài liệu.
* Ghi log lên Amazon CloudWatch.
* Đọc secret từ AWS Secrets Manager.

Luồng sử dụng IAM Role:

```text id="6xkw9n"
Backend / Worker / EC2
  |
  v
IAM Role
  |
  v
AWS Services
  |
  |-- Amazon S3
  |-- Amazon SQS
  |-- Amazon Textract
  |-- CloudWatch
  |-- Secrets Manager
```

![create-iam-role](/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.1-create-iam-role/create-iam-role.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình IAM Role vừa tạo trong AWS Console, hoặc vẽ sơ đồ Backend/Worker sử dụng IAM Role để truy cập S3, SQS, Textract, CloudWatch và Secrets Manager.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.11-IAM-Role-and-Policy/5.11.1-create-iam-role/create-iam-role.png`

---

#### Bước 1: Truy cập IAM

Đăng nhập vào AWS Console, sau đó tìm kiếm dịch vụ:

```text id="yfg2at"
IAM
```

Chọn:

```text id="j7od6i"
Identity and Access Management
```

Trong sidebar, chọn:

```text id="a19n6p"
Roles
```

Sau đó nhấn:

```text id="zf6tww"
Create role
```

---

#### Bước 2: Chọn Trusted Entity

Ở phần **Trusted entity type**, nếu backend/worker chạy trên EC2, chọn:

```text id="e25cbh"
AWS service
```

Sau đó chọn use case:

```text id="qnspgm"
EC2
```

Điều này cho phép EC2 instance sử dụng IAM Role để gọi AWS services.

Nếu sau này backend chạy ở môi trường khác như ECS hoặc Lambda, bạn cần chọn trusted entity tương ứng. Trong workshop này, chọn EC2 là phù hợp nếu backend deploy lên server EC2.

---

#### Bước 3: Chưa cần gắn policy ngay

Ở bước chọn permission, bạn có thể tạm thời bỏ qua hoặc chưa gắn policy ngay.

Các policy chi tiết sẽ được tạo ở các phần tiếp theo:

* S3 Policy.
* SQS + Textract Policy.
* CloudWatch + Secrets Manager Policy.

Sau khi tạo xong các policy, chúng ta sẽ quay lại gắn vào IAM Role.

---

#### Bước 4: Đặt tên IAM Role

Đặt tên role rõ ràng theo dự án:

```text id="kkkdft"
DocuMindBackendWorkerRole
```

Hoặc:

```text id="uhqx3i"
DocMind-Backend-Worker-Role
```

Tên role nên thể hiện rõ role này dùng cho backend và worker của DocuMind AI.

---

#### Bước 5: Thêm mô tả cho Role

Ở phần description, có thể ghi:

```text id="2jknbi"
IAM Role for DocuMind AI backend and worker to access S3, SQS, Textract, CloudWatch and Secrets Manager.
```

---

#### Bước 6: Tạo Role

Kiểm tra lại thông tin:

| Thành phần     | Giá trị đề xuất                    |
| -------------- | ---------------------------------- |
| Trusted entity | AWS Service                        |
| Use case       | EC2                                |
| Role name      | DocuMindBackendWorkerRole          |
| Purpose        | Backend/Worker access AWS services |

Sau đó nhấn:

```text id="zrvbhs"
Create role
```

---

#### Bước 7: Kiểm tra Trust Policy

Sau khi tạo role, mở role vừa tạo và kiểm tra tab:

```text id="6cj6tx"
Trust relationships
```

Trust policy cho EC2 thường có dạng:

```json id="mckoxy"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Policy này cho phép EC2 assume role để lấy quyền gọi AWS services.

---

#### Bước 8: Vì sao không nên hardcode AWS credentials?

Không nên lưu trực tiếp các biến sau trong `.env` production:

```env id="mzpuwu"
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
```

Lý do:

* Dễ bị lộ khi commit nhầm lên GitHub.
* Khó rotate key.
* Không an toàn khi nhiều môi trường cùng dùng chung key.
* Không phù hợp với best practice khi deploy trên AWS.

Thay vào đó, khi backend chạy trên EC2 có gắn IAM Role, AWS SDK sẽ tự lấy credential tạm thời thông qua IAM Role.

---

#### Bước 9: Biến môi trường vẫn cần giữ

Dù dùng IAM Role, backend vẫn cần các biến cấu hình như:

```env id="i0xlcq"
AWS_REGION="ap-southeast-1"
AWS_S3_BUCKET_NAME="documind-document-storage"
AWS_SQS_QUEUE_URL="https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue"
```

Nhưng không cần hardcode access key trong production.

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo IAM Role cho backend/worker.
* Role có trusted entity phù hợp.
* Nếu dùng EC2, trust policy cho phép `ec2.amazonaws.com`.
* Role có tên rõ ràng theo dự án.
* Role sẵn sàng để gắn các policy S3, SQS, Textract, CloudWatch và Secrets Manager.
* Bạn hiểu vì sao không nên hardcode AWS credentials trong production.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có IAM Role nền tảng để cấp quyền cho backend và worker truy cập các dịch vụ AWS một cách an toàn hơn. Các policy chi tiết sẽ được tạo và gắn vào role ở các bước tiếp theo.
