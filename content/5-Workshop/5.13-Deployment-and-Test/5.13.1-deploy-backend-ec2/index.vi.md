---

title : "Deploy Backend lên EC2"
date : 2026-06-10
weight : 1
chapter : false
pre : " <b> 5.13.1. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ deploy **Backend API và Worker** của hệ thống **DocuMind AI** lên **Amazon EC2**.

Backend API chịu trách nhiệm nhận request từ frontend, xử lý authentication, upload tài liệu lên Amazon S3, gửi message vào Amazon SQS và cung cấp API cho Dashboard. Worker chịu trách nhiệm nhận message từ SQS, gọi Amazon Textract OCR, gửi OCR text sang Gemini/OpenAI và lưu kết quả vào PostgreSQL.

Việc deploy backend lên EC2 giúp hệ thống có thể chạy ổn định trên môi trường server thay vì chỉ chạy local.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo hoặc chuẩn bị EC2 instance cho backend.
* Cài đặt Node.js, Git và các công cụ cần thiết.
* Clone source code backend lên EC2.
* Cấu hình file môi trường production.
* Cài đặt dependencies.
* Build backend TypeScript.
* Chạy Backend API và Worker bằng PM2.
* Kiểm tra API backend hoạt động từ server.

---

#### Kiến trúc deploy backend

Luồng deploy:

```text id="deploy-flow"
Developer
  |
  v
Git Repository
  |
  v
Amazon EC2
  |
  |-- Backend API
  |-- SQS Worker
  |
  v
AWS Services + PostgreSQL + AI Providers
```

![deploy-backend-ec2](/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.1-deploy-backend-ec2/deploy-backend-ec2.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình EC2 instance đang chạy, terminal SSH vào EC2 và PM2 hiển thị backend/worker đang online.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.13-Deployment-and-Test/5.13.1-deploy-backend-ec2/deploy-backend-ec2.png`

---

#### Bước 1: Chuẩn bị EC2 instance

Trong AWS Console, tạo EC2 instance với cấu hình đề xuất:

| Thành phần     | Cấu hình đề xuất                       |
| -------------- | -------------------------------------- |
| AMI            | Ubuntu Server 22.04 LTS hoặc 24.04 LTS |
| Instance type  | t2.micro hoặc t3.micro cho demo        |
| Storage        | 20GB trở lên                           |
| Security Group | Mở port SSH và backend port            |
| IAM Role       | `DocuMindBackendWorkerRole`            |

Port cần mở:

```text id="ports"
22    SSH
3000  Backend API
80    HTTP nếu dùng Nginx
443   HTTPS nếu dùng SSL
```

Nếu backend chỉ test demo, có thể mở port `3000`. Nếu production, nên dùng Nginx reverse proxy qua port `80/443`.

---

#### Bước 2: SSH vào EC2

Từ máy local, SSH vào EC2:

```bash id="ssh-ec2"
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

Nếu file key bị lỗi permission, chạy:

```bash id="chmod-key"
chmod 400 your-key.pem
```

---

#### Bước 3: Cập nhật server

Sau khi SSH vào EC2:

```bash id="update-server"
sudo apt update
sudo apt upgrade -y
```

Cài các package cơ bản:

```bash id="install-basic"
sudo apt install git curl unzip build-essential -y
```

---

#### Bước 4: Cài Node.js

Cài Node.js LTS:

```bash id="install-node"
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

Kiểm tra version:

```bash id="check-node"
node -v
npm -v
```

---

#### Bước 5: Cài PM2

PM2 giúp chạy backend và worker như process nền:

```bash id="install-pm2"
sudo npm install -g pm2
```

Kiểm tra:

```bash id="check-pm2"
pm2 -v
```

---

#### Bước 6: Clone source code backend

Clone project từ Git:

```bash id="clone-project"
git clone https://github.com/your-username/documind-ai.git
```

Di chuyển vào thư mục backend:

```bash id="cd-backend"
cd documind-ai/backend
```

Nếu tên folder của bạn khác, hãy kiểm tra bằng:

```bash id="ls-folder"
ls
```

> ⚠️ **Lưu ý:**
> Nếu gặp lỗi `cd documind-ai/backend: No such file or directory`, nghĩa là tên folder hoặc cấu trúc repo khác với ví dụ. Hãy dùng `ls` để xem đúng tên thư mục rồi `cd` vào đúng folder backend.

---

#### Bước 7: Cài dependencies

Trong thư mục backend:

```bash id="npm-install"
npm install
```

Nếu có lỗi dependency, có thể thử:

```bash id="npm-ci"
npm ci
```

---

#### Bước 8: Tạo file `.env.production`

Tạo file môi trường:

```bash id="create-env"
nano .env.production
```

Ví dụ nội dung:

```env id="env-production"
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

JWT_SECRET="your-jwt-secret"
```

Nếu dùng AWS Secrets Manager:

```env id="env-secrets"
NODE_ENV="production"
PORT=3000

USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

---

#### Bước 9: Build backend

Nếu backend dùng TypeScript:

```bash id="build-backend"
npm run build
```

Nếu build thành công, thường sẽ có thư mục:

```text id="dist-folder"
dist/
```

---

#### Bước 10: Chạy Prisma migration production

Nếu database production chưa có bảng, chạy:

```bash id="prisma-deploy"
npx prisma migrate deploy
```

Generate Prisma Client:

```bash id="prisma-generate"
npx prisma generate
```

Không nên dùng `npx prisma migrate dev` trên production.

---

#### Bước 11: Chạy Backend API bằng PM2

Ví dụ nếu file build là `dist/server.js`:

```bash id="pm2-backend"
pm2 start dist/server.js --name documind-backend
```

Nếu backend dùng npm script:

```bash id="pm2-npm"
pm2 start npm --name documind-backend -- start
```

Kiểm tra process:

```bash id="pm2-list"
pm2 list
```

---

#### Bước 12: Chạy SQS Worker bằng PM2

Nếu worker build ra `dist/workers/document.worker.js`:

```bash id="pm2-worker"
pm2 start dist/workers/document.worker.js --name documind-worker
```

Hoặc dùng npm script:

```bash id="pm2-worker-npm"
pm2 start npm --name documind-worker -- run worker
```

Kiểm tra log:

```bash id="pm2-logs"
pm2 logs documind-backend
pm2 logs documind-worker
```

---

#### Bước 13: Cho PM2 tự chạy khi reboot

Chạy:

```bash id="pm2-startup"
pm2 startup
```

Sau đó chạy lệnh mà PM2 gợi ý.

Lưu danh sách process:

```bash id="pm2-save"
pm2 save
```

---

#### Bước 14: Test backend health check

Gọi API health:

```bash id="curl-health"
curl http://localhost:3000/api/health
```

Nếu test từ bên ngoài:

```text id="public-health"
http://your-ec2-public-ip:3000/api/health
```

Response mong đợi:

```json id="health-response"
{
  "success": true,
  "message": "DocuMind AI backend is running"
}
```

---

#### Bước 15: Kiểm tra IAM Role trên EC2

Chạy:

```bash id="sts-check"
aws sts get-caller-identity
```

Kết quả mong đợi có dạng:

```text id="assumed-role"
assumed-role/DocuMindBackendWorkerRole
```

Nếu không có, kiểm tra lại IAM Role đã gắn vào EC2 chưa.

---

#### Các lỗi thường gặp

| Lỗi                                   | Nguyên nhân                                 | Cách xử lý                        |
| ------------------------------------- | ------------------------------------------- | --------------------------------- |
| Không SSH được                        | Sai key hoặc security group chưa mở port 22 | Kiểm tra key pair và inbound rule |
| `cd` không đúng folder                | Tên folder khác repo                        | Dùng `ls` để kiểm tra             |
| `npm install` lỗi                     | Thiếu Node.js hoặc dependency conflict      | Kiểm tra Node version             |
| `npm run build` lỗi                   | TypeScript/config lỗi                       | Kiểm tra log build                |
| Backend không truy cập được port 3000 | Security group chưa mở port                 | Mở inbound TCP 3000               |
| `DATABASE_URL` lỗi                    | Sai thông tin PostgreSQL                    | Kiểm tra kết nối database         |
| AWS `AccessDenied`                    | IAM Role thiếu policy                       | Kiểm tra phần 5.11                |
| Worker không chạy                     | Sai file path hoặc script                   | Kiểm tra `package.json`           |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* EC2 instance đã chạy.
* SSH vào EC2 thành công.
* Node.js, Git và PM2 đã được cài đặt.
* Backend source code đã được clone.
* `.env.production` đã được cấu hình.
* Backend build thành công.
* Prisma migration production chạy thành công.
* Backend API chạy bằng PM2.
* SQS Worker chạy bằng PM2.
* API health check hoạt động.
* EC2 đang dùng đúng IAM Role.

---

#### Kết quả kỳ vọng

Sau bước này, Backend API và SQS Worker của DocuMind AI đã chạy trên EC2. Hệ thống sẵn sàng kết nối với frontend, Amazon S3, Amazon SQS, Amazon Textract, PostgreSQL, Gemini/OpenAI và các dịch vụ monitoring/security.
