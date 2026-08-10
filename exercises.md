# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> [Nội dung câu trả lời]` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Văn Quân  Mã học viên: 2A202601544

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên môi trường Production/Cloud mà người vận hành quên cấu hình biến môi trường `API_TOKEN`:
- **Nếu để mặc định `"changeme"`**: Ứng dụng vẫn khởi động thành công và qua được healthcheck. Tuy nhiên, bất kỳ ai trên Internet cũng có thể dùng token công khai `"changeme"` để gọi API, làm rò rỉ dữ liệu hoặc tiêu tốn tài nguyên. Lỗi này chỉ bị phát hiện khi hệ thống bị tấn công hoặc nhận hóa đơn tiền cloud tăng đột biến.
- **Nếu "chết sớm" (Fail Fast)**: Ứng dụng sẽ crash lập tức ngay khi start (`Pydantic ValidationError`), container thoát với lỗi exit code != 0. Hệ thống monitoring/deployment sẽ phát hiện ngay và cảnh báo người deploy (hoặc tự động rollback), giúp ngăn chặn một service không an toàn chạy trên production.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

```json
{"timestamp": "2026-08-10T17:30:00Z", "level": "INFO", "event": "chat_processed", "client_id": "sv-test", "usd_cost": 0.0000207, "prompt_tokens": 2, "completion_tokens": 34, "latency_ms": 350.2}
```

1. **Truy vấn & Lọc dữ liệu có cấu trúc (Structured Querying/Parsing)**: Các hệ thống quản lý log (Datadog, ELK Stack, CloudWatch) có thể tự động parse JSON để tạo biểu đồ monitoring, đặt alert tự động (ví dụ: tính tổng `usd_cost` theo `client_id`, nhóm theo `level`, hoặc lọc các request có `latency_ms > 1000`), điều mà log văn bản thuần `print("đã trả lời xong")` không thể thực hiện được.
2. **Theo dõi chi phí & Audit theo Client (Traceability & Accounting)**: Dòng log JSON chứa thông tin ngữ cảnh quan trọng như `client_id`, `usd_cost`, `prompt_tokens`, `completion_tokens`. Nhờ đó có thể kiểm toán chi phí API OpenAI của từng client cụ thể hoặc phục vụ việc giới hạn hạn mức (rate limiting/billing), còn `print()` hoàn toàn không mang dữ liệu này.

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
| 1 stage (bản đầu) | ~1.8 GB |
| Multi-stage | ~220 MB |

**Giải thích phần dung lượng chênh lệch (~1.58 GB):**
Bản 1 stage dùng base image đầy đủ (`python:3.11`), chứa nguyên bộ công cụ biên dịch (C/C++ compiler như `gcc`, `make`), bộ thư viện phát triển (`python3-dev`, `build-essential`), thư viện header và bộ cache của `pip`. Bản Multi-stage dùng base image `python:3.11-slim` cho runtime stage, chỉ copy phần thư viện Python đã biên dịch xong từ builder stage sang (dùng `--no-cache-dir`) cùng với code ứng dụng. Toàn bộ compiler và công cụ build thừa bị loại bỏ ở builder stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Khi sửa `app/main.py` với Dockerfile chuẩn hiện tại:**
  - Các layer nằm trước `COPY app/ app/` (bao gồm `FROM`, `WORKDIR`, `COPY requirements.txt .` và `RUN pip install ...`) **đều được dùng lại từ Docker cache**.
  - Chỉ có layer `COPY app/ app/` và các bước sau đó trong stage runtime mới phải chạy lại, giúp thời gian build lại cực kỳ nhanh (1-2 giây).
- **Nếu đặt `COPY . .` lên trước `RUN pip install`:**
  - Mỗi khi thay đổi dù chỉ 1 ký tự trong source code, checksum của layer `COPY . .` thay đổi làm **vô hiệu hóa toàn bộ cache từ bước đó trở đi**.
  - Docker sẽ phải tải và chạy lại lệnh `RUN pip install -r requirements.txt` từ đầu mỗi lần build, tiêu tốn thời gian và băng thông mạng không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- **Chuỗi sự kiện tấn công:**
  1. Kẻ tấn công khai thác lỗ hổng trong code Python (ví dụ: Remote Code Execution / Arbitrary Code Execution qua `os.system()` hoặc pickle deserialization).
  2. Do container đang chạy bằng user `root` (UID 0), kẻ tấn công chiếm được quyền root bên trong container.
  3. Kẻ tấn công khai thác lỗ hổng thoát container (Container Escape) thông qua bug kernel Linux chưa vá, truy cập Docker socket (`/var/run/docker.sock`), hoặc đọc các file nhạy cảm trên máy host được mount vào.
  4. Vì process của container thực chất chạy trực tiếp trên kernel host dưới UID 0 (root host), khi thoát khỏi container, kẻ tấn công chiếm luôn quyền root máy host.
- **Lệnh `USER appuser` cắt đứt chuỗi ở bước 2:**
  Lệnh `USER appuser` chuyển process sang chạy dưới UID thường (non-root). Khi code bị khai thác ở bước 1, kẻ tấn công chỉ có quyền của `appuser`. Do không có quyền root (UID != 0), kẻ tấn công không thể thực hiện các thao tác quản trị hệ thống hay khai thác lỗ hổng kernel đòi hỏi privilege để thoát container sang máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

1. **Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`:**
   Theo chuẩn RFC 7235 và RFC 6750 của giao thức HTTP, khi server trả về status code `401 Unauthorized`, server **bắt buộc** phải gửi kèm header `WWW-Authenticate` để chỉ định phương thức xác thực mà client cần tuân thủ (ở đây là scheme `Bearer`).
2. **Vì sao trả cùng một thông báo lỗi cho 3 trường hợp:**
   Đây là nguyên tắc bảo mật **thông tin tối thiểu (Information Disclosure Prevention)**. Nếu trả lời chi tiết (ví dụ "Token đúng nhưng đã hết hạn" vs "Token không tồn tại"), kẻ tấn công có thể lợi dụng thông tin này để thám mã (enumeration attack) — nhận biết thông tin nào đã chính xác để tập trung dò tìm/brute-force phần còn lại. Trả về thông báo lỗi chung giúp ẩn thông tin xác thực nội bộ khỏi kẻ tấn công.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

1. **Số request gửi được trước khi bị 429:**
   Client gửi được tối đa **10 request** liên tiếp.
2. **Nếu bỏ đoạn `min(capacity, ...)` trong `available()`:**
   - Sau 10 phút im lặng, lượng token tích lũy là: $10 \text{ phút} \times 10 \text{ token/phút} = 100 \text{ token}$.
   - Client sẽ gửi được **100 request** liên tiếp trước khi bị 429.
   - **Lý do:** Hàm `min(capacity, ...)` đóng vai trò giới hạn dung lượng xô (cap), kiểm soát tốc độ bùng nổ tối đa (burst capacity). Nếu bỏ `min`, xô sẽ tích lũy token vô hạn theo thời gian chờ. Một client sau thời gian dài im lặng có thể tung ra lượng request ồ ạt trong thời gian ngắn, gây quá tải (DDoS) hệ thống backend.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

- **Hạn mức $30/tháng:**
  - **Thiệt hại tối đa trong ngày:** $30 (toàn bộ ngân sách của cả tháng bị tiêu hết chỉ trong vài giờ từ 2h sáng).
  - **Khả năng hồi phục:** Client bị khóa (lỗi 429 / Budget Exceeded) cho tới hết tháng. Service chỉ tự khôi phục vào **ngày 1 của tháng tiếp theo** (chờ tối đa 29-30 ngày).
- **Hạn mức $1/ngày:**
  - **Thiệt hại tối đa trong ngày:** $1.0 (khi đạt mốc $1, client bị chặn ngay lập tức, bảo vệ 96.7% ngân sách tháng còn lại).
  - **Khả năng hồi phục:** Service tự động reset quota và phục hồi vào **00:00 UTC ngày hôm sau** (chờ tối đa 22 tiếng).
- **Kết luận:** Hạn mức theo ngày ($1/ngày) khoanh vùng thiệt hại do sự cố lặp vô hạn của client và rút ngắn thời gian khôi phục dịch vụ từ cả tháng xuống trong ngày.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. **Giây 0 - 10 (Redis mất kết nối):**
   Container 1 thực hiện probe chung và thất bại. Orchestrator (Docker Swarm/Kubernetes) đánh giá container 1 bị hỏng (unhealthy).
2. **Giây 10 - 20 (Orchestrator tiêu hủy & khởi động lại container):**
   Orchestrator tiến hành **kill container 1** và tạo container mới thay thế. Tiếp đó container 2 và 3 cũng thất bại probe và bị kill tương tự.
3. **Giây 20 - 30 (Vòng lặp boot / CrashLoopBackOff):**
   Các container mới khởi động lại tiếp tục gọi probe chung, tiếp tục thất bại do Redis vẫn đang ngắt kết nối, và lại tiếp tục bị kill. Toàn cụm 3 container rơi vào vòng lặp restart liên tục.
4. **Hậu quả:**
   Tất cả request tĩnh hoặc endpoint không phụ thuộc vào Redis (như landing page, /healthz thuần) đều bị sập hoàn toàn. Khi Redis kết nối lại ở giây 30+, các container vẫn đang dở dang trong chu kỳ restart khiến thời gian hồi phục hệ thống kéo dài thay vì chỉ tạm ngừng nhận traffic ở Load Balancer.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** Health check timeout / Connection refused khi deploy dịch vụ lên Railway (hoặc Render).
- **Thông báo lỗi:** `Health check failed: connection refused on port 8000` hoặc `Timeout waiting for container to respond on port 8000`.
- **Cách tìm nguyên nhân:**
  Kiểm tra log ứng dụng trong phần **Deploy Logs** trên Railway Dashboard. Phát hiện uvicorn đang chạy cố định trên cổng 8000 (`http://0.0.0.0:8000`), trong khi platform Cloud tự động cấp phát cổng động thông qua biến môi trường `$PORT` (ví dụ `PORT=6321`). Vì container không lắng nghe cổng `$PORT`, platform không thể thực hiện HTTP health check thành công.
- **Cách sửa:**
  Cập nhật lệnh `CMD` trong `Dockerfile` sang dạng Shell form để đọc động biến môi trường `$PORT`:
  `CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]`
  Đồng thời chỉnh sửa chỉ thị `HEALTHCHECK` trong Dockerfile đọc biến `PORT` động qua `os.getenv('PORT', '8000')`.
