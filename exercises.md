# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Chí Công  Mã học viên: 2A202602634

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu dùng khóa mặc định `"changeme"` và quên cấu hình trên cloud, service vẫn
khởi động và người lạ có thể đoán khóa để gọi `/ask`, làm phát sinh chi phí.
Khi `agent_api_key` là trường bắt buộc, lần deploy đó thất bại ngay với lỗi
validation; tôi sửa cấu hình trước khi URL được mở cho người dùng. Như vậy lỗi
được phát hiện lúc tôi còn theo dõi quá trình deploy, thay vì qua hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log thực tế lấy từ `docker compose logs agent`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T07:53:31.862405+00:00", "user_id": "rate-check", "tokens_in": 321, "tokens_out": 45, "cost_usd": 7.515e-05}
```

Từ JSON này tôi có thể lọc và đếm số request theo `user_id`, đồng thời cộng
`cost_usd` hoặc `tokens_in` để tìm người dùng tiêu nhiều nhất. Với chuỗi
`print("đã trả lời xong")`, cả hai việc đều thiếu trường dữ liệu có cấu trúc và
phải đoán thông tin từ nội dung văn bản.

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
| 1 stage (bản đầu) | 1696 MB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Tôi đo bằng `docker image inspect`: bản một stage là 1.696.065.931 byte, bản
multi-stage là 271.185.178 byte. Phần chênh lệch chủ yếu đến từ base image
`python:3.11` đầy đủ và các thành phần phục vụ build/cài đặt. Bản multi-stage
dùng `python:3.11-slim`; stage runtime chỉ nhận các package đã cài từ builder,
không mang toàn bộ môi trường build sang image cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Sau khi thay đổi `app/main.py`, log build cho thấy `COPY requirements.txt`,
`RUN pip install --prefix=/install`, và `COPY --from=builder /install` đều
`CACHED`. Các bước `COPY app ./app`, `COPY utils ./utils` và bước sau đó tạo/
chown user runtime được chạy lại. Nếu `COPY . .` đứng trước `RUN pip install`,
bất kỳ thay đổi source nào cũng làm cache của lớp copy mất hiệu lực, kéo theo
việc cài lại toàn bộ dependency dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng thực thi lệnh trong ứng dụng có thể cho kẻ tấn công chạy lệnh với
quyền của process trong container. Nếu process là root và container còn được
cấp capability, mount socket Docker hoặc gặp lỗ hổng container runtime, kẻ đó
có thể sửa file hệ thống/container rồi tìm đường thoát ra host với quyền cao.
`USER appuser` cắt chuỗi ở bước đầu: mã bị chiếm quyền chỉ chạy với UID 10001,
không có quyền root để sửa phần lớn hệ thống hay sử dụng nhiều thao tác đặc
quyền. Nó giảm tác động, dù vẫn phải tránh mount/capability nguy hiểm.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa là 20 request trong khoảng hai giây: gửi 10 request ở 10:00:59, sau đó
gửi thêm 10 request ở 10:01:01. Bộ đếm theo phút coi đây là hai cửa sổ khác
nhau nên cả hai nhóm đều hợp lệ. Sliding window 60 giây vẫn nhìn thấy nhóm đầu
khi nhóm sau đến và sẽ chặn các request vượt quá tổng hạn mức 10.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ/số request trong 60 giây, còn cost guard giới hạn
tổng tiền của từng user trong cả tháng. Một user gửi 5 request rất dài, mỗi
request dùng hàng chục nghìn token, vẫn dưới 10 request/phút nhưng có thể vượt
ngân sách nên cost guard phải chặn. Ngược lại, user gửi 11 câu cực ngắn trong
vài giây khi mới tiêu vài xu: ngân sách còn nhiều nhưng rate limiter phải trả
429 để ngăn burst traffic.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu endpoint liveness cũng gọi Redis, khi Redis mất kết nối cả ba container
cùng trả 503. Orchestrator hiểu nhầm ba process đã chết nên restart cả ba gần
như đồng thời. Trong lúc restart không còn instance phục vụ; nếu Redis vẫn
chưa hồi phục, các container mới lại tiếp tục bị đánh dấu lỗi và restart.
Sự cố dependency 30 giây vì vậy biến thành downtime toàn service. Tách
`/health` và `/ready` giúp process vẫn sống, còn load balancer chỉ tạm rút các
instance chưa sẵn sàng khỏi traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Tôi chạy ba container Agent dùng chung Redis trên các cổng 8000, 8001 và 8002,
rồi gửi cùng `X-User-Id` lần lượt vào từng cổng. `history_length` quan sát được
là `0, 2, 4`, chứng minh container sau đọc được dữ liệu container trước đã ghi.
Nếu dùng dict Python, mỗi container có bộ nhớ riêng nên lần đầu rơi vào từng
container thường đều thấy `0`; sau đó con số tăng riêng theo container và có
thể nhảy `0, 2, 0, 4...` tùy request được phân phối vào đâu.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thực tế tôi gặp nằm ở lần chạy GitHub Actions đầu tiên: job `Deploy to Render`
dừng tại bước `Trigger Render deploy` với `curl: (3) URL rejected: No host part in
the URL`. Tôi mở log của job và thấy biến truyền cho `curl` rỗng, từ đó xác định
repository chưa có secret chứa Render Deploy Hook URL. Tôi sửa workflow để chỉ
chạy job deploy khi biến `DEPLOY_ENABLED=true`, sau đó thêm secret
`RENDER_DEPLOY_HOOK_URL` và biến `PUBLIC_URL`. Lần chạy tiếp theo đã hoàn thành cả
ba job test, build và deploy; bước smoke test gọi `/health` thành công trên URL
Render thật.
