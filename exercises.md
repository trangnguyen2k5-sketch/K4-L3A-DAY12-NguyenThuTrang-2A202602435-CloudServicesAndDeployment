# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng câu hỏi trả lời bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Thu Trang  Mã học viên: 2A202602435

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên Cloud (Railway/Render) mà quên thiết lập biến môi trường `AGENT_API_KEY`:
- Nếu có giá trị mặc định `"changeme"`: Server vẫn khởi động bình thường. Ứng dụng chạy trên Production nhưng người lạ hoặc các bot quét tự động có thể thử key mặc định `"changeme"` để gọi API `/ask`, làm tiêu tốn toàn bộ ngân sách LLM của bạn mà bạn không hề hay biết cho đến khi nhận hóa đơn.
- Với cơ chế "Fail fast" (chết sớm): Ngay lúc ứng dụng khởi động, Pydantic kiểm tra thiếu secret sẽ ném `ValidationError` và làm app crash lập tức. Bạn nhận được thông báo lỗi ngay trên log deploy và bổ sung bí mật kịp thời trước khi dịch vụ mở cho người dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T16:48:16+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}`

Hai việc làm được với log JSON:
1. **Lọc và truy vấn chính xác (Log Aggregation)**: Đẩy log vào các hệ thống tập trung (Datadog/ELK/CloudWatch) để truy vấn cấu trúc, ví dụ: *"Lọc ra các request có cost_usd > 0.01"* hoặc *"Thống kê tổng số token đã dùng theo từng user_id"*.
2. **Cảnh báo tự động (Automated Alerting)**: Đặt luật giám sát dựa trên trường `cost_usd` hoặc `level` để tự động gửi cảnh báo về Telegram/Slack khi phát hiện chi phí bất thường hoặc tỉ lệ lỗi vượt ngưỡng.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1020 MB |
| Multi-stage | ~385 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~635 MB) bao gồm các công cụ biên dịch C/C++ (`gcc`, `make`, `build-essential`), các thư viện phát triển (`header files`), bộ nhớ đệm của `pip`, và các tiện ích hệ điều hành đầy đủ của hình ảnh gốc Debian. Trong Multi-stage build, stage builder chịu trách nhiệm biên dịch và cài thư viện, sau đó stage runtime (`python:3.11-slim`) chỉ copy kết quả sang mà không mang theo bất kỳ công cụ biên dịch thừa nào.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile tối ưu (`COPY requirements.txt` ➔ `RUN pip install` ➔ `COPY . .`): Các layer cài đặt base image và layer `RUN pip install` được giữ nguyên từ cache (`CACHED`). Chỉ có layer `COPY . .` và các lệnh sau đó phải chạy lại, thời gian build chỉ mất khoảng 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần thay đổi 1 ký tự trong code (`app/main.py`), layer `COPY . .` sẽ bị invalid cache. Do Docker hủy cache từ bước bị thay đổi trở đi, lệnh `RUN pip install` bắt buộc phải chạy lại từ đầu, khiến Docker phải tải và cài lại toàn bộ thư viện Python (mất vài phút mỗi lần sửa code).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện tấn công:
1. Kẻ tấn công lợi dụng lỗ hổng thực thi mã từ xa (RCE) hoặc Command Injection trong code Python.
2. Kẻ tấn công thực thi lệnh shell trong container dưới quyền `root` (quyền mặc định của container).
3. Lợi dụng quyền root container kết hợp với lỗ hổng kernel hoặc mount socket Docker (`/var/run/docker.sock`), kẻ tấn công thoát khỏi container (container escape) và chiếm quyền `root` của máy host.

Lệnh `USER appuser` cắt đứt chuỗi tấn công ngay từ **Bước 2**: Tiến trình ứng dụng chạy dưới tài khoản unprivileged `appuser` (UID 10001). Dù kẻ tấn công thực thi được lệnh shell, họ không có quyền root trong container nên không thể can thiệp hệ thống hay khai thác kỹ thuật leo leo quyền lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 request** trong 2 giây liên tiếp.
Giải thích:
- Hạn mức là 10 request/phút đồng hồ (reset counter ở giây 00).
- Người dùng gửi 10 request đầu tiên ở giây `10:00:59` (1 giây trước khi sang phút mới).
- Ở giây `10:01:00`, counter reset về 0. Người dùng gửi tiếp 10 request ở giây `10:01:01`.
- Tổng cộng: Trong 2 giây (từ 10:00:59 đến 10:01:01), người dùng đã gửi 20 request thành công. Thuật toán cửa sổ trượt (sliding window) loại bỏ kẽ hở này bằng cách tính tổng request trong 60 giây liên tục trượt theo thời gian thực.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Khác biệt:
- **Rate Limit**: Giới hạn **số lượng request** trong một khoảng thời gian ngắn (ví dụ: 10 request/phút) để ngăn dồn dập traffic và chống tấn công DOS.
- **Cost Guard**: Giới hạn **tổng chi phí tài chính (USD)** trong một chu kỳ (ví dụ: ngân sách tháng) để tránh bùng nổ hóa đơn LLM.

Tình huống:
1. *Rate limit cho qua nhưng Cost guard chặn*: User chỉ gửi 1 request trong phút đó (đúng hạn mức 10 request/phút), nhưng request chứa prompt cực kỳ dài 50,000 tokens khiến chi phí vượt quá ngân sách tháng $10.0 ➔ Cost guard chặn trả lỗi 402 Payment Required.
2. *Cost guard cho qua nhưng Rate limit chặn*: User mới bắt đầu tháng, chưa tiêu tiền ($0/$10.0 budget), nhưng gửi liên tục 15 request chỉ trong vòng 3 giây ➔ Rate limit chặn ở request thứ 11 trả lỗi 429 Too Many Requests vì tần suất quá nhanh.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện thảm họa xảy ra:
1. Redis mất kết nối trong 30 giây.
2. Endpoint gộp kiểm tra Redis trả lỗi, khiến Liveness Probe của cả 3 container `agent` đều báo unhealthy.
3. Orchestrator (Docker/Kubernetes) cho rằng tiến trình của container bị hỏng ➔ **kill và restart đồng loạt cả 3 container**.
4. Trong thời gian 3 container đang khởi động lại, Redis kết nối thành công trở lại. Tuy nhiên lúc này **không còn container agent nào sống** để xử lý request ➔ toàn bộ hệ thống bị sập hoàn toàn (downtime).
5. Khi tách riêng: `/health` (liveness) chỉ kiểm tra process app, `/ready` (readiness) kiểm tra Redis. Khi Redis mất kết nối 30s, `/ready` báo 503 để Load Balancer ngừng routing (không restart container), và khi Redis trở lại, hệ thống phục hồi tức thì.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless): `history_length` tăng tiến liên tục (0, 2, 4, 6...) ở mọi request, vì cả 3 container agent đều đọc/ghi chung một lịch sử trên Redis.
- Khi lưu trong dict Python (Stateful): `history_length` sẽ **nhảy ngẫu nhiên và không đồng nhất** giữa các lần gọi (ví dụ: request 1 vào container A ➔ 0; request 2 vào B ➔ 0; request 3 vào A ➔ 2; request 4 vào C ➔ 0...). Do mỗi container có vùng nhớ RAM riêng, Load Balancer chia request ngẫu nhiên làm agent bị "mất trí nhớ".

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi**: Gọi Public URL thu được phản hồi `HTTP/2 502 Bad Gateway` (`{"status":"error","code":502,"message":"Application failed to respond"}`).
- **Cách tìm nguyên nhân**: Vào tab **Deployments** của service trên Railway, bấm **View logs** để xem log container. Log hiển thị dòng `Uvicorn running on http://0.0.0.0:8080`, cho thấy Uvicorn tự nhận cổng 8080 từ biến `$PORT` của Railway.
- **Cách sửa**: Vào tab **Settings ➔ Networking** của service trên Railway, thay đổi cấu hình cổng định tuyến (Port) từ `8000` thành `8080`. Sau khi lưu, Railway chuyển hướng traffic vào đúng cổng Uvicorn và ứng dụng phản hồi HTTP 200 OK ngay lập tức.
