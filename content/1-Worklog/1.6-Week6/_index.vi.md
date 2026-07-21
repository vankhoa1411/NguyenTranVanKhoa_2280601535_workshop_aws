---
title: "Worklog Tuần 6"
date: 2026-05-25
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---


### Mục tiêu tuần 6:

* Thực hành nâng cao với dịch vụ Amazon EC2.
* Hiểu cơ chế hoạt động của Security Group và cách bảo mật máy chủ trên AWS.
* Thực hành kết nối SSH đến EC2 Instance.
* Triển khai một ứng dụng web đơn giản lên máy chủ EC2.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2 | - Ôn tập kiến thức về Amazon EC2 <br> - Tạo EC2 Instance mới để phục vụ thực hành <br> - Lựa chọn Amazon Linux 2 AMI và cấu hình Instance Type phù hợp với AWS Free Tier | 25/05/2026 | 25/05/2026 | <https://docs.aws.amazon.com/ec2/> |
| 3 | - Tìm hiểu và cấu hình **Security Group** <br>&emsp; + Mở cổng SSH (22) <br>&emsp; + Mở cổng HTTP (80) <br>&emsp; + Mở cổng HTTPS (443) <br>&emsp; + Quy tắc Inbound và Outbound | 26/05/2026 | 27/05/2026 | <https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html> |
| 4 | - Thực hành kết nối SSH đến EC2 bằng Terminal/PowerShell <br>&emsp; + Sử dụng Key Pair (.pem) <br>&emsp; + Kiểm tra trạng thái máy chủ <br>&emsp; + Thực hiện các lệnh Linux cơ bản | 28/05/2026 | 28/05/2026 | <https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-to-linux-instance.html> |
| 5 | - Cài đặt môi trường Web Server trên EC2 <br>&emsp; + Cập nhật hệ điều hành <br>&emsp; + Cài đặt Apache/Nginx <br>&emsp; + Tạo trang web HTML đơn giản <br>&emsp; + Kiểm tra truy cập qua địa chỉ Public IP | 29/05/2026 | 30/05/2026 | <https://docs.aws.amazon.com/> |
| 6 | - Kiểm tra lại toàn bộ quá trình triển khai <br> - Ghi chép các bước cấu hình EC2 và Security Group <br> - Tổng hợp các lỗi gặp phải và cách khắc phục trong quá trình thực hành | 31/05/2026 | 31/05/2026 | <https://explore.skillbuilder.aws/> |

### Kết quả đạt được tuần 6:

* Hiểu quy trình tạo và cấu hình một máy chủ Amazon EC2 phục vụ triển khai ứng dụng.

* Nắm được chức năng của **Security Group** trong việc kiểm soát lưu lượng truy cập đến và đi từ EC2 Instance.

* Cấu hình thành công các cổng dịch vụ cần thiết như SSH (22), HTTP (80) và HTTPS (443).

* Kết nối thành công đến EC2 Instance thông qua giao thức SSH bằng Key Pair.

* Làm quen với các thao tác quản trị cơ bản trên hệ điều hành Linux như cập nhật gói phần mềm, tạo thư mục và quản lý tệp tin.

* Cài đặt và cấu hình thành công Web Server (Apache/Nginx) trên EC2.

* Triển khai thành công một trang web HTML đơn giản và truy cập được thông qua địa chỉ Public IP của EC2.

* Biết cách kiểm tra trạng thái hoạt động của EC2 Instance và xử lý một số lỗi thường gặp khi kết nối SSH hoặc truy cập Website.

* Hoàn thiện tài liệu hướng dẫn triển khai EC2 và Web Server để làm cơ sở cho việc xây dựng hệ thống Cloud ở các tuần tiếp theo.

* Nâng cao kỹ năng quản trị máy chủ Linux và triển khai ứng dụng trên nền tảng AWS Cloud.