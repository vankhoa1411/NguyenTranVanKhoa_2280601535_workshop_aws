---
title: "Worklog Tuần 11"
date: 2026-06-29
weight: 11
chapter: false
pre: " <b> 1.11. </b> "
---


### Mục tiêu tuần 11:

* Phát triển quy trình xử lý tài liệu tự động trên nền tảng AWS.
* Xây dựng chức năng tải tệp từ Frontend lên Amazon S3.
* Tích hợp AWS Lambda và Amazon Textract để xử lý tài liệu.
* Kiểm thử quy trình xử lý dữ liệu theo mô hình Serverless.

### Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------ | --------------- | ----------------------------------------- |
| 2 | - Xây dựng chức năng tải ảnh và tệp PDF từ Frontend lên **Amazon S3** <br> - Kiểm tra việc tạo Bucket và lưu trữ tệp <br> - Thực hiện kiểm thử quá trình Upload với nhiều định dạng tài liệu | 29/06/2026 | 29/06/2026 | https://docs.aws.amazon.com/AmazonS3/latest/userguide/ |
| 3 | - Cấu hình **S3 Event Notification** <br>&emsp; + Kích hoạt sự kiện khi có tệp mới được tải lên <br>&emsp; + Liên kết Event với AWS Lambda <br>&emsp; + Kiểm tra hoạt động của Event Notification | 30/06/2026 | 01/07/2026 | https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html |
| 4 | - Phát triển **AWS Lambda Function** <br>&emsp; + Tiếp nhận thông tin từ S3 Event <br>&emsp; + Đọc thông tin Bucket và Object Key <br>&emsp; + Ghi log quá trình xử lý <br>&emsp; + Xử lý các trường hợp lỗi khi đọc dữ liệu | 02/07/2026 | 03/07/2026 | https://docs.aws.amazon.com/lambda/latest/dg/ |
| 5 | - Tích hợp **Amazon Textract** để trích xuất nội dung văn bản từ ảnh và PDF <br>&emsp; + Gửi yêu cầu phân tích tài liệu <br>&emsp; + Nhận kết quả OCR <br>&emsp; + Kiểm tra độ chính xác của dữ liệu trích xuất | 04/07/2026 | 04/07/2026 | https://docs.aws.amazon.com/textract/ |
| 6 | - Kiểm thử toàn bộ quy trình xử lý <br>&emsp; + Upload tài liệu <br>&emsp; + Kích hoạt Lambda <br>&emsp; + Thực hiện OCR bằng Textract <br>&emsp; + Kiểm tra dữ liệu đầu ra <br> - Ghi nhận kết quả và khắc phục các lỗi phát sinh | 05/07/2026 | 05/07/2026 | AWS Documentation |

### Kết quả đạt được tuần 11:

* Xây dựng thành công chức năng tải ảnh và tài liệu PDF từ giao diện người dùng lên Amazon S3.

* Cấu hình thành công **S3 Event Notification** để tự động kích hoạt AWS Lambda mỗi khi có tài liệu mới được tải lên.

* Phát triển thành công hàm AWS Lambda tiếp nhận và xử lý sự kiện từ Amazon S3.

* Hiểu được cơ chế giao tiếp giữa Amazon S3 và AWS Lambda trong kiến trúc Serverless.

* Tích hợp thành công dịch vụ Amazon Textract để trích xuất nội dung văn bản từ ảnh và tài liệu PDF.

* Kiểm thử thành công quy trình xử lý tài liệu tự động từ khi người dùng tải tệp lên đến khi hệ thống nhận được kết quả OCR.

* Xử lý và ghi nhận các lỗi thường gặp trong quá trình tích hợp như lỗi quyền truy cập IAM, cấu hình Lambda và Event Notification.

* Hoàn thiện luồng xử lý dữ liệu tự động của hệ thống, tạo nền tảng cho việc tích hợp AI và lưu trữ dữ liệu trong các giai đoạn tiếp theo.

* Nâng cao kỹ năng làm việc với các dịch vụ Serverless của AWS và hiểu rõ quy trình xây dựng một ứng dụng xử lý tài liệu trên nền tảng Cloud.

* Hoàn thành tài liệu hướng dẫn triển khai và kiểm thử chức năng xử lý tài liệu tự động phục vụ cho báo cáo thực tập.