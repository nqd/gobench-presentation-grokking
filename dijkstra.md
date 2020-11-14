## 1. Giới thiệu Gobench

Ở Veriksystems, chúng tôi làm việc với các hệ thống Internet of Things (IoT).
Phát triển chức năng mới.
Benchmark để nhìn thấy hiệu năng của hệ thống trước khi đưa ra thị trường.

Bản chất của hệ thống IoT là sử dụng protocol. Bên cạnh HTTP, chúng tôi thường
sử dụng MQTT, và NATS. Chương trình benchmark cần phải hỗ trợ nhiều loại
protocol khác nhau và có thể dễ dàng mở rộng.

Mục tiêu
(1) Expressive
(2) Nhiều loại protocol khác nhau 
(3) Kết quả thời gian thực
(4) Scalable hỗ trợ đến 1 triệu kết nối đồng thời cho các protocol đòi hỏi có
consistant connection như MQTT hay NATS


Bài báo này được chia thành các mục như sau.
Mục 2 giới thiệu cơ chế hoạt động của Gobench.
Chúng tôi trình bày cách implement hệ thống ở Mục 3. Mục 4 so sánh hiệu năng của
Gobench và một số chương trình mã nguồn mở. Chúng tôi chỉ so sánh hiệu năng của
HTTP. Gobench hiện tại vẫn đang được phát triển tích cực, chúng tôi sẽ liệt kê những vấn đề todo ở Mục 5.

## 2. Cơ chế hoạt động

## 3. Thực hiện

## 4. Hiệu năng

## 5. Dự định

