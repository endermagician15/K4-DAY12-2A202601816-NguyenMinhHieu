# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng câu trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Minh Hiếu  Mã học viên: 2A202601816

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên Cloud (Render/Railway) mà kỹ sư quên cài đặt biến môi trường `API_TOKEN` trong trang Dashboard quản lý. Nếu có giá trị mặc định `"changeme"`, ứng dụng vẫn khởi động thành công, chấp nhận traffic và trả lời request nhưng người dùng lạ có thể khai thác token mặc định `"changeme"` để sử dụng dịch vụ miễn phí, hoặc ứng dụng âm thầm tiêu tốn chi phí LLM mà không ai phát hiện cho tới khi nhận hóa đơn. Ngược lại, việc "chết sớm" (Fail Fast với ValidationError) khiến container crash lập tức trong lúc deploy, buộc dev phải kiểm tra log và bổ sung secret ngay trước khi có bất kỳ request nào chạm vào hệ thống.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T15:00:00+00:00", "client_id": "sv01", "prompt_tokens": 12, "completion_tokens": 25, "usd_cost": 0.0001}`

Hai việc làm được:
1. Thống kê và truy vấn tự động: Các hệ thống gom log (Cloud Logging, Datadog) có thể parse trường JSON để tính tổng chi phí `usd_cost` và token tiêu tốn theo từng `client_id` theo thời gian thực.
2. Tự động phát hiện bất thường & Cảnh báo (Alerting): Cấu hình bộ lọc dựa trên trường `severity: "ERROR"` hoặc tạo cảnh báo tự động khi tổng `usd_cost` của một client tăng vọt trong thời gian ngắn.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.7 GB (1700 MB) |
| Multi-stage | 306 MB (71.3 MB content) |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~1.4 GB / 1394 MB) bao gồm toàn bộ hệ điều hành Debian đầy đủ của base image `python:3.11`, các công cụ biên dịch mã nguồn (compiler như gcc, g++, build-essential, header files), pip cache và wheel build temporaries không cần thiết ở môi trường runtime. Trong Multi-stage build (`python:3.11-slim`), stage `builder` chịu trách nhiệm biên dịch và cài đặt dependencies vào virtualenv rồi bị loại bỏ hoàn toàn, stage `runner` sử dụng base image `slim` và chỉ copy duy nhất thư mục `/opt/venv` nhẹ gọn nên giảm dung lượng từ 1.7 GB xuống còn 306 MB.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Các layer được dùng lại từ cache: Tất cả các layer cài đặt venv và dependencies (`COPY requirements.txt .`, `RUN pip install ...`). Chỉ có layer `COPY . .` và các bước cấu hình phía sau mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi khi chỉnh sửa bất kỳ file nào trong source code (`app/main.py`), Docker cache của layer `COPY . .` bị invalided (hủy bỏ), buộc Docker phải tải và cài đặt lại toàn bộ thư viện từ `requirements.txt`, làm tăng thời gian build mỗi lần thay đổi code từ vài giây lên vài phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện: Kẻ tấn công khai thác lỗ hổng Remote Code Execution (RCE) hoặc lỗi thư viện trong app Python -> Lệnh thực thi độc hại được chạy với đặc quyền root của container -> Kẻ tấn công thực hiện kỹ thuật container breakout (thoát khỏi sandbox) thông qua lỗ hổng Linux kernel hoặc bind mount socket/volume -> Chiếm quyền điều khiển root trên máy host.
- Lệnh `USER appuser` cắt đứt chuỗi ở ngay bước đầu tiên: Bằng cách hạ quyền thực thi xuống user không có đặc quyền (`appuser` UID 10001), dù kẻ tấn công có khai thác thành công RCE trong Python app thì các lệnh độc hại vẫn bị giới hạn quyền trong container, không thể ghi sửa file hệ thống container hay thực hiện hành vi leo leo quyền trên host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

- Bắt buộc kèm `WWW-Authenticate: Bearer`: Để tuân thủ đúng chuẩn RFC 6750 / HTTP Protocol, thông báo cho client (như trình duyệt, API Gateway) biết chính xác cơ chế xác thực mà server yêu cầu là Bearer Token.
- Trả cùng một thông báo lỗi: Nhằm đảm bảo nguyên tắc bảo mật thông tin (Defense in Depth). Việc phân biệt chi tiết "thiếu header", "sai scheme" hay "token không tồn tại" sẽ tạo sơ hở cho kẻ tấn công thực hiện kỹ thuật thăm dò (enumeration) để đoán định cấu trúc token hoặc trạng thái của hệ thống.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

- Số request gửi được: Đúng 10 request liên tiếp trước khi bị HTTP 429.
- Nếu bỏ `min(capacity, ...)`: Client im lặng 10 phút sẽ tích trữ được `10 (ban đầu) + 10 * 10 (nạp thêm) = 110` token. Do đó client sẽ gửi được 110 request liên tiếp. Việc bỏ `min` làm mất khả năng khống chế dung lượng tối đa của xô, khiến Rate Limiter không thể bảo vệ hệ thống khỏi các đợt bùng nổ traffic (Burst traffic).

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- Hạn mức $30/tháng: Thiệt hại tối đa là toàn bộ $30 ngay trong đêm (từ 2h sáng). Service bị chặn 402 và chỉ có thể tự hồi phục vào đầu tháng tiếp theo (có thể mất gần 30 ngày).
- Hạn mức $1/ngày: Thiệt hại tối đa bị khoanh vùng ở $1. Service bị chặn 402 trong ngày nhưng tự động khôi phục và phục vụ trở lại vào 00:00 UTC ngày hôm sau.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis gặp sự cố rớt kết nối trong 30 giây.
2. Endpoint `/healthz` (khi gộp) thất bại và trả về mã lỗi 503 Not Ready.
3. Orchestrator (Kubernetes/Docker) lầm tưởng cả 3 container ứng dụng bị treo/hỏng process nên tự động Kill và Restart cả 3 container cùng lúc.
4. Hệ thống rơi vào trạng thái sập hoàn toàn (Total Downtime) vì không còn container nào chạy.
5. Khi Redis phục hồi sau 30s, các container vẫn đang khởi động lại, làm kéo dài thời gian sự cố vô lý.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Thông báo lỗi: Giao diện Web Dashboard hiển thị trạng thái Liveness Probe (/healthz) là `Unreachable` dù cURL vào server vẫn trả `200 OK`.
- Nguyên nhân: Kiểm tra DevTools Console phát hiện trình duyệt Brave Shields (hoặc tiện ích AdBlocker) chặn fetch request `/healthz?_t=...` do khớp với danh sách lọc telemetry (EasyPrivacy).
- Cách sửa: Tắt tính năng Brave Shields cho domain ứng dụng `day12-chat-7x6y.onrender.com` hoặc kiểm tra trực tiếp qua cURL terminal.

