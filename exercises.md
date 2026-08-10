# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay các dòng hướng dẫn bên dưới bằng câu trả lời của bạn.

> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Việt Thắng  Mã học viên: 2A202601321

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy ứng dụng lên Cloud mà quên cấu hình AGENT_API_KEY, việc để giá trị mặc định "changeme" sẽ khiến app vẫn khởi động bình thường, tạo ra lỗ hổng bảo mật nghiêm trọng khi kẻ xấu có thể đoán thử khóa này để gọi API và đốt sạch ngân sách LLM; trong khi việc "chết sớm"sẽ làm ứng dụng sập ngay lập tức khi khởi động, khiến Healthcheck thất bại và thông báo lỗi ngay cho Dev khắc phục cấu hình trước khi bị kẻ xấu lợi dụng.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log mẫu JSON:
`{"event":"ask_completed","level":"info","timestamp":"2026-08-10T05:09:52.123456+00:00","user_id":"sv-test","tokens_in":3,"tokens_out":35,"cost_usd":0.00002145}`

Hai việc làm được với log JSON:
1. Công cụ quản lý log (Datadog/ELK) có thể tự động bóc tách và truy vấn theo trường (ví dụ: tìm tất cả log của `user_id="sv-test"` hoặc lọc `cost_usd > 0.01`).
2. Dễ dàng vẽ biểu đồ giám sát và cài đặt cảnh báo tự động theo thời gian thực (ví dụ: thống kê tổng số token tiêu thụ theo giờ) nhờ cấu trúc JSON nhất quán.

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
| 1 stage (bản đầu) | ~1.01 GB |
| Multi-stage | ~178 MB |

Giải thích: phần dung lượng chênh lệch (~830 MB) là do bản 1-stage chứa toàn bộ trình biên dịch C/C++, SDK, pip build tools, các tập tin cache cài đặt tạm và thư viện hệ thống thừa. Multi-stage build đã lọc bỏ toàn bộ môi trường build, chỉ copy duy nhất các thư viện Python đã compiled sang runtime stage nhẹ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: Các layer `COPY requirements.txt` và `RUN pip install` được dùng lại từ cache (do file requirements không đổi). Chỉ layer `COPY app/ ./app/` và các bước sau mới phải chạy lại.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Mỗi thay đổi nhỏ trong code `app/main.py` sẽ làm hỏng cache của layer `COPY . .`, buộc Docker phải chạy lại toàn bộ lệnh `RUN pip install` từ đầu, làm tăng thời gian build từ vài giây lên nhiều phút.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Kẻ tấn công lợi dụng lỗ hổng RCE trong code Python để thực thi lệnh shell trong container ➔ Vì container chạy quyền `root`, kẻ tấn công lợi dụng lỗ hổng phân tách container (hoặc volume mount) để truy cập hệ tệp host ➔ Quyền root trong container giúp kẻ tấn công chiếm toàn bộ quyền root trên máy host. Lệnh `USER appuser` chuyển tiến trình sang tài khoản thường (UID 1000) ngay trước CMD, cắt đứt chuỗi tấn công vì kẻ thù không có quyền root trong container để thực hiện leo thang đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request** trong 2 giây liên tiếp. Người dùng gửi 10 request vào 1 giây cuối cùng của phút thứ nhất (giây 59) và gửi tiếp 10 request vào 1 giây đầu tiên của phút thứ hai (giây 00). Khi reset theo phút đồng hồ, cả 2 phút đều ghi nhận đúng 10 request/phút (hợp lệ), nhưng thực tế trong cửa sổ 2 giây có tới 20 request đi qua.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Khác biệt: Rate Limit kiểm soát tần suất/số lượng request trong cửa sổ thời gian ngắn (phút), còn Cost Guard kiểm soát tổng số tiền ($ USD) tiêu tốn trong thời hạn dài (tháng).
- Rate Limit cho qua nhưng Cost Guard chặn: User mới gửi 1 request trong phút (dưới hạn mức 10 req/min), nhưng request đó chứa prompt siêu dài làm tổng chi phí tháng vượt mốc $10.0 ➔ Cost Guard chặn (HTTP 402).
- Cost Guard cho qua nhưng Rate Limit chặn: User mới tiêu $0.1 trong tháng (còn nguyên ngân sách $10.0), nhưng gửi dồn dập 11 request trong 10 giây ➔ Rate Limit chặn ở request thứ 11 (HTTP 429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện:
1. Endpoint gộp trả 503 do không kết nối được Redis.
2. Orchestrator coi Liveness probe thất bại ➔ Cho rằng cả 3 container agent đều bị sập.
3. Orchestrator liên tục tiêu hủy và khởi động lại (restart) cả 3 container agent.
4. Cụm ứng dụng rơi vào vòng lặp CrashLoopBackOff, gây quá tải hệ thống và khiến app không thể phục hồi kể cả khi Redis đã hoạt động lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu trong dict Python RAM: Do Nginx phân phối các request xoay vòng qua 3 container (`agent-1`, `agent-2`, `agent-3`), mỗi container giữ 1 dict riêng. Con số `history_length` sẽ không tăng liên tục (0 ➔ 2 ➔ 4 ➔ 6 ➔ 8) mà nhảy ngẫu nhiên thất thường tùy thuộc request rơi vào container nào (ví dụ: `agent-1` trả 0, `agent-2` trả 0, `agent-3` trả 0, rồi `agent-1` trả 2...).

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Thông báo lỗi: `Error: Invalid value for '--port': '$PORT' is not a valid integer` khi chạy `railway up`. Nguyên nhân: Mở `railway logs` phát hiện `startCommand` trong `railway.toml` gọi trực tiếp `uvicorn --port $PORT` mà không bọc qua shell, khiến uvicorn nhận chuỗi thô `"$PORT"`. Cách sửa: Cập nhật `startCommand` trong `railway.toml` thành `sh -c 'uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}'` để shell tự động giải mã `$PORT` thành số nguyên trước khi truyền cho Uvicorn.
