---
title: "Worklog Tuần 8"
date: 2026-06-08
weight: 8
chapter: false
pre: " <b> 1.8. </b> "
---


### Mục tiêu tuần 8:

* Tìm hiểu dịch vụ AWS Identity and Access Management (IAM).
* Hiểu cơ chế xác thực và phân quyền người dùng trên AWS.
* Thực hành tạo User, Group, Role và Policy.
* Áp dụng các nguyên tắc bảo mật cơ bản trong quá trình quản lý tài khoản AWS.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------- | ----------------------------------------- |
| 2 | - Tìm hiểu tổng quan về **AWS Identity and Access Management (IAM)** <br> - Vai trò của IAM trong việc quản lý tài khoản AWS <br> - Tìm hiểu nguyên tắc **Least Privilege** trong bảo mật hệ thống | 08/06/2026 | 08/06/2026 | <https://docs.aws.amazon.com/iam/> |
| 3 | - Thực hành tạo **IAM User** <br>&emsp; + Đăng nhập bằng IAM User <br>&emsp; + Thiết lập mật khẩu <br>&emsp; + Quản lý Access Key <br>&emsp; + Cấu hình Multi-Factor Authentication (MFA) | 09/06/2026 | 10/06/2026 | <https://docs.aws.amazon.com/IAM/latest/UserGuide/> |
| 4 | - Tìm hiểu **IAM Group** và **IAM Policy** <br>&emsp; + Tạo Group <br>&emsp; + Gán User vào Group <br>&emsp; + Gắn Managed Policy <br>&emsp; + Tạo Custom Policy đơn giản | 11/06/2026 | 12/06/2026 | <https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html> |
| 5 | - Thực hành **IAM Role** <br>&emsp; + Tạo IAM Role cho EC2 <br>&emsp; + Tìm hiểu Trust Policy <br>&emsp; + Gắn Role cho EC2 Instance <br>&emsp; + Kiểm tra quyền truy cập tài nguyên AWS | 13/06/2026 | 13/06/2026 | <https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html> |
| 6 | - Ôn tập toàn bộ kiến thức về IAM <br> - Ghi chép quy trình tạo User, Group, Role và Policy <br> - Tổng hợp các nguyên tắc bảo mật khi quản trị tài khoản AWS | 14/06/2026 | 14/06/2026 | <https://explore.skillbuilder.aws/> |

### Kết quả đạt được tuần 8:

* Hiểu được vai trò của AWS IAM trong việc quản lý danh tính và phân quyền người dùng trên nền tảng AWS.

* Phân biệt được các thành phần chính của IAM gồm:
  * IAM User
  * IAM Group
  * IAM Role
  * IAM Policy

* Tạo thành công nhiều IAM User phục vụ cho việc quản lý tài khoản theo từng mục đích sử dụng.

* Biết cách tạo và quản lý IAM Group để cấp quyền cho nhiều người dùng cùng lúc.

* Hiểu cách sử dụng Managed Policy và Custom Policy để kiểm soát quyền truy cập tài nguyên AWS.

* Thực hành thành công việc tạo IAM Role và gán Role cho EC2 Instance nhằm cấp quyền truy cập dịch vụ AWS mà không cần sử dụng Access Key.

* Hiểu được nguyên tắc **Least Privilege**, chỉ cấp các quyền cần thiết nhằm tăng cường bảo mật hệ thống.

* Làm quen với việc thiết lập **Multi-Factor Authentication (MFA)** để nâng cao mức độ an toàn cho tài khoản AWS.

* Hoàn thành tài liệu hướng dẫn quản lý người dùng và phân quyền trên AWS, tạo nền tảng cho việc xây dựng hệ thống Serverless an toàn ở các tuần tiếp theo.

* Nâng cao nhận thức về bảo mật và quản lý quyền truy cập trong môi trường điện toán đám mây.