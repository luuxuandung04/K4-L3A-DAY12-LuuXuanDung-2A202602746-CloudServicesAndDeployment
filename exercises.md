# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền câu trả lời chi tiết cho từng câu hỏi bên dưới.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lưu Xuân Dũng  Mã học viên: L3A202602746

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Trong thực tế, khi deploy ứng dụng lên cloud hoặc staging, lập trình viên có thể sơ suất quên thiết lập biến môi trường `AGENT_API_KEY` trong dashboard. Nếu để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động bình thường và mở public ra Internet. Các công cụ bot quét lỗ hổng sẽ dò ra endpoint `/ask` và sử dụng key mặc định phổ biến này để gọi API liên tục, tiêu tốn sạch ngân sách token LLM mà người quản trị không hề hay biết cho đến khi nhận hóa đơn. Ngược lại, việc không đặt giá trị mặc định (fail-fast) khiến pydantic ném `ValidationError` và crash ngay từ giây đầu tiên khởi động. Lỗi này làm tiến trình deploy thất bại ngay lập tức, buộc lập trình viên phải cấu hình secret hợp lệ trước khi ứng dụng có thể phục vụ traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log JSON thu được:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:32:05.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.00015}`
>
> Hai việc làm được với dòng log này mà `print()` thông thường không làm được:
> 1. **Lọc, tổng hợp và phân tích dữ liệu tự động (Query & Aggregation):** Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch) có thể parse các trường có cấu trúc để tính toán chỉ số theo thời gian thực, ví dụ truy vấn `SELECT SUM(cost_usd) WHERE user_id = 'sv-test'` để tính chi phí của từng người dùng, hoặc tìm user nào tiêu nhiều token nhất hôm nay.
> 2. **Cảnh báo ngưỡng thời gian thực (Automated Alerting):** Có thể thiết lập quy tắc giám sát tự động kích hoạt cảnh báo gửi tới Slack/PagerDuty khi phát hiện một request có chi phí vượt ngưỡng đột biến (ví dụ `cost_usd > 0.05`) hoặc tỷ lệ log `level: "error"` gia tăng bất thường trong 5 phút qua.

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
| Multi-stage | ~175 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (giảm hơn 800 MB) đến từ hai nguyên nhân chính:
> 1. **Base image:** Bản 1 stage dùng base image đầy đủ `python:3.11` dựa trên hệ điều hành Debian hoàn chỉnh chứa rất nhiều gói phần mềm hệ thống, công cụ quản trị, tài liệu hướng dẫn và bộ công cụ phát triển/biên dịch C/C++ (`gcc`, `g++`, `make`, `binutils`...). Trong khi đó, bản multi-stage dùng `python:3.11-slim` chỉ giữ lại các thành phần tối thiểu để chạy Python runtime.
> 2. **Tách biệt môi trường build và runtime:** Stage `builder` chịu trách nhiệm cài đặt và biên dịch thư viện, sau đó stage runtime chỉ copy các package kết quả từ `/install` sang `/usr/local`. Toàn bộ cache của pip (`--no-cache-dir`), compiler tạm và file header không hề bị copy sang image cuối, giúp image cực kỳ gọn nhẹ và an toàn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - Với Dockerfile tối ưu: Các layer từ đầu đến trước lệnh copy mã nguồn (bao gồm `FROM`, `WORKDIR`, `COPY requirements.txt .`, `RUN pip install ...`) hoàn toàn không thay đổi nên Docker tái sử dụng lại 100% từ cache (`CACHED`). Chỉ từ layer `COPY app ./app` trở đi mới bị mất cache và được thực thi lại. Do không phải cài lại thư viện, quá trình build lại chỉ mất khoảng 1-2 giây.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi lần sửa một ký tự trong bất kỳ file mã nguồn nào, layer `COPY . .` sẽ bị thay đổi và làm vô hiệu hóa (invalidate) cache của tất cả các layer đứng sau nó. Kết quả là lệnh `RUN pip install` sẽ bị ép chạy lại từ đầu, Docker phải tải và cài lại toàn bộ gói dependency mỗi lần chỉnh sửa code, làm chậm tiến độ làm việc nghiêm trọng (mất từ 2 đến 5 phút mỗi lần build).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> - Chuỗi sự kiện dẫn tới kiểm soát máy host:
>   1. Ứng dụng Python có một lỗ hổng bảo mật (ví dụ: Remote Code Execution qua Command Injection hoặc Deserialization không an toàn).
>   2. Kẻ tấn công gửi payload khai thác và chiếm được quyền điều khiển shell bên trong container.
>   3. Vì container mặc định chạy với user `root` (UID 0), tiến trình của kẻ tấn công sở hữu toàn quyền root trong namespace của container.
>   4. Khi máy host có một lỗ hổng hạt nhân (kernel vulnerability), misconfiguration quyền hạn (như gán capability `SYS_ADMIN`), hoặc mount nhầm Docker socket (`/var/run/docker.sock`), kẻ tấn công tận dụng quyền root này để thực hiện kỹ thuật container breakout (thoát khỏi container).
>   5. Sau khi breakout thành công, kẻ tấn công xuất hiện trên máy host với quyền `root` (UID 0) tối cao và kiểm soát hoàn toàn máy chủ.
> - Lệnh `USER` cắt đứt chuỗi ở bước 3: Bằng việc tạo và chuyển sang user không có đặc quyền `USER appuser` (UID 10001), khi kẻ tấn công khai thác được ứng dụng, chúng chỉ có quyền hạn của user thường. Chúng không thể can thiệp vào các file nhạy cảm của hệ thống, không có quyền admin bên trong container và bị chặn đứng khả năng thực thi các exploit leo thang đặc quyền để vượt rào ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> - Một người dùng có thể gửi tối đa **20 request** trong vòng 2 giây liên tiếp.
> - Cách đạt được con số đó:
>   Với cơ chế Fixed Window (đếm theo phút đồng hồ và reset lúc giây 00), giới hạn được chia thành các khối cố định độc lập (ví dụ `10:00:00` - `10:00:59` và `10:01:00` - `10:01:59`). Kẻ tấn công sẽ đợi đến cuối phút thứ nhất, gửi dồn dập 10 request tại thời điểm `10:00:59`. Ngay tại giây `10:01:00`, đồng hồ bước sang phút mới và bộ đếm tự động reset về 0. Ngay lập tức tại giây `10:01:00` hoặc `10:01:01`, kẻ tấn công gửi tiếp 10 request nữa. Cả hai đợt đều hợp lệ theo hạn mức 10/phút của từng khối, nhưng thực tế hệ thống phải hứng chịu 20 request chỉ trong khoảng thời gian vỏn vẹn 2 giây, tạo ra traffic spike làm tê liệt dịch vụ. Thuật toán Sliding Window (cửa sổ trượt) loại bỏ hoàn toàn lỗ hổng này vì luôn xét cửa sổ thời gian liên tục `[now - 60s, now]`.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> - **Điểm khác biệt:** Rate limit kiểm soát **tốc độ và số lượng request** trong một khoảng thời gian ngắn (ví dụ: 10 request/phút) để ngăn chặn tấn công từ chối dịch vụ (DoS) và giữ ổn định cho web server. Trong khi đó, Cost Guard kiểm soát **tổng số tiền chi phí phát sinh** (USD) tích lũy trong chu kỳ dài (tháng) dựa trên lượng token LLM tiêu thụ thực tế để bảo vệ ngân sách tài chính.
> - **Tình huống Rate limit cho qua nhưng Cost Guard chặn:** Người dùng chỉ gửi 1 request duy nhất trong cả tiếng đồng hồ (tốc độ rất thấp, Rate limit cho qua thoải mái). Tuy nhiên, request này đính kèm một tài liệu khổng lồ với prompt phức tạp tiêu tốn hơn 100.000 token, chi phí ước tính là $2.0, trong khi hạn mức ngân sách tháng của người dùng chỉ còn lại $0.5. Cost Guard sẽ phát hiện vượt ngân sách và trả về mã lỗi 402 Payment Required để chặn lại.
> - **Tình huống Cost Guard cho qua nhưng Rate limit chặn:** Đầu tháng mới, tài khoản người dùng vừa được cấp lại toàn bộ $10.0 ngân sách và chưa tiêu đồng nào. Người dùng chạy script gửi liên tục 20 câu hỏi ngắn chỉ trong 3 giây. Tổng chi phí của 20 câu này chưa tới $0.005 (Cost Guard thấy ngân sách dư dả), nhưng tốc độ gửi request vượt quá 10 req/phút nên Rate limit lập tức chặn từ request thứ 11 với mã lỗi 429 Too Many Requests để tránh nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự các sự kiện xảy ra khi Redis mất kết nối 30 giây:
> 1. Giây thứ 0: Kết nối mạng tới Redis gặp sự cố. Cả 3 container agent nhận thấy không kết nối được Redis. Do endpoint gộp kiểm tra Redis, liveness probe của cả 3 container đồng loạt báo lỗi (hoặc HTTP 503).
> 2. Giây thứ 5 - 10: Bộ điều phối hạ tầng (Orchestrator như Docker Swarm / Kubernetes / Cloud Agent) thấy liveness probe thất bại liên tiếp liền đưa ra kết luận sai lầm rằng bản thân process ứng dụng bên trong container đã bị chết/treo, và lập tức ra lệnh **kill và khởi động lại (restart) toàn bộ cả 3 container**.
> 3. Giây thứ 10 - 30: Cả 3 container khởi động lại nhưng Redis vẫn chưa phục hồi, dẫn tới endpoint liveness lại tiếp tục trả về lỗi. Orchestrator tiếp tục restart container nhiều lần, đưa toàn bộ cụm vào trạng thái CrashLoopBackOff / restart storm, làm cạn kiệt tài nguyên CPU của máy chủ.
> 4. Giây thứ 30: Redis hoạt động trở lại bình thường. Tuy nhiên, lúc này toàn bộ 3 container agent vẫn đang bị kẹt trong chu kỳ khởi động lại và back-off trễ của Orchestrator, khiến toàn bộ hệ thống sập hoàn toàn (downtime kéo dài) thay vì phục hồi tức thì.
> *(Nếu tách riêng: `/health` độc lập giúp container vẫn sống, `/ready` báo 503 để Load Balancer tạm thời không chuyển traffic, và khi Redis phục hồi thì hệ thống sẵn sàng phục vụ ngay lập tức mà không phải restart).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> - Khi lưu trong Redis (Stateless): Toàn bộ 3 container đều đọc/ghi lịch sử vào cùng một cơ sở dữ liệu Redis chung, do đó mỗi lần hỏi tiếp theo, `history_length` sẽ tăng đều đặn liên tục: 0 -> 2 -> 4 -> 6 -> 8... (mỗi lượt hỏi cộng thêm 1 tin nhắn user và 1 tin nhắn assistant), bất kể request đó được Load Balancer điều phối tới container nào.
> - Nếu lưu trong dict Python (Stateful trong RAM process): Mỗi container agent là một tiến trình độc lập sở hữu vùng nhớ RAM riêng biệt. Khi Load Balancer phân phối các request ngẫu nhiên (round-robin) qua 3 container:
>   - Request 1 vào Container A: `history_length` = 0 (Container A lưu 2 message vào RAM của mình).
>   - Request 2 vào Container B: Container B chưa từng gặp user này nên `history_length` lại là 0 thay vì 2!
>   - Request 3 vào Container C: `history_length` tiếp tục là 0!
>   - Request 4 lại rơi vào Container A: `history_length` bất ngờ nhảy lên 2.
>   Hậu quả là con số `history_length` thay đổi thất thường, nhảy loạn xạ và AI agent bị hiện tượng "mất trí nhớ ngẫu nhiên", không thể duy trì ngữ cảnh hội thoại mạch lạc.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> - **Thông báo lỗi gặp phải:** Khi deploy container lên nền tảng Cloud (Railway / Render), tiến trình deploy bị treo và báo thất bại sau 5 phút với log: `Health check failed: timeout connecting to service on port ...` hoặc `Application failed to respond on allocated port`.
> - **Cách tìm ra nguyên nhân:** Mở tab Deployment Runtime Logs trên dashboard của nền tảng cloud để kiểm tra luồng khởi động của service. Nhận thấy nền tảng tự động cấp phát một cổng ngẫu nhiên cho container qua biến môi trường `$PORT` (ví dụ `PORT=10000` hoặc một cổng nội bộ động), tuy nhiên lệnh khởi động ban đầu trong Dockerfile lại gán cứng `--port 8000`. Do đó, health check probe của platform gửi request tới cổng `$PORT` nhưng không nhận được phản hồi vì uvicorn đang lắng nghe ở cổng khác.
> - **Cách sửa:** Sửa lại lệnh `CMD` trong Dockerfile để uvicorn lắng nghe linh hoạt theo biến `$PORT` được platform truyền vào (sử dụng cú pháp shell fallback `${PORT:-8000}`) và lắng nghe trên toàn bộ network interface `0.0.0.0`:
>   `CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
>   Sau khi cấu hình, uvicorn nhận đúng cổng do cloud cấp, liveness probe phản hồi HTTP 200 và service chuyển sang trạng thái Live / Active thành công.
