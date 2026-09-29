# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vo Cong Danh  Mã học viên: 2A202602739

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu tôi quên đặt `AGENT_API_KEY` trên cloud, app sẽ dừng ngay lúc khởi động vì
secret là bắt buộc. Nhờ vậy deployment bị đánh dấu lỗi trước khi nhận request,
thay vì âm thầm chạy bằng khóa `changeme` mà người khác có thể đoán được để gọi
API và làm phát sinh chi phí.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log tôi quan sát được là:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:51:22.133322+00:00", "user_id": "sv-test", "cost_usd": 0.0001}
```

Vì đây là một JSON object trên một dòng, tôi có thể lọc riêng các request của
`user_id`, hoặc đếm/tổng hợp `cost_usd` theo thời gian bằng công cụ log. Một
chuỗi `print` thông thường không có các trường cố định để máy phân tích và
không mang theo thời điểm, người dùng hay chi phí một cách có cấu trúc.

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
| 1 stage (bản đầu) | ... MB |
| Multi-stage | ... MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Image multi-stage đã build được có dung lượng khoảng **271 MB**. Bản một stage
không được giữ lại trong phiên build này để đo đối chiếu. Mục tiêu của
multi-stage là không mang các file cài đặt trung gian, cache và công cụ build
từ stage builder sang runtime; runtime chỉ giữ Python, dependency đã cài và
source cần chạy. Vì vậy khi dependency có phần build nặng, bản multi-stage sẽ
nhỏ hơn và ít thành phần thừa hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi chỉ sửa một ký tự trong `app/main.py`, các layer cài dependency phía trước
(`COPY requirements.txt` và `pip install`) vẫn được lấy từ cache. Layer copy
source phía sau phải chạy lại. Nếu đặt `COPY . .` trước `pip install`, mọi thay
đổi source cũng làm layer đó đổi và khiến bước cài dependency chạy lại, làm
build chậm hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng trong app có thể cho phép kẻ tấn công chạy lệnh bên trong container.
Nếu process là root, lệnh đó có quyền cao nhất trong container và có thể khai
thác thêm quyền hoặc truy cập tài nguyên được gắn từ host. `USER appuser`
chuyển process sang user thường trước khi app chạy, nên kể cả khi app bị khai
thác thì quyền mặc định cũng bị giới hạn và không phải root trên container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với cách reset theo phút đồng hồ, có thể gửi tối đa **20 request trong khoảng
2 giây**: gửi 10 request ngay trước thời điểm giây `00`, rồi gửi tiếp 10
request ngay sau khi bộ đếm phút mới được reset. Sliding window tính đúng 60
giây gần nhất nên không có khoảng hở ở ranh giới phút như vậy.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn số lần gọi trong một cửa sổ thời gian, còn cost guard giới
 hạn tổng chi phí theo user trong tháng. Một user có thể gửi ít request nhưng
mỗi request có chi phí lớn, nên rate limit cho qua nhưng cost guard trả 402 khi
vượt ngân sách. Ngược lại, user có thể gửi quá nhiều request rẻ trong một phút
và bị rate limit trả 429 dù tổng chi phí tháng vẫn còn thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp liveness và readiness, khi Redis mất kết nối cả ba container sẽ trả lỗi
probe trong khoảng 30 giây. Load balancer có thể loại cả ba instance khỏi pool
hoặc hệ thống restart chúng, dù process và endpoint HTTP vẫn còn sống. Request
vì vậy bị gián đoạn đồng loạt. Với hai endpoint riêng, `/health` vẫn báo process
đang sống, còn `/ready` báo 503 để ngừng nhận traffic cho tới khi Redis kết nối
lại.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi lưu dict trong Python, mỗi container có một bản lịch sử riêng. Request đi
qua instance khác sẽ thấy `history_length` bị quay về thấp hơn hoặc về 0, nên
con số thay đổi không ổn định. Khi lưu trong Redis, cả ba instance dùng chung
store; gọi nhiều lần với cùng `X-User-Id` sẽ thấy lịch sử tăng nhất quán dù
request được phân phối sang container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi chạy `railway up`, tôi gặp lỗi `Access is denied. (os error 5)` sau bước
đóng gói source. Nguyên nhân là CLI phải đọc cả các thư mục local như `.venv`,
`.pytest_cache` và `.git`, trong đó có thư mục bị giới hạn quyền. Tôi loại các
thư mục này cùng `.env` bằng `.railwayignore` và deploy lại từ source sạch.
Sau khi service chạy, một lần gọi `/ask` cũng trả `invalid or missing API key`
vì tôi gửi nhầm header `X-Agent-Key`; code yêu cầu `X-API-Key`. Đổi đúng tên
header thì request được xác thực.
