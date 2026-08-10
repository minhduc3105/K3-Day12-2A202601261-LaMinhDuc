# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trả lời mẫu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lã Minh Đức  Mã học viên: 2A202601261

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu quên khai báo `AGENT_API_KEY` khi deploy, việc fail fast làm tiến trình dừng ngay ở lúc khởi động và log báo thiếu biến cấu hình. Tôi có thể sửa biến secret trước khi service nhận traffic. Nếu để mặc định `"changeme"`, service vẫn lên bình thường nhưng bot hoặc người biết khóa mặc định có thể gọi `/ask`, tiêu quota/chi phí LLM; lỗi chỉ lộ ra muộn khi xem log hay hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Ví dụ một log JSON của một lượt hỏi có dạng: `{"event":"ask_completed","level":"info","timestamp":"2026-08-10T02:30:00+00:00","user_id":"2A202601261","tokens_in":120,"tokens_out":80,"cost_usd":0.0001}`. Với cấu trúc này, tôi có thể (1) lọc và cộng `cost_usd` theo `user_id` để biết ai đang tiêu nhiều nhất, và (2) đếm/lọc các event theo thời gian, mức `level` để theo dõi tỉ lệ lỗi và đặt cảnh báo. `print("đã trả lời xong")` chỉ là chuỗi tự do, không có trường dữ liệu ổn định để máy truy vấn hay tổng hợp.

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
| 1 stage (bản đầu) | Chưa đo — cần build image 1 stage trên máy nộp bài |
| Multi-stage | Chưa đo — cần build image multi-stage trên cùng máy |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Số đo phải được lấy bằng `docker images` sau khi hoàn thiện Dockerfile, vì còn phụ thuộc cache và phiên bản image base trên máy build. Multi-stage nhỏ hơn do image cuối chỉ chứa Python runtime, các package đã cài và source cần chạy; compiler, header, pip cache và các công cụ dùng để build dependency ở stage `builder` không được copy sang stage cuối.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Sau khi chỉ sửa `app/main.py`, các layer base image, tạo thư mục làm việc, `COPY requirements.txt` và `RUN pip install` vẫn được lấy từ cache. Layer `COPY app` (hoặc `COPY . .`) thay đổi và các layer nằm sau nó phải chạy lại; lệnh khởi động chỉ là metadata của image. Nếu đặt `COPY . .` trước `RUN pip install`, một thay đổi nhỏ trong code cũng làm layer `COPY . .` đổi, kéo theo cache của `pip install` bị mất và phải cài lại toàn bộ dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng, ví dụ endpoint cho phép thực thi lệnh hệ điều hành từ dữ liệu người dùng, có thể cho kẻ tấn công chạy shell trong container. Nếu process chạy root, shell đó có quyền root trong container; khi kết hợp một lỗi cấu hình (socket Docker được mount, volume host ghi được) hoặc lỗ hổng container escape, họ có thể sửa file hay chiếm quyền cao trên host. `USER appuser` chuyển process ứng dụng sang tài khoản không đặc quyền trước khi chạy app, nên shell có được từ lỗi Python cũng chỉ có quyền của `appuser`; chuỗi leo thang bị chặn/giảm tác hại ngay tại bước chiếm quyền trong container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa là 20 request trong 2 giây. Người dùng gửi 10 request ngay trước lúc phút cũ kết thúc, chẳng hạn 10:00:59, rồi gửi tiếp 10 request ngay sau khi phút mới bắt đầu, 10:01:01. Bộ đếm theo phút đồng hồ reset ở giây 00 nên cả hai nhóm đều không vượt 10/phút, dù xét một khoảng 2 giây thì đã có 20 request. Sliding window luôn nhìn 60 giây gần nhất nên chặn nhóm thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tốc độ/số lượt gọi trong một cửa sổ ngắn; cost guard giới hạn tổng tiền đã dùng theo tháng. Ví dụ user gửi thưa, chỉ 1 request/phút, nhưng các request có prompt rất dài khiến tổng chi phí đã chạm ngân sách tháng: rate limit cho qua nhưng cost guard phải trả 402. Ngược lại, user mới trong tháng nên chưa tiêu hết ngân sách, nhưng bấm gửi 11 lần trong một phút khi giới hạn là 10: cost guard cho qua còn rate limit trả 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp endpoint và endpoint đó kiểm tra Redis, Redis mất kết nối khiến cả 3 container trả probe lỗi/`503`. Orchestrator coi chúng unhealthy rồi restart từng container hoặc đồng thời; trong lúc khởi động lại không còn instance sẵn sàng nhận traffic. Khi Redis hồi sau 30 giây, các container vẫn cần thời gian boot và health check mới được đưa lại vào load balancer, nên sự cố Redis ngắn trở thành gián đoạn toàn dịch vụ. Tách riêng thì `/health` vẫn 200 vì process còn sống, còn `/ready` trả 503 để load balancer tạm ngừng gửi traffic mà không restart container.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis dùng chung, `history_length` tăng dần theo mỗi lượt hỏi (thường tăng thêm 2 message: câu hỏi và câu trả lời), dù load balancer chuyển request sang instance khác. Nếu lưu bằng dict Python, mỗi instance có một lịch sử riêng: khi request tới cùng instance thì số tăng, nhưng khi sang instance khác số có thể quay về 0 hoặc thấp hơn. Vì vậy người dùng sẽ thấy agent nhớ/mất nhớ không ổn định.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Chưa có bản deploy cloud trong repo hiện tại để ghi nhận một lỗi thật. Khi deploy, lỗi cần kiểm tra đầu tiên là health check timeout: dấu hiệu thường là platform không gọi được `/health`. Tôi sẽ xem build/runtime log để xác định uvicorn đang bind `127.0.0.1` hoặc cố định cổng `8000` trong khi platform cấp `$PORT`; cách sửa là bind `0.0.0.0` và chạy `--port ${PORT:-8000}`, sau đó deploy lại và kiểm tra `/health`, `/ready`.
