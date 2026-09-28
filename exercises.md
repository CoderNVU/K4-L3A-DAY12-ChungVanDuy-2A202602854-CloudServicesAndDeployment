# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
> Cách trả lời: thay thế bằng câu trả lời của bạn bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Chung Văn Duy  Mã học viên: 2A202602854

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: Khi deploy ứng dụng lên môi trường production hoặc cloud (như Railway/Kubernetes), người triển khai sơ suất quên cấu hình biến môi trường `AGENT_API_KEY` trong dashboard. 

Nếu hệ thống để giá trị mặc định là `"changeme"`, ứng dụng vẫn khởi động trơn tru và báo healthy, nhưng mang lại hai hiểm họa lớn:
1. **Lỗ hổng bảo mật nghiêm trọng:** Bất kỳ ai cũng có thể dùng key mặc định phổ biến `"changeme"` để gửi request trái phép, chiếm dụng tài nguyên hoặc làm cạn kiệt hạn mức chi phí.
2. **Lỗi ngầm runtime khó phát hiện:** Nếu key này được dùng để xác thực với các dịch vụ downstream hoặc external API, ứng dụng sẽ âm thầm nhận lỗi xác thực 401 hoặc sinh ra lỗi 500 giữa đêm khi người dùng thật gọi vào, rất tốn thời gian truy vết nguyên nhân.

Nhờ cơ chế fail-fast (không cho giá trị mặc định), ứng dụng lập tức crash ngay từ lúc startup khi thiếu key. Nền tảng deployment phát hiện container exit/fail lập tức cảnh báo đỏ và chặn rollout, giúp ngăn chặn triệt để việc đưa một phiên bản sai cấu hình lên phục vụ người dùng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
```json
{"timestamp": "2026-09-28T09:04:50.123456Z", "level": "INFO", "event": "request_completed", "method": "POST", "path": "/ask", "status": 200, "latency_ms": 42.15, "user_id": "test_user"}
```

Hai việc làm được với dòng log JSON này mà `print("đã trả lời xong")` không thể làm được:
1. **Truy vấn, lọc và gom nhóm có cấu trúc (Structured Querying):** Các công cụ quản lý log tập trung (như Datadog, ELK Stack, AWS CloudWatch, Grafana Loki) có thể tự động parse các trường JSON thành các indexable fields. Người vận hành có thể lọc tức thì các request lỗi (`status >= 400`), tìm kiếm theo endpoint cụ thể, hoặc gom nhóm toàn bộ hành vi request theo từng `user_id` để điều tra sự cố.
2. **Đo lường hiệu năng (APM) và tự động kích hoạt cảnh báo (Automated Alerting):** Nhờ có trường số học `latency_ms` và timestamp chuẩn ISO-8601 UTC, hệ thống giám sát có thể vẽ biểu đồ phân phối độ trễ (p50, p95, p99) theo thời gian thực và tự động kích hoạt cảnh báo (gửi webhook tới Slack, PagerDuty) khi tỷ lệ lỗi tăng cao hoặc độ trễ phản hồi vượt quá ngưỡng SLO cho phép.

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch (~750 MB) bao gồm:
1. **Các công cụ biên dịch và build tools hệ thống:** `build-essential`, trình biên dịch `gcc`, `g++`, tiện ích `make`, cùng các tệp header (`python3-dev`, `linux-headers`) chỉ cần thiết trong giai đoạn build để biên dịch các thư viện Python viết bằng C/C++ extension.
2. **Bộ nhớ đệm (cache) trong quá trình cài đặt:** Toàn bộ apt package cache (`/var/lib/apt/lists/*`) và pip wheel download cache phát sinh khi cài đặt dependencies.
3. **Các file và thư viện rác phát sinh khi build:** Trong mô hình multi-stage, stage `runtime` sử dụng base image `python:3.11-slim` siêu nhẹ, chỉ copy thư mục site-packages đã cài đặt hoàn chỉnh từ stage `builder` sang mà không hề sao chép bất kỳ công cụ build hay tệp rác nào, giúp image gọn nhẹ, bảo mật và pull/push nhanh hơn nhiều lần.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Khi sửa một ký tự trong `app/main.py` và build lại:**
  - *Layer được dùng lại từ cache (`CACHED`):* Toàn bộ các layer đứng trước lệnh copy mã nguồn: base image, tạo user/group `appuser`, thiết lập workdir `/app`, `COPY requirements.txt .`, và quan trọng nhất là bước `RUN pip install ...` (cài đặt dependencies).
  - *Layer phải chạy lại:* Bắt đầu từ layer `COPY app/ ./app/` (do nội dung thư mục `app` bị đổi checksum), các lệnh cấp quyền `chown`, thiết lập `USER`, `EXPOSE` và `CMD`. Toàn bộ quá trình build lại chỉ mất 1-2 giây vì không phải tải lại bất kỳ thư viện nào.
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Mỗi khi sửa bất kỳ file mã nguồn nào, checksum của context thay đổi làm layer `COPY . .` bị cache-bust. Docker buộc phải thực thi lại lệnh `RUN pip install` ngay phía sau từ đầu. Quá trình build sẽ phải tải lại và cài đặt lại toàn bộ dependencies qua mạng, kéo dài từ vài phút thay vì vài giây, gây lãng phí băng thông và tài nguyên CI/CD.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện dẫn tới chiếm quyền máy host:**
  1. *Khai thác lỗ hổng:* Mã nguồn Python hoặc một thư viện bên thứ ba tồn tại lỗ hổng bảo mật nghiêm trọng (ví dụ: Remote Code Execution qua insecure deserialization `pickle`, command injection qua `os.system` / `subprocess`, hoặc buffer overflow trong thư viện C-extension).
  2. *Chiếm quyền shell trong container:* Kẻ tấn công gửi payload khai thác và mở được một reverse shell thực thi lệnh bên trong container.
  3. *Sở hữu quyền root (UID 0):* Vì container mặc định chạy bằng root, tiến trình shell của kẻ tấn công mang UID 0. Theo cơ chế mặc định của Linux kernel khi không bật user namespace mapping, UID 0 bên trong container ánh xạ trực tiếp tới UID 0 (root) của máy host.
  4. *Container Escape:* Từ quyền root container, kẻ tấn công khai thác tiếp các cơ chế như mount Docker socket (`docker.sock`), các quyền kernel nguy hiểm, lỗ hổng kernel Linux (như Dirty COW, runc CVE) hoặc truy cập các tệp thiết bị `/dev` để thoát ra khỏi container và nắm toàn quyền điều khiển hệ điều hành máy host.
- **Lệnh `USER appuser` cắt đứt chuỗi ở đâu:**
  Lệnh `USER appuser` (UID 10001) cắt đứt chuỗi ngay tại Bước 3. Khi kẻ tấn công có được shell, chúng chỉ có quyền của một user không đặc quyền (`appuser`): không thể ghi đè các file hệ điều hành trong container (`/bin`, `/etc`), không thể cài thêm công cụ độc hại, không thể can thiệp vào các tiến trình khác và bị tước bỏ hầu hết các Linux capabilities, chặn đứng hoàn toàn khả năng thực hiện container escape ra máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- Người dùng có thể gửi tối đa: **20 requests** trong 2 giây liên tiếp.
- **Giải thích cách đạt được:**
  - Với cơ chế fixed window reset theo phút đồng hồ tại giây 00:
  - Ở phút thứ nhất, người dùng gửi dồn dập 10 request vào giây cuối cùng: từ `00:59.000` đến `00:59.999`. Hệ thống ghi nhận đủ 10 request cho phút đó.
  - Ngay tại giây tiếp theo `01:00.000`, đồng hồ bước sang phút mới và bộ đếm tự động reset về 0. Người dùng lập tức gửi tiếp 10 request nữa trong 1 giây đầu tiên: từ `01:00.000` đến `01:00.999`.
  - Kết quả: Trong khoảng thời gian chỉ 2 giây liên tiếp (từ `00:59` đến `01:01`), hệ thống đã phải gánh tới 20 request (gấp 2 lần hạn mức thiết kế), có thể gây spike quá tải dịch vụ.
  - Cơ chế sliding window (cửa sổ trượt 60 giây) giải quyết triệt để vấn đề này vì luôn tính tổng số request trong đúng 60 giây trôi qua tính tới thời điểm hiện tại.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Sự khác biệt cốt lõi:**
  - *Rate limit:* Kiểm soát **tốc độ / tần suất request tức thời** trong một khoảng thời gian ngắn (ví dụ: 10 request / 60 giây) nhằm bảo vệ hạ tầng máy chủ khỏi nghẽn mạng, sập tài nguyên và phòng chống tấn công DoS/Brute-force.
  - *Cost guard:* Kiểm soát **chi phí tài chính tích lũy** theo chu kỳ dài (ví dụ: ngân sách $10.00 / tháng tính theo token/lượt gọi LLM) nhằm bảo vệ ngân sách tài chính của hệ thống, không để tài khoản bị cạn tiền do lạm dụng gọi model AI đắt đỏ.
- **Tình huống Rate limit cho qua nhưng Cost guard chặn:**
  Người dùng gửi request đầu tiên trong ngày (tốc độ chỉ 1 req/phút $\rightarrow$ Rate limiter thoải mái cho qua), nhưng tổng chi phí sử dụng tích lũy của user đó trong tháng đã chạm ngưỡng $10.01 (vượt mức $10.00 cho phép) $\rightarrow$ Cost guard chặn ngay lập tức và trả về mã lỗi HTTP 402 Payment Required.
- **Tình huống Cost guard cho qua nhưng Rate limit chặn:**
  Một người dùng mới toanh chưa tiêu đồng nào (ngân sách tháng còn nguyên $10.00 $\rightarrow$ Cost guard hoàn toàn đồng ý), nhưng người này dùng tool/script bắn liên tục 15 request chỉ trong vòng 3 giây $\rightarrow$ Rate limiter phát hiện vượt ngưỡng 10 req/phút và chặn ngay từ request thứ 11, trả về mã lỗi HTTP 429 Too Many Requests.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra khi gộp 2 endpoint và Redis mất kết nối 30 giây:
1. **Giây 0–5:** Kết nối tới Redis bị gián đoạn. Endpoint gộp nhận thấy không ping được Redis nên trả về mã lỗi 503 Service Unavailable hoặc timeout.
2. **Giây 5–10:** Liveness probe của nền tảng quản trị container (Kubernetes / Railway / Docker daemon) định kỳ gọi vào endpoint kiểm tra. Thấy probe trả về lỗi liên tiếp, orchestrator lầm tưởng rằng process container của ứng dụng đã bị treo (hung/deadlock) nên gửi tín hiệu `SIGKILL` để tiêu diệt (kill) cả 3 container `agent`.
3. **Giây 10–30:** Cả 3 container được khởi động lại. Tuy nhiên, khi vừa khởi động xong, endpoint gộp lại tiếp tục fail vì Redis vẫn chưa khôi phục $\rightarrow$ Cả 3 container rơi vào trạng thái khởi động lại liên tục (CrashLoopBackOff), tiêu tốn tài nguyên CPU/RAM vô ích.
4. **Giây 30+:** Redis hồi phục hoàn toàn. Tuy nhiên, 3 container ứng dụng lúc này đang ở trạng thái khởi động lại dở dang hoặc bị orchestrator áp đặt thời gian trừng phạt restart backoff $\rightarrow$ Toàn bộ traffic của người dùng bị từ chối 100%, gây ra thảm họa sập dịch vụ dây chuyền (cascading failure).
*(Thiết kế chuẩn: `/health` độc lập chỉ kiểm tra app còn sống để không bao giờ bị restart oan; `/ready` kiểm tra phụ thuộc Redis để chỉ tạm thời ngắt định tuyến traffic tới container cho tới khi Redis phục hồi).*

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Nếu lưu trong dict Python (in-memory):**
  - Do mỗi container là một tiến trình độc lập với vùng nhớ riêng, khi scale 3 container đằng sau Load Balancer, các request cùng một `X-User-Id` sẽ được router phân phối ngẫu nhiên hoặc xoay vòng (round-robin) qua container A, B, và C.
  - Dict của container nào thì chỉ lưu các tin nhắn mà container đó từng xử lý. Kết quả là giá trị `history_length` sẽ thay đổi trồi sụt, nhảy lộn xộn không thể đoán trước: Request 1 vào A (`history_length = 1`), Request 2 bị điều hướng sang B (`history_length = 1`), Request 3 sang C (`history_length = 1`), Request 4 quay lại A (`history_length = 2`). Bot sẽ bị mất trí nhớ và trả lời sai ngữ cảnh hội thoại trước đó.
- **Khi lưu trong Redis tập trung:** Cả 3 container đều đọc và ghi chung vào danh sách `history:<user_id>` trên Redis. Dù request có được định tuyến tới container nào, `history_length` luôn tăng đều đặn chính xác: 1, 2, 3, 4, 5... đảm bảo tính phi trạng thái (stateless) hoàn hảo của các instance ứng dụng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Thông báo lỗi gặp phải:** Khi kiểm tra endpoint `/ready` trên domain thật của Railway (`https://k4-l3a-day12-chungvanduy-2a202602854-cloudservic-production.up.railway.app/ready`), nhận mã phản hồi:
  `HTTP/1.1 500 Internal Server Error` với body `Internal Server Error`.
- **Cách tìm ra nguyên nhân:**
  1. Mở Deployment Logs trên Dashboard của Railway và theo dõi traceback.
  2. Phát hiện lỗi xuất phát từ việc commit được Railway kéo về build là commit của CP3, trong đó endpoint `/ready` ở `app/main.py` vẫn giữ code mẫu ban đầu là `raise NotImplementedError("CP4: /ready probe")`.
  3. Đồng thời kiểm tra mục Variables của Web service trên Railway thì thấy biến môi trường `REDIS_URL` chưa được liên kết tham chiếu tới Redis Database service trong cùng project.
- **Cách khắc phục:**
  1. Hoàn thiện hàm xử lý `/ready` trong `app/main.py`: gọi `store.ping()` để kiểm tra kết nối Redis, trả về `{"status": "ready", "redis": True}` khi kết nối thành công, hoặc raise HTTPException 503 khi Redis lỗi.
  2. Cấu hình biến môi trường `REDIS_URL` trên Railway liên kết chính xác với biến kết nối của Redis service.
  3. Thực hiện commit và push mã nguồn hoàn thiện lên GitHub. Railway tự động trigger build và deploy lại phiên bản mới nhất. Kết quả kiểm tra lại endpoint `/ready` đã trả về thành công: `HTTP/1.1 200 OK {"status":"ready","redis":true}`.
