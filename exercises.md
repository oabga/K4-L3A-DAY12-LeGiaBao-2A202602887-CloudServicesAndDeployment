# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: LeGiaBao  Mã học viên: 2A202602887

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu để giá trị mặc định "changeme", khi deploy lên production nhưng quên set `AGENT_API_KEY`, app sẽ chạy bình thường nhưng dùng key mặc định đó. Bất kỳ ai biết key này (vì nó public) có thể truy cập API miễn phí, tiêu tốn budget của bạn. Nhưng vì app chết ngay khi khởi động nếu thiếu API key, bạn sẽ ngay lập tức phát hiện lỗi cấu hình trong health check của orchestrator, từ đó sửa trước khi tệ hơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Ví dụ log JSON: `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T12:34:56.789123+00:00", "user_id": "sv01", "tokens_in": 50, "tokens_out": 100, "cost_usd": 0.0015}`

Hai việc có thể làm:
1. **Lọc theo trường cụ thể**: Datadog/CloudWatch có thể lọc tất cả request có `cost_usd > 1.0` hoặc `user_id = "sv01"` để phân tích chi phí hoặc hành vi user, mà không cần parse text.
2. **Thống kê tự động**: Có thể chạy aggregation để tính tổng `tokens_in`, `tokens_out`, trung bình `cost_usd` theo hour/day/user. `print()` đơn thuần không thể làm được điều này khi có hàng triệu dòng log.

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
| 1 stage (bản đầu) | ~800 MB |
| Multi-stage | ~250 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Khoảng ~550 MB khác nhau là các công cụ build mà chỉ cần trong quá trình cài đặt dependencies (build-essential, gcc, headers, source code). Với 1-stage Dockerfile, `python:3.11` base image bao gồm tất cả những thứ này, và chúng đều được copy vào final image. Multi-stage Dockerfile chỉ copy wheel files (compiled Python packages) từ builder stage sang runtime stage, loại bỏ compiler, source code, và build tools không cần thiết trong production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với thứ tự hiện tại (COPY requirements.txt trước, RUN pip install, sau đó COPY .):
- Cache layers: FROM, RUN apt-get, RUN useradd, COPY requirements.txt, RUN pip install — tất cả đều reuse vì requirements.txt không thay đổi
- Phải chạy lại: COPY . . (vì code thay đổi), và các layer sau đó

Nếu đặt `COPY . .` lên trước `RUN pip install`:
- Cache bị phá vỡ tại `COPY . .` (vì code thay đổi)
- Phải chạy lại `RUN pip install` toàn bộ, dẫn tới build chậm hơn rất nhiều
- Lợi ích của caching dependencies bị mất khi mỗi lần sửa code đều phải reinstall tất cả packages

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: 
1. Lỗ hổng RCE (Remote Code Execution) trong app Python → attacker chạy được shell command bên trong container
2. Container chạy bằng root → attacker có quyền `uid=0`
3. Docker daemon chạy trên host (thường cũng quyền cao hoặc root) → attacker có thể mount volumes của host, truy cập `/var/run/docker.sock`
4. Attacker kiểm soát Docker daemon → có thể chạy container mới với volume mount toàn bộ filesystem của host → toàn quyền quản trị host

Lệnh `USER appuser` cắt đứt ở bước 2: attacker sẽ chạy lệnh với `uid=1000` (appuser), không phải root. Kể cả lỗ hổng RCE tồn tại, thiệt hại được giới hạn chỉ những file/quyền mà user `appuser` có thể truy cập, thường là app directory.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request** trong 2 giây liên tiếp.

Giải thích: Giả sử hạn mức là 10 request/phút với fixed window (reset lúc giây 00 của mỗi phút):
- User gửi 10 request lúc 12:00:59 (trong phút 12:00-12:01) — được phép vì chưa vượt 10
- Lúc 12:01:01, phút đó reset, user lại có quota 10 request mới → gửi thêm 10 request được
- Tổng 20 request trong khoảng thời gian chỉ 2 giây (12:00:59 - 12:01:01)

Sliding window tránh được loophole này bằng cách chỉ xóa request cũ hơn 60 giây, nên sự kiện reset không bao giờ xảy ra đột ngột.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> **Khác nhau:**
- Rate limit: đếm **số lượng request** trong cửa sổ 60 giây
- Cost guard: đếm **số tiền chi phí** từ đầu tháng đến nay

**Tình huống 1 — Rate limit cho qua, cost guard chặn:**
User gửi 8 request/phút (dưới limit 10), nhưng mỗi request dùng 5000 token input + 10000 token output = $0.15/request. Sau 100 request, user chi $15/tháng vượt budget $10 → cost guard chặn request thứ 101, nhưng rate limit vẫn cho phép vì chưa đủ 10 request/phút.

**Tình huống 2 — Rate limit chặn, cost guard cho qua:**
User gửi 15 request trong 60 giây (vượt limit 10) → rate limit chặn request thứ 11 ngay. Mỗi request chỉ dùng 10 token input + 20 token output = $0.0001/request → chi phí chỉ $0.001/tháng dưới budget $10.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
1. Redis mất kết nối → ping() timeout
2. Gộp endpoint trả 503 (unhealthy)
3. Orchestrator (Kubernetes/Docker Swarm) thấy liveness probe fail → tự động RESTART tất cả 3 container
4. Lúc đó, tất cả container đang xử lý request sẽ bị SIGTERM → graceful shutdown
5. Các request đang chạy dở bị cutr giữa chừng → user nhận 502 Bad Gateway
6. Sau 30 giây Redis phục hồi, nhưng 3 container mới khởi động xong cũng lúc đó
7. Container khởi động lại chạy lifespan handler → gọi lifecycle.install(), log service_started
8. /health trả 200 lại, orchestrator bắt đầu đẩy traffic

**Tệ hại:** Total outage trong 30+ giây vì health check bị lẫn với readiness check. Tất cả user traffic bị interrupt.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis (hiện tại):
- Request 1 → Agent A: history = [] (trống), append user message → history_length = 1
- Request 2 → Agent B: history = [msg1] (từ Redis), append user message → history_length = 2  
- Request 3 → Agent C: history = [msg1, msg2], append user message → history_length = 3
- **history_length tăng dần như mong đợi**

Nếu lịch sử nằm trong dict Python của mỗi process:
- Request 1 → Agent A: history = {} (empty dict trong RAM của A), append → history_length = 1
- Request 2 → Agent B: history = {} (empty dict riêng của B, không biết gì về A), append → history_length = 1 (!!!!)
- Request 3 → Agent C: history = {} (empty dict riêng của C), append → history_length = 1 (!!!!)
- **history_length luôn là 1 vì mỗi container không biết khoảng ghi của container khác → agent "mất trí nhớ"**

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** `Error: Invalid value for '--port': '${PORT:-8000}' is not a valid integer.`

**Cách tìm ra nguyên nhân:** Chạy `docker logs <container-id>` để xem error message. Thấy `${PORT:-8000}` được truyền như một chuỗi literal (không được expand). Kiểm tra Dockerfile CMD — nguyên nhân là dùng JSON array form `["uvicorn", ..., "--port", "${PORT:-8000}"]` mà JSON form không trigger shell expansion.

**Sửa:** Đổi CMD thành shell form hoặc exec form với `/bin/sh -c`:
```dockerfile
CMD ["/bin/sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

Hoặc dùng Python để read env variable trực tiếp:
```dockerfile
CMD python -c "import os, uvicorn; uvicorn.run('app.main:app', host='0.0.0.0', port=int(os.environ.get('PORT', 8000)))"
```
