Gobench, một benchmark framework

Tác giả Nguyễn Quốc Đính, Phạm Đức Thọ

## 1. Giới thiệu Gobench

Ở Veriksystems, chúng tôi làm việc với các hệ thống Internet of Things (IoT).
Phát triển chức năng mới.
Benchmark để nhìn thấy hiệu năng của hệ thống trước khi đưa ra thị trường.


Mục tiêu:

1. Expressive: Kịch bản benchmark phải đủ phức phức tạp để thể hiện các luồng
chương trình khác nhau. Các công cụ có dạng `./tool [options]
http://auth@host/path#hash` sẽ không đáp ứng được nhu cầu.

2. Nhiều loại protocol khác nhau.
Bản chất của hệ thống IoT là sử dụng protocol. Bên cạnh HTTP, chúng tôi thường
sử dụng MQTT, và NATS. Chương trình benchmark cần phải hỗ trợ nhiều loại
protocol khác nhau và có thể dễ dàng mở rộng.

3. Kết quả thời gian thực: Các thông số benchmark phải được hiển thị theo thời
gian, và tốt nhất nằm trên biểu đồ. Việc chỉ tóm gọn kết quả cuối cùng làm mất
đi việc quan sát tính cale up/down trong quá trình benchmark.

4. Scalable hỗ trợ đến 1 triệu kết nối đồng thời cho các protocol đòi hỏi có
consistant connection như MQTT hay NATS. Trong trường hợp chương trình client
tạo một kết nối (dựa trên TCP) đến một endpoint, địa chỉ kết nối này được thể
hiện bởi bốn thông số <IP nguồn, port nguồn, IP đích, port đích>, do đó số lượng
kết nối bị giới hạn bởi con số 65536. Để đạt được mục tiêu 1M kết nối,  phải có
một cựm (cluster) cá client phối hợp với nhau.

Các chương trình benchmark hiện tại trên thị trường chưa đáp ứng được các yêu
cầu nêu trên, do đó chung tôi tự xây dựng một framework có tên Gobench. Gobench
hiện tại đạt được các mục tiêu (1), (2) và (3), và được host tại
https://github.com/gobench-io/gobench. Chúng tôi đang tích cực phát triển chức
năng (4) cho bản release v0.1.0.

Bài báo này được chia thành các mục như sau.
Mục 2 giới thiệu cơ chế hoạt động của Gobench.
Chúng tôi trình bày cách implement hệ thống ở Mục 3. Mục 4 so sánh hiệu năng của
Gobench và một số chương trình mã nguồn mở trong việc sử dụng HTTP client. Gobench hiện tại vẫn đang được phát triển tích cực, chúng tôi sẽ liệt kê những vấn đề todo ở Mục 5.

## 2. Cơ chế hoạt động

Cũng như các chương trình benchmark khác, Gobench là một hoặc nhiều client tấn
công vào một đối tượng (target) cần kiểm tra. Vấn đề đặt ra là kịch bản cho
client hoạt động như thế nào. Có thể đơn giản là options cho CLI như hey, ab,
vegeta; hoặc XML như Jmeter; hoặc domain specific language (DSL) như Gatling,
MZBench; hoặc ngôn ngữ lập trình phổ biến như Nodejs ở k6, Python ở Locust.

Với Gobench, CLI option không đáp ứng được nhu cầu

Trở ngại của DSL là người dùng phải học ngôn ngữ mô tả mới, và DSL bộc lộ hạn
chế khi kịch bản mô phỏng trở nên phức tạp.

### 2.1. Kịch bản test

Vì Gobench được xây dựng trên Go, kịch bản cũng được viết bằng Go. ...

```
package main

import (
	"context"
	"log"
	"time"

	httpClient "github.com/gobench-io/gobench/clients/http"
	"github.com/gobench-io/gobench/executor/scenario"
)

func export() scenario.Vus {
	return scenario.Vus{
		{
			Nu:   12,
			Rate: 100,
			Fu:   f,
		},
	}
}

func f(ctx context.Context, vui int) {
	for {
		log.Println("tic")
		time.Sleep(1 * time.Second)
	}
}
```

Một kịch bản phải định nghĩa một `func export()` trả về một mảng các virtual
user (Vu). `Vu` cho phép tester định nghĩa hành vi của một người dùng trong
chương trình test.

Mỗi `vu` được định nghĩa bởi ba thông số :
1. `Nu` là tổng số lượng user cho loại `vu` này.
2. `Rate` là tốc độ khởi tạo user của lớp `vu` này. Gobench không tạo tất cả
   user cùng một lúc, mà tuần tự dựa trên phân bố Poisson. Trong ví dụ trên 12
   user được khởi tạo với lamda = 100. Như vậy trung bình sau 0.12 giây toàn bộ
   user của lớp `vu` này sẽ được tạo xong.
3. `Fu` định nghĩa hành vi của một `vu`. Ở ví dụ trên, user chi đơn giản in tic
   sau mỗi một giây. 

Mỗi user trong `vu` sẽ được chạy trong mỗi goroutine riêng biệt. Vì goroutine là
một greenthread của go sử dụng ít tài nguyên, nên Gobench có khả năng tạo được
một lượng lớn các user trong mỗi host.

### 2.2. Master, Agent, và Executor

Gobench khi nhận được một ngữ cảnh sẽ đưa vào hàng đợi, và thực thi tuần tự.
Hình 1 mô tả cách thức hoạt động của Gobench. Để tiện theo dõi, chúng tôi giới
thiệu các khái niệm được sử dụng trong hệ thống như sau.

<img src="./gobench-model.svg" alt="gobench model" style="width: 100%;"/>

Hình 1: Mô hình hoạt động của Gobench

### 2.3. Master

### 2.4. Agent

### 2.5. Executor


## 3. Thực hiện

## 4. Hiệu năng

## 5. Dự định

