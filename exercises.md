# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Hồng Yến  Mã học viên: 2A202601065

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu đặt mặc định `agent_api_key = "changeme"`, khi deploy lên Railway mà quên set biến `AGENT_API_KEY`, app vẫn khởi động bình thường. Lúc này bất kỳ ai biết key mặc định đó đều gọi được `/ask`, tốn ngân sách LLM mà mình không hay. Với `agent_api_key: str` không có default, app crash ngay lúc `get_settings()` được gọi lần đầu — Railway báo container unhealthy, mình nhìn log thấy `ValidationError: field required`, biết ngay là chưa set secret và sửa trước khi có ai khai thác được.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log JSON thu được khi gọi `/ask`:
> `{"timestamp": "2026-08-10T03:30:00Z", "level": "INFO", "user_id": "sv-001", "tokens_in": 45, "tokens_out": 120, "cost_usd": 0.00021, "latency_ms": 380}`
>
> Hai việc làm được mà `print()` không làm được:
> 1. **Lọc và tổng hợp tự động**: Dùng `jq '.cost_usd' log.json | awk '{sum+=$1} END{print sum}'` để tính tổng chi phí theo ngày — `print` chỉ là chuỗi thô, không parse được.
> 2. **Alert theo ngưỡng**: Hệ thống như Datadog hay Grafana Loki đọc JSON, tự tạo alert khi `latency_ms > 1000` — với `print` không có trường nào để so sánh.

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
| 1 stage (bản đầu) | ~210 MB |
| Multi-stage | ~165 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch (~45 MB) là các công cụ build-time mà giai đoạn `builder` dùng: `pip`, `setuptools`, các file `.h` header của C extension, cache của wheel, và toàn bộ metadata của pip. Ở multi-stage build, stage `builder` cài xong thì chỉ copy thư mục `/install` (chứa code đã compile) sang stage runtime — compiler và metadata bị bỏ lại, không theo vào image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (COPY requirements.txt → pip install → COPY app/):
> - Các layer `FROM`, `WORKDIR`, `COPY requirements.txt`, `RUN pip install` đều được **dùng lại từ cache** vì requirements.txt không thay đổi.
> - Chỉ layer `COPY app ./app` và các layer sau mới chạy lại.
>
> Nếu đặt `COPY . .` trước `pip install`: mỗi lần sửa bất kỳ file Python nào, Docker thấy context thay đổi → invalidate cache từ bước COPY trở đi → **pip install chạy lại toàn bộ** dù requirements.txt không đổi → build chậm hơn hàng phút mỗi lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện khi chạy root:
> 1. Kẻ tấn công gửi payload độc hại vào `/ask`, khai thác lỗ hổng trong code xử lý input.
> 2. Thực thi được lệnh shell bên trong container với quyền **root**.
> 3. Container root = UID 0 — nếu Docker daemon có cấu hình sai hoặc dùng volume mount, kẻ tấn công có thể ghi vào `/etc/passwd` của host, hoặc escape container qua `docker.sock`.
> 4. Kết quả: quyền root trên máy host.
>
> Lệnh `USER appuser` (UID 10001) cắt đứt ở bước 2: dù exploit thành công, process chỉ có quyền của `appuser` — không ghi được vào thư mục hệ thống, không tương tác được với Docker daemon.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với fixed window (reset lúc giây :00), có thể gửi tối đa **20 request trong 2 giây**:
> - Gửi 10 request vào giây :59 của phút thứ N (vẫn trong cửa sổ phút N → hợp lệ).
> - Đến giây :00 phút N+1, counter reset về 0.
> - Gửi thêm 10 request vào giây :00 (cửa sổ phút N+1, vẫn hợp lệ).
> - Tổng: 20 request trong khoảng 1-2 giây.
>
> Sliding window tránh được điều này vì nó nhìn vào đúng 60 giây trước thời điểm hiện tại, không reset theo đồng hồ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau**: Rate limit kiểm soát **tần suất** (số request/phút), cost guard kiểm soát **chi phí tích lũy** (USD/tháng).
>
> - **Rate limit cho qua, cost guard chặn**: User gửi 5 request/phút (dưới ngưỡng 10), nhưng mỗi request hỏi câu cực dài → mỗi lần tốn 0.5 USD → sau 21 request trong nhiều ngày, vượt budget 10 USD/tháng → cost guard chặn dù tần suất thấp.
>
> - **Cost guard cho qua, rate limit chặn**: Đầu tháng budget còn đầy (9.99 USD), user script tự động gửi 50 request trong 1 phút → rate limit 10/phút chặn từ request thứ 11, dù chi phí mỗi request rất nhỏ và tổng chưa vượt budget.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng kiểm tra Redis:
> 1. Redis mất kết nối (30 giây).
> 2. Load balancer gọi `/health` → app thử kết nối Redis → timeout → trả `503`.
> 3. Load balancer đánh dấu container **unhealthy**.
> 4. Orchestrator (Kubernetes/Railway) thấy liveness probe fail → **restart toàn bộ 3 container**.
> 5. Trong lúc restart, không container nào phục vụ được → **downtime hoàn toàn**.
> 6. Redis phục hồi sau 30 giây, nhưng 3 container vẫn đang trong quá trình khởi động lại.
>
> Với hai endpoint riêng: `/health` không kiểm tra Redis nên container không bị restart. `/ready` trả 503 → load balancer ngừng gửi traffic mới vào → không có request nào thất bại, chỉ queue lại cho đến khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (stateless): mọi container đọc/ghi cùng một key Redis → `history_length` tăng đều mỗi lần gọi: 1, 2, 3, 4... bất kể request vào container nào.
>
> Nếu dùng dict Python trong bộ nhớ: mỗi container có dict riêng. Load balancer round-robin phân phối request lần lượt sang 3 container → container 1 thấy history_length=1, container 2 thấy =1, container 3 thấy =1, rồi container 1 lại thấy =2... Con số nhảy loạn không theo thứ tự, lịch sử hội thoại bị mất liên tục vì mỗi container chỉ nhớ các request của chính nó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi gặp phải**: `Error: Invalid value for '--port': '$PORT' is not a valid integer.`
>
> **Nguyên nhân**: Railway ban đầu dùng builder **Railpack** (không phải Dockerfile). Railpack tự phát hiện đây là Python app có uvicorn, rồi tự sinh lệnh khởi động kèm `--port $PORT`. Khi lệnh đó chạy, biến `$PORT` không được shell expand vì Railpack truyền argument trực tiếp mà không qua `sh -c` → uvicorn nhận đúng chuỗi `"$PORT"` thay vì số nguyên.
>
> **Cách tìm ra**: Vào tab Deployments → View logs → thấy dòng lỗi uvicorn xuất hiện lặp đi lặp lại. Vào Settings → Build → thấy builder đang là "Railpack (Default)", không phải Dockerfile.
>
> **Cách sửa**: Đổi builder sang "Dockerfile" trong Railway Settings, thêm `railway.toml` với `startCommand = "sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'"` để shell tự expand `$PORT`. Đồng thời cập nhật Target port trong Networking khớp với cổng Railway gán thực tế.
