Gobench, một benchmark framework

*Tác giả: Nguyễn Quốc Đính, Phạm Đức Thọ*

19/11/2020

## 1. Giới thiệu Gobench

Ở Veriksystems, chúng tôi làm việc với các hệ thống Internet of Things (IoT).
Khi phát triển hệ thống mới, chúng tôi có nhu cầu benchmark để kiểm tra tính bền
vững (robustness), tính sẵn dùng (availability), và xử lý lỗi (error handling)
dưới tải lớn.

Đặt tính của hệ thống IoT là có nhiều loại protocol khác nhau, và thường giữ kết
nối giữa client và server. Sau khảo sát thấy các công cụ mã nguồn mở hiện tại
chưa đáp ứng được nhu cầu, chúng tôi xây dựng Gobech một framework mã nguồn mở
cho benchmark bằng golang. Gobech được xây dựng với bốn mục tiêu sau:

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

Vào thời điểm viết bài báo nào, Gobench đã đạt được các mục tiêu (1), (2) và
(3), và được host tại https://github.com/gobench-io/gobench. Chúng tôi đang tích
cực phát triển chức năng (4) cho bản release v0.1.0.

Bài báo này được chia thành các mục như sau. Mục 2 giới thiệu cơ chế hoạt động
của Gobench. Chúng tôi trình bày cách implement hệ thống ở Mục 3. Mục 4 so sánh
hiệu năng của Gobench và một số chương trình mã nguồn mở trong việc sử dụng HTTP
client. Gobench hiện tại vẫn đang được phát triển tích cực, chúng tôi sẽ liệt kê
những vấn đề todo ở Mục 5.

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
		log.Println("tictoc")
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
3. `Fu` định nghĩa hành vi của một `vu`. Ở ví dụ trên, user chi đơn giản in
   tictoc sau mỗi một giây. 

Mỗi user trong `vu` sẽ được chạy trong mỗi goroutine riêng biệt. Vì goroutine là
một greenthread của go sử dụng ít tài nguyên, nên Gobench có khả năng tạo được
một lượng lớn các user trong mỗi host.

### 2.2. Master, Agent, và Executor

Gobench khi nhận được một kịch bản sẽ đưa vào hàng đợi và thực thi tuần tự. Hình
1 mô tả cách thức hoạt động của Gobench. Để tiện theo dõi, chúng tôi giới thiệu
các khái niệm được sử dụng trong hệ thống như sau.

<img src="./gobench-model.svg" alt="gobench model" style="width: 100%;"/>

Hình 1: Mô hình hoạt động của Gobench.

#### 2.2.1. Master

Mỗi hệ thống Gobench có một master. Như tên gọi của nó, master là điều phối viên
của hệ thống. Đầu tiên, master là nơi giao tiếp với tester thông qua Web UI hoặc
HTTP API. Master tuần tự lấy các kịch bản ra để thực thi. Vào mỗi thời điểm, chỉ
một kịch bản hoạt động.

<img src="./gobench-compile-scenario.svg" alt="gobench compile senario"
class="center" style="width: 60%;">

Hình 2: Master tạo nên Executor từ kịch bản.

Để chạy một kịch bản, master trước tiên dịch kịch bản thành một file thực thi
gọi là Executor như Hình 2, sau đó gởi file này đến các agent trong cluster. Mỗi
agent nhận được một công việc (job) bao gồm (1) file thực thi, (2) `Nu_{i}`,
`Rate_{i}` là số lượng user và tốc độ khởi tạo tương ứng cho agent `i`. Dĩ nhiên
tổng `Nu_{i}`, `Rate_{i}` phải bằng Nu và Rate tương ứng. Một cách đơn giản,
Gobench chia điều Nu và Rate trên tổng số các agent trong hệ thống. Một cách
phức tạp hơn `Nu_{i}` sẽ tỉ lệ thuận với tài nguyên (CPU, RAM, băng thông) của
một agent. Chúng tôi chọn cách đơn giản trong implement của Gobench.

Khi một kịch bản được thực thi, các agent sẽ báo báo các metric về master.
Master lưu kết quả này vào trong cơ sở dữ liệu nhúng là sqlite3. Chúng tôi chọn
sqlite3 vì CSDL này đi kèm với chương trình, đơn giản khi vận hành. Nghi ngại về
tốc độ ghi của sqlite3 sẽ không thành vấn đề khi metrics được giản lượt hóa
trước khi đưa về master (chúng tôi sẽ đề cập kỹ hơn trong phần sau). Là loại SQL
cũng giúp cho việc truy vấn thuận tiện hơn các loại NoSQL nhúng khác.

Master là single point of failure (SPOF) của hệ thống. Sẽ rất dễ dàng kiểm tra
trạng thái hoạt động của master. Với việc chỉ có duy nhất một master, khả năng
master chết rất ít khi xảy ra. Nếu master chết, một instance mới có thể được
dựng lên, bất kì job nào đang chạy sẽ bị hủy. Tester có thể chạy lại kịch bản
này nếu muốn.

#### 2.2.2. Agent

Mỗi hệ thống Gobench có một hoặc nhiều Agent. Agent có thể chạy trên bất cứ hệ
thống Unix nào. Agent giữ liên lạc với master để tạo nên cluster. Khi agent nhận
job từ master, nó sẽ chạy file executor trong một thread riêng biệt. Agent và
Executor liên lạc với nhau thông qua Unix socket. 

Agent đóng vai trò trung gian trong việc báo cáo metrics từ Executor đến Master.
Và ở chiều ngược lại, trong quá trình hoạt động, nếu Agent nhận lệnh hủy một
job, nó sẽ giết Executor thread.

Trên nền Unix socket, giao tiếp giữa Agent và Executor là gRPC như Hình 3.

<img src="./gobench-agent-executor.svg" alt="gobench model" class="center"
style="width: 60%;">

Hình 3: Giao tiếp giữa Agent và Executor.

#### 2.2.3. Executor

Executor khởi chạy bởi Agent khi thực thi một job mới. Mỗi host sẽ chạy hoặc là
Master hoặc là Agent, và mỗi Agent chỉ tạo chạy duy nhất một Executor. Nguyên
nhân cho việc này là benchmark thường sử dụng nhiều tài nguyên (CPU, RAM, băng
thông mạng). Tester nên cài đặt để Agent và Executor sử dụng hết các tài nguyên
nó đang có.

Executor là nơi chạy kịch bản benchmark. Các kịch bản này thường sử dụng các thư
viện (hiện tại là HTTP client, MQTT client, NATs client) của Gobench tương tác
với đối tượng cần benchmark. Cứ mỗi hành vi của một client sẽ được ghi nhận lại
trong metrics. Ví dụ với HTTP thì số lượt request thành công, thất bại, hay độ
trễ (delay) của request (ns) được báo cáo đến metrics collector ở ngay bên trong
Executor. Executor tổng hợp (aggregate) các metric này trước khi chuyển đến
Agent và cuối cùng tập hợp về Master mỗi 10s. Chúng tôi chọn phương pháp này để
giảm số lượng message gởi về Master.

Các metric Gobench hỗ trợ là counter, histogram, và gauge.

<!-- ## 3. Thực hiện -->
### 2.3. Tạo một client mới

Một trong những mục tiêu hàng đầu của Gobench là hỗ trợ nhiều protocol khác
nhau. Để đạt được điều này, API để hỗ trợ client mới phải đơn giản.

Gobench cung cấp hai API để một client báo cáo kết quả về như bên đưới. API đầu
tiên khai báo các metric mà client này hỗ trợ. Ví dụ với MQTT có số lượng các
kết nối thành công, thất bại, độ trễ kết nối, số lượng message pub và sub với
các QoS bằng 0, 1, hoặc 2.

```go
import (
    "github.com/gobench-io/gobench/executor"
)

executor.Setup(groups)
executor.Notify(metric-id, value)
```

API thứ hai dùng để báo cáo kết quả của một metric nào đấy (đã được đăng ký ở
API đầu tiên).

Việc hỗ trợ client mới dễ dàng đang là thế mạnh của Gobench. Thời gian sắp tới
chúng tôi sẽ hỗ trợ một số giao thức phổ biến khác như gRPC, websocket, graphQL.

## 3. Hiệu năng

[Tiếp tục]

## 4. Kết luận

Có rất nhiều chương trình benchmark vẫn đang được tích cực phát triển. Một số
theo hướng CLI như Wrk, Apachebench, Drill, Hey, Vegeta. Một số chỉ hỗ trợ HTTP
như Locust, k6.

Chương trình gần nhất đạt được bốn mục tiêu ban đầu chúng tôi đưa ra có thể kể
đến là MZBench, Gatling, và Artillery. MZBench là một hệ thống rất thú vị được
viết bởi Machine Zone với Erlang. MZBench scaling rất tốt nhờ vào hỗ trợ của
OTP; đáng tiếc là các tác giả đã ngưng phát triển MZBench, chương trình tải về
từ Github bị lỗi không chạy được. Gobench chịu nhiều ảnh hưởng của MZBench về
thiết kế. Gatling phát triển với Scala và hỗ trợ nhiều loại protocol khác nhau.
Bản Community tuy vậy chỉ cho phép chạy trên một node. Cả MZBench và Gatling cho
phép viết kịch bản bằng DSL riêng. Artillery phát triển bởi công ty Shoreditch
Ops bằng NodeJS; kịch bản có thể viết bằng Javascript, do đó khá dễ nắm bắt,
giống Golang. Tuy nhiên chỉ với bản premium thì mới chạy phân tán trên nhiều
node được. Và chính vì sử dụng NodeJS nên chương trình không tận dụng được nhiều
core của benchmark client.

[Một vài kết luận về hiệu năng, sau khi hoàn thành Mục 3]

Cho đến thời điểm viết bài báo này Gobench đã đạt được ba trong bốn mục tiêu ban
đầu được đặt ra là (1) expressive, (2) hỗ trợ nhiều protocol là HTTP, MQTT,
NATs, và (3) Kết qủa thời gian thực được hiện lên dashboard.

Chúng tôi đã sử dung Gobench để benchmark một hệ thống IoT khá phức tạp, bao gồm
cả HTTP và MQTT trong mỗi `vu`. Benchmark client chỉ sử dung 100% CPU, với một
số thời điểm tăng lên 250% CPU của tổng 800% CPU (8 core) khi số lượng `vu` =
10000.

Nếu chỉ chạy trong một node thì khả năng benchmark sẽ bị giới hạn, trong thời
gian sắp tới chúng tôi sẽ xây dựng mục tiêu số (4) là tạo ra scalable benchmark
có khả năng tạo đến 1 triệu kết nối đồng thời.

Thêm các loại client phổ biến khác như gRPC, websocket, graphQL cũng nằm trong
danh sách cần phải làm. 