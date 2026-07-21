---

title : "Secrets Manager"
date : 2026-06-10
weight : 2
chapter : false
pre : " <b> 5.12.2. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ cấu hình **AWS Secrets Manager** cho hệ thống **DocuMind AI**.

Secrets Manager được sử dụng để lưu trữ các thông tin nhạy cảm như `DATABASE_URL`, `JWT_SECRET`, `OPENAI_API_KEY`, `GEMINI_API_KEY` và các cấu hình production khác. Thay vì lưu trực tiếp các secret này trong file `.env` production hoặc hardcode trong source code, backend có thể đọc secret từ AWS Secrets Manager khi khởi động.

Cách này giúp hệ thống an toàn hơn, dễ quản lý secret hơn và giảm nguy cơ lộ thông tin khi deploy.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Tạo secret cho DocuMind AI trong AWS Secrets Manager.
* Lưu các biến nhạy cảm vào secret.
* Cấu hình backend đọc secret khi chạy production.
* Kiểm tra IAM Role có quyền đọc secret.
* Tránh commit `.env` thật lên GitHub.
* Chuẩn bị môi trường production an toàn hơn.

---

#### Vai trò của Secrets Manager trong DocuMind AI

Secrets Manager được dùng để lưu:

```text id="8nmy4f"
DATABASE_URL
JWT_SECRET
OPENAI_API_KEY
GEMINI_API_KEY
AI_PROVIDER
AWS_SQS_QUEUE_URL
```

Có thể lưu thêm:

```text id="yg9tns"
SMTP_HOST
SMTP_USER
SMTP_PASSWORD
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
```

Nếu hệ thống có email OTP hoặc Google OAuth.

---

#### Kiến trúc Secrets Manager

Luồng đọc secret:

```text id="ug46x8"
Backend / Worker
  |
  v
IAM Role
  |
  v
AWS Secrets Manager
  |
  v
Load secrets into application config
```

![secrets-manager](/images/5-Workshop/5.12-Monitoring-and-Security/5.12.2-secrets-manager/secrets-manager.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình secret `docmind/backend` trong AWS Secrets Manager, nhưng che các giá trị nhạy cảm trước khi đưa vào báo cáo.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.2-secrets-manager/secrets-manager.png`

---

#### Bước 1: Truy cập Secrets Manager

Đăng nhập AWS Console, tìm:

```text id="q3wkia"
Secrets Manager
```

Chọn:

```text id="27uynp"
AWS Secrets Manager
```

Nhấn:

```text id="g7epx2"
Store a new secret
```

---

#### Bước 2: Chọn loại secret

Chọn:

```text id="gwtfhj"
Other type of secret
```

Sau đó chọn dạng key/value để nhập các biến môi trường.

---

#### Bước 3: Thêm secret key/value

Thêm các biến nhạy cảm:

```json id="kd3zu5"
{
  "DATABASE_URL": "postgresql://username:password@host:5432/docmind",
  "JWT_SECRET": "your-jwt-secret",
  "OPENAI_API_KEY": "your-openai-api-key",
  "GEMINI_API_KEY": "your-gemini-api-key",
  "AI_PROVIDER": "openai"
}
```

Không nên đưa giá trị thật vào tài liệu báo cáo. Khi chụp hình, cần che secret value.

---

#### Bước 4: Đặt tên secret

Tên secret đề xuất:

```text id="l9gzz0"
docmind/backend
```

Hoặc:

```text id="m3be15"
docmind/production
docmind/ai-keys
```

Description:

```text id="igjyxg"
Secrets for DocuMind AI backend production environment.
```

---

#### Bước 5: Cấu hình rotation

Với workshop/demo, có thể tạm thời không bật rotation:

```text id="va2djl"
Disable automatic rotation
```

Trong production, có thể bật rotation cho một số loại secret nếu có quy trình phù hợp.

---

#### Bước 6: Tạo secret

Kiểm tra lại thông tin, sau đó nhấn:

```text id="o4v3ik"
Store
```

Sau khi tạo, mở secret và copy Secret name:

```text id="r9g4zk"
docmind/backend
```

---

#### Bước 7: Cấu hình backend dùng Secrets Manager

Trong `.env` production, chỉ cần giữ:

```env id="pycfw1"
USE_SECRETS_MANAGER="true"
AWS_SECRET_NAME="docmind/backend"
AWS_REGION="ap-southeast-1"
```

Không cần lưu trực tiếp:

```env id="ulbuxw"
OPENAI_API_KEY=""
GEMINI_API_KEY=""
JWT_SECRET=""
DATABASE_URL=""
```

---

#### Bước 8: Cài AWS SDK cho Secrets Manager

Nếu backend dùng Node.js:

```bash id="h6sqyh"
npm install @aws-sdk/client-secrets-manager
```

---

#### Bước 9: Tạo service đọc secret

Tạo file:

```text id="xdoil8"
src/services/secrets.service.ts
```

Ví dụ:

```ts id="3kuv7z"
import {
  SecretsManagerClient,
  GetSecretValueCommand
} from "@aws-sdk/client-secrets-manager";

const client = new SecretsManagerClient({
  region: process.env.AWS_REGION
});

export async function loadSecretsFromManager() {
  if (process.env.USE_SECRETS_MANAGER !== "true") {
    return {};
  }

  const secretName = process.env.AWS_SECRET_NAME;

  if (!secretName) {
    throw new Error("AWS_SECRET_NAME is required when USE_SECRETS_MANAGER=true");
  }

  const command = new GetSecretValueCommand({
    SecretId: secretName
  });

  const response = await client.send(command);

  if (!response.SecretString) {
    throw new Error("SecretString is empty");
  }

  return JSON.parse(response.SecretString);
}
```

---

#### Bước 10: Load secret khi app khởi động

Trong file start app, ví dụ `server.ts`:

```ts id="d3yjnr"
import dotenv from "dotenv";
import { loadSecretsFromManager } from "./services/secrets.service";

dotenv.config();

async function bootstrap() {
  const secrets = await loadSecretsFromManager();

  for (const [key, value] of Object.entries(secrets)) {
    if (!process.env[key]) {
      process.env[key] = String(value);
    }
  }

  console.log("[SECRETS_LOADED]");

  // start app here
}

bootstrap();
```

Không nên log giá trị secret. Chỉ log rằng secret đã được load thành công.

---

#### Bước 11: Kiểm tra quyền IAM

Backend/worker cần quyền:

```text id="jdpqb9"
secretsmanager:GetSecretValue
```

Policy đã được cấu hình ở phần IAM Role and Policy.

Test bằng AWS CLI:

```bash id="dpy08m"
aws secretsmanager get-secret-value \
  --secret-id docmind/backend \
  --region ap-southeast-1
```

Nếu thành công, IAM Role có quyền đọc secret.

---

#### Quy tắc bảo mật secret

Cần tuân thủ:

* Không commit `.env` thật lên GitHub.
* Không hardcode API key trong source code.
* Không log secret ra console hoặc CloudWatch.
* Chỉ cấp quyền `GetSecretValue` cho role cần dùng.
* Secret name nên có prefix rõ ràng như `docmind/`.
* Khi chụp hình báo cáo, phải che secret value.
* Nếu nghi ngờ lộ key, cần rotate key ngay.

---

#### Test Secrets Manager

Các bước test:

1. Tạo secret `docmind/backend`.
2. Thêm `OPENAI_API_KEY`, `GEMINI_API_KEY`, `JWT_SECRET`.
3. Cấu hình `.env` production với `USE_SECRETS_MANAGER=true`.
4. Gắn IAM Role có quyền đọc secret.
5. Start backend.
6. Kiểm tra log có `[SECRETS_LOADED]`.
7. Test API AI Analysis.
8. Xác nhận backend dùng key từ Secrets Manager.

---

#### Các lỗi thường gặp

| Lỗi                         | Nguyên nhân                     | Cách xử lý                       |
| --------------------------- | ------------------------------- | -------------------------------- |
| `AccessDeniedException`     | IAM Role thiếu quyền đọc secret | Kiểm tra policy `GetSecretValue` |
| `ResourceNotFoundException` | Sai secret name                 | Kiểm tra `AWS_SECRET_NAME`       |
| `SecretString is empty`     | Secret không có value           | Kiểm tra secret content          |
| JSON parse error            | Secret không đúng JSON          | Lưu secret dạng key/value JSON   |
| Backend không load secret   | `USE_SECRETS_MANAGER` chưa bật  | Kiểm tra `.env`                  |
| Lộ secret trong log         | Log nhầm object secrets         | Không log secret value           |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo secret trong AWS Secrets Manager.
* Secret chứa các biến nhạy cảm cần thiết.
* Backend có thể đọc secret khi khởi động.
* IAM Role có quyền `secretsmanager:GetSecretValue`.
* Không còn lưu API key thật trong `.env` production.
* Không log secret ra console hoặc CloudWatch.
* AI provider có thể hoạt động bằng secret được load.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI đã có cơ chế quản lý secret an toàn hơn bằng AWS Secrets Manager. Backend có thể đọc các thông tin nhạy cảm khi chạy production mà không cần hardcode trong source code hoặc file `.env`.
