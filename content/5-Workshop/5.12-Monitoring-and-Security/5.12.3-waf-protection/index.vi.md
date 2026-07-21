---

title : "WAF Protection"
date : 2026-06-10
weight : 3
chapter : false
pre : " <b> 5.12.3. </b> "
--------------------------

#### Tổng quan

Trong bước này, chúng ta sẽ cấu hình **AWS WAF** để tăng cường bảo vệ cho hệ thống **DocuMind AI**.

AWS WAF là Web Application Firewall, giúp bảo vệ ứng dụng web khỏi các request độc hại như SQL Injection, Cross-Site Scripting, bot traffic hoặc các request bất thường. Trong DocuMind AI, WAF có thể được gắn vào **CloudFront** hoặc **Application Load Balancer** nếu hệ thống deploy production.

WAF không thay thế hoàn toàn bảo mật backend, nhưng giúp thêm một lớp phòng vệ ở phía trước ứng dụng.

---

#### Mục tiêu của bước này

Sau khi hoàn thành bước này, bạn sẽ:

* Hiểu vai trò của AWS WAF trong DocuMind AI.
* Tạo Web ACL cho ứng dụng.
* Thêm AWS Managed Rules.
* Cấu hình rule cho SQL Injection và XSS.
* Gắn WAF vào CloudFront hoặc ALB nếu có.
* Kiểm tra request bị chặn hoặc được đếm.
* Chuẩn bị log và monitoring cho security events.

---

#### Vai trò của WAF trong DocuMind AI

AWS WAF giúp bảo vệ:

* Login/Register API.
* Document Upload API.
* Admin API.
* AI Chat/RAG API.
* Enterprise Sandbox API.
* Các endpoint có nguy cơ bị abuse.
* Các request chứa payload SQL Injection hoặc XSS.
* Một số bot hoặc request bất thường.

Các loại tấn công có thể giảm rủi ro:

```text id="r3hchh"
SQL Injection
Cross-Site Scripting
Bad bot traffic
Suspicious request patterns
Common web exploits
```

---

#### Kiến trúc WAF Protection

WAF thường được đặt trước ứng dụng:

```text id="s85fhi"
User
  |
  v
AWS WAF
  |
  v
CloudFront hoặc Application Load Balancer
  |
  v
Backend API / Frontend
```

![waf-protection](/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.3-waf-protection/waf-protection.png)

> ⚠️ **Gợi ý chụp hình (Screenshot Suggestion):**
> Bạn hãy chụp màn hình AWS WAF Web ACL đã tạo và phần Associated AWS resources nếu có gắn với CloudFront hoặc ALB.
> Lưu ảnh tại đường dẫn:
> `/static/images/5-Workshop/5.12-Monitoring-and-Security/5.12.3-waf-protection/waf-protection.png`

---

#### Lưu ý quan trọng

AWS WAF không gắn trực tiếp vào backend Node.js/Express.

WAF thường được gắn vào:

```text id="07ck60"
Amazon CloudFront
Application Load Balancer
API Gateway
AppSync
```

Trong workshop này, nếu bạn chưa có CloudFront hoặc ALB, có thể tạo Web ACL và mô tả cách gắn vào resource production. Khi triển khai thực tế, nên đặt backend sau ALB hoặc CloudFront để WAF bảo vệ request đi vào.

---

#### Bước 1: Truy cập AWS WAF

Vào AWS Console, tìm:

```text id="rn5w3e"
WAF
```

Chọn:

```text id="s0w2iu"
AWS WAF & Shield
```

Sau đó chọn:

```text id="ymcxyy"
Web ACLs
```

Nhấn:

```text id="d88mt8"
Create web ACL
```

---

#### Bước 2: Cấu hình Web ACL

Đặt tên:

```text id="pt36st"
docmind-web-acl
```

Description:

```text id="avq4l9"
Web ACL for DocuMind AI frontend and backend protection.
```

Chọn resource type phù hợp:

Nếu dùng CloudFront:

```text id="y9r978"
Amazon CloudFront distributions
```

Nếu dùng ALB:

```text id="oxmgnn"
Regional resources
```

Chọn region đang dùng nếu là regional resource.

---

#### Bước 3: Associated AWS resources

Nếu đã có CloudFront hoặc ALB, chọn resource để gắn WAF.

Ví dụ:

```text id="agv43y"
CloudFront Distribution
```

hoặc:

```text id="6v6dc3"
Application Load Balancer
```

Nếu chưa có resource, có thể bỏ qua trong demo và gắn sau khi deploy production.

---

#### Bước 4: Thêm AWS Managed Rules

Chọn:

```text id="rczfkd"
Add managed rule groups
```

Nên thêm các rule group:

```text id="sdguyv"
AWSManagedRulesCommonRuleSet
AWSManagedRulesKnownBadInputsRuleSet
AWSManagedRulesSQLiRuleSet
AmazonIpReputationList
```

Ý nghĩa:

| Rule Group             | Mục đích                      |
| ---------------------- | ----------------------------- |
| CommonRuleSet          | Chặn các web exploit phổ biến |
| KnownBadInputsRuleSet  | Chặn payload đầu vào độc hại  |
| SQLiRuleSet            | Phát hiện SQL Injection       |
| AmazonIpReputationList | Chặn IP có danh tiếng xấu     |

---

#### Bước 5: Chọn Count mode trước khi Block

Khi mới cấu hình, nên chạy rule ở chế độ:

```text id="bgv2s0"
Count
```

để quan sát request nào bị match rule trước.

Sau khi kiểm tra không ảnh hưởng request hợp lệ, có thể đổi sang:

```text id="56fhg5"
Block
```

Điều này giúp tránh việc WAF chặn nhầm traffic hợp lệ của hệ thống.

---

#### Bước 6: Default Web ACL action

Chọn default action:

```text id="u947l1"
Allow
```

Sau đó các rule cụ thể sẽ Count hoặc Block request bất thường.

---

#### Bước 7: Bật CloudWatch metrics

Trong phần visibility, bật:

```text id="hdtj85"
CloudWatch metrics
Sampled requests
```

Metric name đề xuất:

```text id="v16cjl"
docmind-web-acl
```

Việc bật metrics giúp Admin kiểm tra số request bị allow, count hoặc block.

---

#### Bước 8: Tạo Web ACL

Kiểm tra lại cấu hình:

* Web ACL name đúng.
* Resource type đúng.
* Rule groups đã thêm.
* Default action là Allow.
* Metrics đã bật.

Sau đó nhấn:

```text id="n21ks7"
Create web ACL
```

---

#### Bước 9: Test SQL Injection request

Có thể test request chứa SQLi pattern:

```text id="1fyasu"
/api/auth/login?email=' OR '1'='1
```

Hoặc body test:

```json id="ft5uz2"
{
  "email": "' OR '1'='1",
  "password": "test"
}
```

Nếu rule đang ở Count mode, request có thể vẫn đi qua nhưng được ghi nhận trong WAF metrics.

Nếu rule ở Block mode, request có thể bị chặn.

---

#### Bước 10: Test XSS request

Test payload:

```text id="eq3q75"
<script>alert('xss')</script>
```

Ví dụ request:

```json id="tf90eg"
{
  "question": "<script>alert('xss')</script>"
}
```

WAF có thể phát hiện payload bất thường tùy rule group.

---

#### Bước 11: Kiểm tra WAF Metrics

Vào:

```text id="n8t1jq"
CloudWatch
→ Metrics
→ AWS/WAFV2
```

Kiểm tra các metric:

```text id="6o4493"
AllowedRequests
BlockedRequests
CountedRequests
```

Nếu có request match rule, metric sẽ tăng.

---

#### Bước 12: Kiểm tra Sampled Requests

Trong Web ACL, chọn:

```text id="9u2b3i"
Sampled requests
```

Tại đây bạn có thể xem request nào đã match rule.

Không nên hiển thị dữ liệu nhạy cảm khi chụp hình báo cáo.

---

#### Các lỗi thường gặp

| Lỗi                              | Nguyên nhân                    | Cách xử lý                         |
| -------------------------------- | ------------------------------ | ---------------------------------- |
| WAF không chặn request           | Chưa attach vào CloudFront/ALB | Kiểm tra Associated AWS resources  |
| Không thấy metrics               | Chưa bật CloudWatch metrics    | Bật visibility config              |
| Request hợp lệ bị chặn           | Rule quá strict                | Đổi sang Count hoặc thêm exception |
| Không gắn được CloudFront        | Sai scope                      | CloudFront dùng Global scope       |
| Không gắn được ALB               | Sai region                     | Chọn đúng region ALB               |
| Backend vẫn nhận request độc hại | WAF chưa nằm trước backend     | Đặt backend sau ALB/CloudFront     |

---

#### Checklist hoàn thành

Bạn đã hoàn thành bước này khi:

* Đã tạo Web ACL cho DocuMind AI.
* Đã thêm AWS Managed Rules.
* Đã bật CloudWatch metrics.
* Đã bật Sampled requests.
* Đã gắn WAF vào CloudFront/ALB nếu có.
* Test SQLi/XSS được WAF count hoặc block.
* Biết cách chuyển từ Count mode sang Block mode.
* Admin có thể theo dõi metrics WAF.

---

#### Kết quả kỳ vọng

Sau bước này, DocuMind AI có thêm lớp bảo vệ web bằng AWS WAF. Hệ thống có thể phát hiện hoặc chặn các request bất thường, hỗ trợ bảo vệ frontend/backend và tăng khả năng giám sát bảo mật trong môi trường production.
