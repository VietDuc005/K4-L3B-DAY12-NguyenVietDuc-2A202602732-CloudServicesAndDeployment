# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: điền câu trả lời chi tiết bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Viết Đức  Mã học viên: 2A202602732

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi triển khai ứng dụng lên môi trường Cloud mới (Render/Railway/K8s), lập trình viên có thể sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard hoặc secret manager. Nếu trường này có giá trị mặc định (như `"changeme"`), service vẫn khởi động thành công và container báo trạng thái Healthy. Hậu quả là API endpoint `/ask` sẽ vận hành công khai với secret mặc định mà ai cũng đoán được. Kẻ tấn công hoặc bot tự động quét trên Internet có thể gửi `X-API-Key: changeme` để gọi LLM tùy ý, gây rò rỉ dữ liệu hoặc đốt sạch ngân sách API. Ngược lại, với cơ chế Fail-fast (không có mặc định), Pydantic sẽ ném `ValidationError` ngay lúc khởi động làm container dừng ngay lập tức. DevOps sẽ phát hiện lỗi thiếu biến môi trường ngay trong log deploy và bổ sung kịp thời trước khi service phục vụ bất kỳ traffic công khai nào.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thực tế thu được từ hệ thống:
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T12:28:49.123456+00:00", "user_id": "sv-123", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.000029}
```

Hai việc làm được với log structured JSON mà lệnh print dạng chuỗi tự do không làm được:
1. **Lọc, tìm kiếm và truy vấn tự động theo trường cấu trúc trên Cloud:** Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch, Render Logs) có thể tự động parse các trường JSON để lọc chính xác: ví dụ `event == "ask_completed" AND cost_usd > 0.01` hoặc gom nhóm thống kê theo `user_id` mà không cần viết regex phức tạp và dễ gãy khi định dạng chuỗi thay đổi.
2. **Xây dựng Dashboard giám sát và kích hoạt Cảnh báo ngưỡng (Metrics & Alerting):** Có thể trích xuất trực tiếp số liệu định lượng (`tokens_in`, `tokens_out`, `cost_usd`) để vẽ biểu đồ chi phí LLM theo thời gian thực và thiết lập cảnh báo tự động (Alert) khi chi phí trong 5 phút vượt ngưỡng, điều mà câu print text đơn thuần không thể hỗ trợ tổng hợp.

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
| 1 stage (bản đầu) | 1024 MB |
| Multi-stage | 215 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~800 MB) gồm hai thành phần chính:
1. **Base image:** Bản 1-stage sử dụng `python:3.11` đầy đủ (full Debian), chứa toàn bộ các công cụ biên dịch C/C++ (`gcc`, `g++`, `make`), các thư viện header hệ thống (`linux-headers`, `libc-dev`) và các gói tiện ích không cần thiết cho runtime. Trong khi đó, bản multi-stage dùng `python:3.11-slim` chỉ giữ lại môi trường tối thiểu cần thiết để chạy Python.
2. **Build dependencies và pip cache:** Trong multi-stage build, trình biên dịch wheel, package cache (`~/.cache/pip`) và các tệp trung gian chỉ tồn tại ở stage `builder`. Stage `runtime` chỉ copy thư mục `/opt/venv` chứa mã nhị phân Python đã cài đặt hoàn chỉnh và mã nguồn ứng dụng, giúp loại bỏ hoàn toàn các file rác phát sinh trong quá trình cài đặt.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại:
  - **Các layer được dùng lại từ cache (CACHED):** Toàn bộ stage `builder` (bao gồm `COPY requirements.txt .` và `RUN pip install -r requirements.txt`) cùng các layer đầu của stage `runtime` (`FROM`, `WORKDIR`, `RUN useradd`, `COPY --from=builder /opt/venv`) đều được tái sử dụng từ cache vì `requirements.txt` không thay đổi.
  - **Các layer phải chạy lại:** Chỉ từ layer `COPY . .` trở đi ở stage runtime (vì mã nguồn ứng dụng có sự thay đổi checksum) và các lệnh tiếp theo. Thời gian build lại chỉ mất 1-2 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`:
  - Mỗi khi sửa bất kỳ ký tự nào trong mã nguồn `app/main.py`, layer `COPY . .` bị mất cache.
  - Khi một layer bị mất cache, mọi layer phía sau nó bắt buộc phải chạy lại từ đầu. Docker sẽ phải tải và cài đặt lại toàn bộ các thư viện trong `requirements.txt`, làm thời gian build kéo dài thêm vài phút mỗi lần commit code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện tấn công leo thang:
  1. Kẻ tấn công khai thác thành công một lỗ hổng RCE (Remote Code Execution, ví dụ qua deserialization, Command Injection hoặc lỗ hổng thư viện) trong ứng dụng Python để mở shell trong container.
  2. Vì container chạy mặc định không khai báo `USER`, tiến trình shell chiếm được có quyền `root` (UID 0). Do container dùng chung nhân Linux kernel với máy host, UID 0 bên trong container ánh xạ trực tiếp tới UID 0 của máy host (nếu không cấu hình user namespaces).
  3. Kẻ tấn công lợi dụng quyền root trong container để khai thác các cấu hình mount lỏng lẻo (như Docker socket `/var/run/docker.sock` hoặc volume hệ thống) hoặc khai thác lỗ hổng kernel escape để truy cập file hệ thống của host, cài đặt backdoor và kiểm soát toàn bộ server vật lý.
- Lệnh `USER appuser` cắt đứt chuỗi tấn công:
  - Lệnh này chuyển tiến trình chạy dưới một người dùng không có đặc quyền (UID 1000).
  - Khi kẻ tấn công khai thác RCE, shell chỉ có quyền của `appuser`: không thể chỉnh sửa file hệ thống container (`/etc`, `/bin`), không thể cài đặt công cụ leo thang bằng `apt`, và không có đặc quyền root để tương tác với Docker socket hay thực hiện container breakout.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Người dùng có thể gửi tối đa **20 requests** trong 2 giây liên tiếp.
Giải thích:
- Với cơ chế đếm theo phút đồng hồ (Fixed Window reset lúc giây 00):
  - Người dùng gửi 10 requests vào giây thứ **10:00:59** (cuối phút thứ nhất). Hệ thống ghi nhận 10/10 requests cho phút 10:00, vẫn hoàn toàn hợp lệ.
  - Ngay 1 giây sau, khi đồng hồ chuyển sang **10:01:00**, bộ đếm của fixed window được reset về 0 cho phút mới.
  - Người dùng lập tức gửi tiếp 10 requests nữa vào giây **10:01:00** (hoặc 10:01:01). Hệ thống ghi nhận 10/10 requests cho phút 10:01, vẫn được chấp thuận.
- Kết quả là trong khoảng thời gian chỉ 2 giây (10:00:59 - 10:01:01), hệ thống đã phải nhận tới 20 requests (gấp 2 lần hạn mức).
- Thuật toán Sliding Window (cửa sổ trượt 60 giây) giải quyết lỗi này bằng cách luôn tính tổng số request trong đúng 60 giây lùi lại từ thời điểm hiện tại (`now - 60s`), chặn đứng đợt burst request thứ 11 tại giây 10:01:00.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Điểm khác nhau cơ bản:**
  - **Rate Limit:** Giới hạn theo **tần suất và số lượng request** trong một khoảng thời gian ngắn (ví dụ: tối đa 10 requests/phút) để ngăn chặn tấn công từ chối dịch vụ (DoS), spam và bảo vệ năng lực xử lý tức thời của server. Trả về mã lỗi HTTP 429 (Too Many Requests).
  - **Cost Guard:** Giới hạn theo **tổng chi phí tiền tệ / lượng token tiêu thụ** tích lũy trong chu kỳ dài (ví dụ: tối đa $10.0/tháng). Mục đích là bảo vệ ngân sách tài chính vì trong các ứng dụng AI, mỗi request có độ dài prompt khác nhau sẽ tốn chi phí rất khác nhau. Trả về mã lỗi HTTP 402 (Payment Required).
- **Tình huống Rate limit cho qua nhưng Cost guard chặn:**
  - User cả ngày chỉ gửi đúng 1 request (tần suất cực thấp, Rate Limit hoàn toàn cho qua). Nhưng câu hỏi này đính kèm một tài liệu khổng lồ với số lượng token ước tính vượt quá ngân sách tháng còn lại của user (hoặc user đã chạm mốc tiêu $10.0 trước đó). Cost Guard sẽ chặn lại và trả về HTTP 402.
- **Tình huống Cost guard cho qua nhưng Rate limit chặn:**
  - User mới tạo tài khoản, ngân sách tháng còn nguyên $10.0 (Cost Guard cho phép). Tuy nhiên user dùng script gửi liên tục 15 requests trong vòng 2 giây (mỗi request chỉ hỏi một từ "Hi" tốn vài cent). Khi đến request thứ 11, Rate Limiter sẽ lập tức chặn lại và trả về HTTP 429 để bảo vệ server khỏi bị quá tải tức thời.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra dẫn đến sập toàn bộ hệ thống (Cascading failure):
1. **Redis gặp sự cố:** Kết nối tới Redis bị gián đoạn trong vòng 30 giây.
2. **Health check probe kiểm tra:** Orchestrator (Docker/K8s/Cloud) gửi request định kỳ tới endpoint gộp `/health` của cả 3 container agent.
3. **Endpoint báo lỗi:** Do Redis không phản hồi, endpoint kiểm tra Redis thất bại và trả về HTTP 503 (hoặc timeout) trên cả 3 container.
4. **Orchestrator restart container:** Vì `/health` đóng vai trò là liveness probe (chỉ thị container có bị treo cứng hay không), orchestrator thấy cả 3 container liên tục trả về 503 nên quyết định **kill và restart toàn bộ cả 3 container**.
5. **Rơi vào vòng lặp CrashLoopBackOff:** Khi các container mới vừa khởi động xong, chúng lại thực hiện kiểm tra Redis qua `/health`. Vì Redis vẫn đang trong khoảng 30s mất kết nối, các container mới khởi động lại tiếp tục bị coi là hỏng và bị restart lặp đi lặp lại.
6. **Hậu quả:** Toàn bộ service sập hoàn toàn (100% downtime). Nếu tách riêng `/health` (liveness) và `/ready` (readiness): container không bị restart vô ích, `/ready` chỉ tạm thời báo 503 để load balancer ngừng đẩy traffic, và khi Redis phục hồi sau 30s thì hệ thống tiếp tục hoạt động trơn tru ngay lập tức.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- Khi lưu trong Redis (Stateless):
  - Dù Load Balancer phân bổ request vào bất kỳ instance nào (Agent 1, Agent 2, hay Agent 3), tất cả các instance đều đọc và ghi chung vào Redis List `history:{user_id}`. Do đó `history_length` sẽ tăng tuần tự và nhất quán: 0 -> 2 -> 4 -> 6 -> 8...
- Nếu lưu trong dict Python nội bộ (Stateful):
  - Mỗi instance là một tiến trình riêng với bộ nhớ RAM tách biệt hoàn toàn.
  - Khi người dùng gửi liên tiếp các request và được Load Balancer điều phối round-robin:
    - Request 1 rơi vào Agent 1: `history_length = 0`, ghi vào dict của Agent 1 (lưu 2 message).
    - Request 2 rơi vào Agent 2: do dict của Agent 2 chưa có dữ liệu user này, `history_length = 0` (Agent 2 tưởng là cuộc trò chuyện mới).
    - Request 3 rơi vào Agent 3: `history_length = 0` (tiếp tục mất ngữ cảnh).
    - Request 4 quay lại Agent 1: `history_length = 2` (chỉ nhớ lại request 1, hoàn toàn bỏ sót request 2 và 3).
  - Kết quả là `history_length` sẽ nhảy thất thường (0, 0, 2, 0, 4...), khiến câu trả lời của AI bị mất ngữ cảnh hội thoại liên tục.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:**
  `Health check failed on port 8000. Service failed to respond within timeout. Container terminated with exit code 1.`
- **Cách tìm ra nguyên nhân:**
  Kiểm tra tab **Runtime Logs / Deploy Logs** trên Dashboard của Render/Railway. Quan sát thấy nền tảng tự động gán một cổng ngẫu nhiên thông qua biến môi trường `$PORT` (ví dụ `$PORT=10000` trên Render), nhưng trong câu lệnh CMD ban đầu của Dockerfile, Uvicorn lại bị gán cứng cổng `--port 8000`. Khi bộ định tuyến của Cloud gửi health probe đến cổng `$PORT` do nó cấp, server không lắng nghe trên cổng đó nên bị báo timeout và platform coi deployment thất bại.
- **Cách khắc phục:**
  1. Chỉnh sửa lệnh khởi chạy trong Dockerfile để đọc biến môi trường `$PORT` linh hoạt và bind vào `0.0.0.0`:
     `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  2. Đảm bảo endpoint `/health` trả về HTTP 200 nhanh chóng mà không chờ bất kỳ kết nối mạng ngoài nào. Sau khi cập nhật và deploy lại, platform kết nối thành công và service chuyển sang trạng thái Live.
