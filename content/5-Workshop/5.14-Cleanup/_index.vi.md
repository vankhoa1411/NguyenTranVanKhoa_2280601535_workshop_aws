---
title : "Dọn dẹp tài nguyên"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

#### Dọn dẹp tài nguyên sau workshop

Sau khi hoàn thành thử nghiệm hệ thống **DocuMind AI**, việc dọn dẹp các tài nguyên trên AWS là vô cùng quan trọng để tránh phát sinh chi phí ngoài ý muốn trong tài khoản của bạn.

---

#### Quy trình dọn dẹp chi tiết

1. **Dọn dẹp Cơ sở dữ liệu PostgreSQL cục bộ**:
   - Mở terminal hoặc client SQL và kết nối vào cơ sở dữ liệu local.
   - Chạy lệnh SQL để xóa database thử nghiệm:
     ```sql
     DROP DATABASE documind;
     ```
   - (Tùy chọn) Dừng dịch vụ PostgreSQL nếu không còn nhu cầu sử dụng:
     ```bash
     brew services stop postgresql
     ```

![DROP DATABASE documind](/images/5-Workshop/5.14-Cleanup/pg-delete.png)

---

2. **Xóa Amazon S3 Bucket**:
   - Truy cập **S3 Console**, chọn bucket `documind-assets-<your-name>`.
   - Nhấn **Empty** để xóa sạch các tệp tài liệu bên trong trước, nhập `permanently delete` để xác nhận.
   - Quay lại danh sách Bucket, chọn bucket đó và nhấn **Delete**, nhập tên bucket để xác nhận xóa hẳn.

 ![documind-assets-<your-name>](/images/5-Workshop/5.14-Cleanup/s3-delete.png)

---

3. **Xóa Amazon SQS Queue**:
   - Truy cập **SQS Console**, chọn hàng đợi `documind-processing-queue`.
   - Nhấn **Delete** ở thanh menu trên cùng và xác nhận xóa.

---

4. **Xóa các cấu hình bảo mật khác**:
   - **AWS Secrets Manager**: Chọn secret `documind/config`, nhấn **Actions > Delete secret**, đặt thời gian chờ tối thiểu (7 days) để hệ thống lên lịch xóa.
   - **AWS WAF**: Chọn Web ACL của bạn, nhấn **Delete** (Lưu ý disassociate khỏi API Gateway/Load Balancer trước khi xóa).

---

#### Tổng kết workshop
Xin chúc mừng bạn đã hoàn thành thiết kế, tích hợp và bảo mật hệ thống **DocuMind AI** trên môi trường AWS Cloud! Qua bài thực hành này, bạn đã làm chủ luồng dữ liệu trích xuất thông tin thông minh sử dụng kiến trúc bất đồng bộ bền bỉ.
