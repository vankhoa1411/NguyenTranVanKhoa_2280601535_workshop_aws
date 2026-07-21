---

title : "Các lỗi thường gặp"
date : 2026-06-10
weight : 4
chapter : false
pre : " <b> 5.13.4. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ tổng hợp các **lỗi thường gặp** khi deploy và kiểm tra hệ thống **DocuMind AI**.

Các lỗi có thể xảy ra ở nhiều phần khác nhau như EC2, Node.js backend, PostgreSQL, Prisma, Amazon S3, Amazon SQS, Amazon Textract, Gemini/OpenAI, IAM Role, CloudWatch, CORS hoặc frontend dashboard.

Việc tổng hợp lỗi giúp quá trình debug nhanh hơn và giúp người triển khai biết nên kiểm tra thành phần nào trước.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Biết các lỗi thường gặp khi deploy backend.
* Biết cách kiểm tra lỗi EC2 và PM2.
* Biết cách xử lý lỗi database và Prisma.
* Biết cách xử lý lỗi AWS permission.
* Biết cách kiểm tra lỗi S3, SQS, Textract.
* Biết cách xử lý lỗi Gemini/OpenAI.
* Biết cách kiểm tra lỗi frontend và CORS.
* Có checklist debug full workflow.

---

#### Kiến trúc debug lỗi

Khi gặp lỗi, nên kiểm tra theo luồng:

```text id="debug-flow"
Frontend
  |
  v
Backend API
  |
  v
PostgreSQL + Prisma
  |
  v
Amazon S3
  |
  v
Amazon SQS
  |
  v
Worker
  |
  v
Amazon Textract
  |
  v
Gemini / OpenAI
  |
  v
Dashboard Result
```

![common-errors](/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.4-common-errors/common-errors.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình terminal hoặc CloudWatch Logs hiển thị lỗi thực tế và phần xử lý lỗi tương ứng trong báo cáo.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.4-common-errors/common-errors.png`

---

#### Lỗi EC2 và server

| Lỗi                                  | Nguyên nhân                                       | Cách xử lý                                     |
| ------------------------------------ | ------------------------------------------------- | ---------------------------------------------- |
| Không SSH được vào EC2               | Sai key, sai user, Security Group chưa mở port 22 | Kiểm tra key pair, user `ubuntu`, inbound rule |
| Backend không truy cập được từ ngoài | Port 3000 chưa mở                                 | Mở inbound TCP 3000 hoặc dùng Nginx            |
| Server hết RAM                       | Instance quá nhỏ hoặc worker nặng                 | Dùng swap, nâng instance type                  |
| Lệnh `node` không tồn tại            | Chưa cài Node.js                                  | Cài Node.js LTS                                |
| `git clone` lỗi                      | Repo private hoặc thiếu quyền                     | Kiểm tra GitHub token/SSH key                  |

Kiểm tra EC2:

```bash id="ec2-check"
df -h
free -m
pm2 list
pm2 logs
```

---

#### Lỗi PM2

| Lỗi                                  | Nguyên nhân                      | Cách xử lý                  |
| ------------------------------------ | -------------------------------- | --------------------------- |
| PM2 process stopped                  | App crash                        | Xem `pm2 logs`              |
| Backend không tự chạy lại sau reboot | Chưa `pm2 startup` và `pm2 save` | Chạy lại PM2 startup        |
| Worker không chạy                    | Sai file worker hoặc script      | Kiểm tra `package.json`     |
| Log quá nhiều                        | Không cấu hình log rotate        | Cài `pm2-logrotate` nếu cần |

Lệnh kiểm tra:

```bash id="pm2-debug"
pm2 list
pm2 logs documind-backend
pm2 logs documind-worker
pm2 restart documind-backend
pm2 restart documind-worker
```

---

#### Lỗi Node.js và build

| Lỗi                   | Nguyên nhân                           | Cách xử lý                              |
| --------------------- | ------------------------------------- | --------------------------------------- |
| `npm install` lỗi     | Dependency conflict                   | Thử `npm ci` hoặc kiểm tra Node version |
| `npm run build` lỗi   | TypeScript error                      | Kiểm tra log build                      |
| `Cannot find module`  | Build thiếu file hoặc import sai path | Kiểm tra import path                    |
| `.env` không đọc được | Chưa load dotenv hoặc sai file        | Kiểm tra config loader                  |
| `PORT already in use` | Port đã bị process khác dùng          | Kill process hoặc đổi port              |

Kiểm tra port:

```bash id="check-port"
sudo lsof -i :3000
```

Kill process nếu cần:

```bash id="kill-port"
sudo kill -9 process_id
```

---

#### Lỗi PostgreSQL và Prisma

| Lỗi                       | Nguyên nhân                 | Cách xử lý                       |
| ------------------------- | --------------------------- | -------------------------------- |
| `P1001`                   | Không kết nối được database | Kiểm tra PostgreSQL đang chạy    |
| `P1000`                   | Sai username/password       | Kiểm tra `DATABASE_URL`          |
| `database does not exist` | Chưa tạo database           | Tạo database `docmind`           |
| `relation does not exist` | Chưa chạy migration         | Chạy `npx prisma migrate deploy` |
| Prisma Client lỗi         | Chưa generate client        | Chạy `npx prisma generate`       |
| Migration lỗi production  | Dùng sai lệnh               | Dùng `npx prisma migrate deploy` |

Kiểm tra database:

```bash id="db-check"
psql "postgresql://docmind:docmind_password@localhost:5432/docmind"
```

Chạy Prisma:

```bash id="prisma-debug"
npx prisma validate
npx prisma generate
npx prisma migrate deploy
```

---

#### Lỗi Amazon S3

| Lỗi                      | Nguyên nhân                        | Cách xử lý                           |
| ------------------------ | ---------------------------------- | ------------------------------------ |
| `AccessDenied`           | IAM Role thiếu quyền               | Kiểm tra S3 Policy                   |
| `NoSuchBucket`           | Sai bucket name                    | Kiểm tra `AWS_S3_BUCKET_NAME`        |
| `PermanentRedirect`      | Sai region bucket                  | Kiểm tra `AWS_REGION`                |
| File không upload        | Backend lỗi hoặc S3 PutObject fail | Kiểm tra backend log                 |
| Object URL không mở được | Bucket private                     | Đây là hành vi đúng nếu không public |

Test S3:

```bash id="s3-debug"
aws s3 ls s3://documind-document-storage
aws s3 cp ./test.pdf s3://documind-document-storage/test/test.pdf
```

---

#### Lỗi Amazon SQS

| Lỗi                       | Nguyên nhân                      | Cách xử lý                   |
| ------------------------- | -------------------------------- | ---------------------------- |
| `QueueDoesNotExist`       | Sai Queue URL hoặc region        | Kiểm tra `AWS_SQS_QUEUE_URL` |
| `AccessDenied`            | Thiếu quyền SQS                  | Kiểm tra SQS Policy          |
| Worker không nhận message | Worker chưa chạy hoặc queue rỗng | Kiểm tra PM2 và SQS Console  |
| Message xử lý lặp         | Worker không gọi `DeleteMessage` | Kiểm tra worker logic        |
| Message vào DLQ           | Worker lỗi nhiều lần             | Kiểm tra DLQ và worker log   |

Test SQS:

```bash id="sqs-debug"
aws sqs send-message \
  --queue-url "https://sqs.ap-southeast-1.amazonaws.com/account-id/docmind-document-processing-queue" \
  --message-body '{"documentId":"test","action":"PROCESS_DOCUMENT"}'
```

---

#### Lỗi Amazon Textract

| Lỗi                            | Nguyên nhân                                        | Cách xử lý                            |
| ------------------------------ | -------------------------------------------------- | ------------------------------------- |
| `AccessDeniedException`        | Thiếu quyền Textract                               | Kiểm tra Textract Policy              |
| `InvalidS3ObjectException`     | Sai bucket/key hoặc Textract không đọc được object | Kiểm tra S3 object và quyền GetObject |
| `UnsupportedDocumentException` | File không hỗ trợ                                  | Dùng PDF, PNG, JPG, JPEG              |
| OCR text rỗng                  | File mờ hoặc không có chữ                          | Dùng file test rõ hơn                 |
| Region error                   | Textract không hỗ trợ region hoặc sai region       | Kiểm tra `AWS_REGION`                 |

Định dạng nên dùng:

```text id="textract-format"
PDF
PNG
JPG
JPEG
```

Không nên dùng trực tiếp:

```text id="textract-not-support"
DOCX
XLSX
PPTX
```

---

#### Lỗi Gemini/OpenAI

| Lỗi                  | Nguyên nhân          | Cách xử lý                   |
| -------------------- | -------------------- | ---------------------------- |
| `401 Unauthorized`   | OpenAI API key sai   | Kiểm tra `OPENAI_API_KEY`    |
| `API key not valid`  | Gemini API key sai   | Kiểm tra `GEMINI_API_KEY`    |
| `429 Rate limit`     | Vượt quota           | Retry hoặc fallback provider |
| `insufficient_quota` | Tài khoản hết quota  | Kiểm tra billing/quota       |
| `model_not_found`    | Sai model name       | Kiểm tra model trong `.env`  |
| JSON parse error     | AI trả markdown/text | Siết prompt hoặc clean JSON  |
| Timeout              | AI phản hồi chậm     | Tăng timeout hoặc fallback   |

Log cần kiểm tra:

```text id="ai-log"
[AI_PROVIDER_SELECTED]
[OPENAI_REQUEST_STARTED]
[GEMINI_REQUEST_STARTED]
[AI_PROVIDER_FALLBACK_USED]
[AI_ANALYSIS_FAILED]
```

---

#### Lỗi IAM Role và Permission

| Lỗi                             | Nguyên nhân                           | Cách xử lý                 |
| ------------------------------- | ------------------------------------- | -------------------------- |
| `Unable to locate credentials`  | EC2 chưa gắn IAM Role                 | Attach IAM Role vào EC2    |
| `AccessDenied`                  | Policy thiếu quyền                    | Kiểm tra policy tương ứng  |
| `aws sts` không ra assumed role | AWS CLI đang dùng credential khác     | Kiểm tra credential chain  |
| Secret không đọc được           | Thiếu `secretsmanager:GetSecretValue` | Kiểm tra Secrets Policy    |
| CloudWatch không ghi log        | Thiếu `logs:PutLogEvents`             | Kiểm tra CloudWatch Policy |

Kiểm tra identity:

```bash id="iam-debug"
aws sts get-caller-identity
```

---

#### Lỗi CloudWatch Logs

| Lỗi                 | Nguyên nhân                    | Cách xử lý                 |
| ------------------- | ------------------------------ | -------------------------- |
| Không có log group  | Chưa tạo hoặc sai region       | Tạo `/docmind/application` |
| Không có log stream | App chưa tạo stream            | Kiểm tra logger config     |
| Không ghi được log  | Thiếu IAM permission           | Kiểm tra logs policy       |
| Log lộ secret       | Logger log toàn bộ env/request | Mask dữ liệu nhạy cảm      |

Không log:

```text id="no-secret-log"
password
JWT token
OPENAI_API_KEY
GEMINI_API_KEY
DATABASE_URL password
AWS credentials
```

---

#### Lỗi CORS và frontend

| Lỗi                      | Nguyên nhân                        | Cách xử lý                   |
| ------------------------ | ---------------------------------- | ---------------------------- |
| CORS error               | Backend chưa allow frontend domain | Kiểm tra `CORS_ORIGIN`       |
| API URL sai              | Frontend trỏ sai backend URL       | Kiểm tra env frontend        |
| 401 Unauthorized         | Token thiếu hoặc hết hạn           | Login lại hoặc refresh token |
| 403 Forbidden            | User không có quyền document       | Kiểm tra ownership           |
| Dashboard không cập nhật | Chưa polling status                | Kiểm tra Document Status API |
| Upload fail trên UI      | Field file sai hoặc backend lỗi    | Kiểm tra Network tab         |

Frontend env ví dụ:

```env id="frontend-env"
VITE_API_BASE_URL="http://your-ec2-public-ip:3000"
```

---

#### Lỗi WAF

| Lỗi                              | Nguyên nhân                 | Cách xử lý                            |
| -------------------------------- | --------------------------- | ------------------------------------- |
| WAF không chặn request           | Chưa attach CloudFront/ALB  | Kiểm tra Associated resources         |
| Không thấy metric                | Chưa bật CloudWatch metrics | Kiểm tra visibility config            |
| Request hợp lệ bị chặn           | Rule quá strict             | Chuyển sang Count hoặc thêm exception |
| Backend vẫn nhận payload độc hại | WAF chưa nằm trước backend  | Đặt backend sau ALB/CloudFront        |

---

#### Checklist debug nhanh

Khi hệ thống lỗi, kiểm tra theo thứ tự:

```text id="debug-checklist"
1. Backend API health
2. PM2 backend logs
3. PM2 worker logs
4. DATABASE_URL và Prisma
5. IAM Role bằng aws sts
6. S3 bucket và object
7. SQS queue và message
8. Textract OCR log
9. Gemini/OpenAI API key
10. Document status trong database
11. Frontend API base URL
12. CloudWatch Logs
```

---

#### Lệnh debug nhanh

```bash id="quick-debug"
pm2 list
pm2 logs documind-backend
pm2 logs documind-worker

aws sts get-caller-identity
aws s3 ls s3://documind-document-storage

npx prisma validate
npx prisma generate
npx prisma migrate deploy
```

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Biết cách kiểm tra lỗi EC2/PM2.
* Biết cách kiểm tra lỗi backend build.
* Biết cách kiểm tra PostgreSQL/Prisma.
* Biết cách kiểm tra S3/SQS/Textract.
* Biết cách kiểm tra Gemini/OpenAI.
* Biết cách kiểm tra IAM Role.
* Biết cách kiểm tra CloudWatch Logs.
* Biết cách xử lý CORS/frontend.
* Có checklist debug full workflow.

---

#### Kết quả kỳ vọng

Sau bước này, bạn có tài liệu tổng hợp lỗi thường gặp cho quá trình deploy và test DocuMind AI. Khi hệ thống gặp lỗi, bạn có thể nhanh chóng xác định vấn đề nằm ở frontend, backend, database, AWS service, AI provider hay IAM permission.
