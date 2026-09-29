# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết câu trả lời dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đinh Bảo Hưng  Mã học viên: 2A202602524

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu lúc tạo service Railway tôi quên đặt `AGENT_API_KEY`, app dừng ngay khi khởi động và deployment báo lỗi để tôi sửa trước khi nhận traffic. Nếu dùng mặc định `changeme`, service vẫn lên mạng; người biết key mẫu có thể gọi `/ask`, tạo request và phát sinh chi phí dưới tài khoản của tôi.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi lấy từ `docker compose logs agent` sau khi gọi `/ask`:
>
> ```json
> {"user_id": "cp4-scale-smoke", "tokens_in": 49, "tokens_out": 52, "cost_usd": 3.855e-05, "event": "ask_completed", "level": "info", "timestamp": "2026-09-29T03:32:07.120978+00:00"}
> ```
>
> Tôi có thể lọc các bản ghi có `event=ask_completed` theo khoảng thời gian để đếm số request, và cộng `cost_usd` theo `user_id` để tìm người dùng tiêu nhiều nhất. Dòng `print("đã trả lời xong")` không chứa các trường cần cho hai phép tính đó.

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
| 1 stage (bản đầu) | 446,5 MB |
| Multi-stage | 71,9 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build Dockerfile một stage từ commit gốc `1bf8ea5` thành `agent:single-exercise`, rồi build Dockerfile hiện tại thành `agent:multi-exercise`. `docker image inspect` trả lần lượt 446.510.969 và 71.898.361 byte, chênh khoảng 374,6 MB. Bản đầu dùng base `python:3.11` đầy đủ; bản mới dùng `python:3.11-slim`, đây là phần lớn chênh lệch. Multi-stage còn chỉ chép virtualenv từ builder sang runtime, nên các file chỉ phục vụ build không đi theo. Thư viện đã cài trong virtualenv vẫn nằm ở runtime; không thể quy toàn bộ chênh lệch cho multi-stage.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Tôi nối đúng một ký tự `#` vào cuối `app/main.py`, build lại rồi khôi phục file gốc. Log build ghi layer `RUN python -m venv ... pip install ...` là `CACHED`; layer `COPY ... app ./app` và layer `COPY ... utils ./utils` chạy lại vì source đã đổi và layer sau phụ thuộc layer trước. Base image, `COPY requirements.txt` và các bước tạo môi trường trước source vẫn lấy từ cache. Nếu `COPY . .` đứng trước `RUN pip install`, một ký tự trong source sẽ làm mất cache của lệnh cài thư viện, nên build chậm hơn và phải cài lại dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗi cho phép thực thi lệnh, kẻ tấn công trước tiên nhận quyền của tiến trình trong container. Khi tiến trình chạy root, họ có quyền cao trong container; nếu container còn được cấp quyền quá rộng, gắn `docker.sock`, mount thư mục host nhạy cảm, hoặc có lỗ hổng thoát container, họ có thể tác động đến host. `USER appuser` cắt bớt quyền ngay ở bước thực thi lệnh trong container: tiến trình không còn là root và không tự ý sửa các file chỉ root được phép sửa. Nó không thay thế việc giới hạn mount/quyền container hay vá lỗ hổng thoát container.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request: gửi 10 request ở giây 59 của phút trước và 10 request ở giây 00 của phút sau. Bộ đếm theo phút reset đúng ranh giới nên chấp nhận cả hai nhóm trong khoảng hai giây. Sliding window 60 giây vẫn nhìn thấy nhóm đầu khi nhóm sau đến và chặn nhóm sau.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit chặn theo số request trong 60 giây; cost guard chặn theo tổng tiền của từng user trong tháng. Ví dụ user chỉ gửi một request lúc này nhưng đã tiêu hết ngân sách tháng: rate limit cho qua, cost guard trả 402. Ngược lại, user còn nhiều ngân sách nhưng gửi request thứ 11 trong cùng cửa sổ 60 giây: cost guard còn cho phép, rate limit trả 429. Trong code, rate limit chạy trước nên tình huống thứ hai dừng ở đó.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối → endpoint gộp bắt đầu trả 503 trên cả ba agent → sau các lần kiểm tra lỗi liên tiếp, orchestrator coi cả ba container không sống và khởi động lại chúng. Redis vẫn chưa trở lại trong 30 giây thì các container mới tiếp tục báo lỗi; kết nối của request đang xử lý có thể bị ngắt và cụm dao động vì restart. Với hai endpoint hiện tại, `/ready` báo 503 để ngừng đưa traffic vào agent phụ thuộc Redis, còn `/health` vẫn 200 khi process còn sống, nên không khởi động lại cả cụm chỉ vì Redis bị lỗi tạm thời.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi gửi ba request cùng `X-User-Id` tới URL cloud và nhận `history_length` lần lượt là 0, 2, 4. Mỗi lượt hoàn thành ghi hai message (user và assistant) vào Redis, nên lượt sau thấy lịch sử trước đó dù request đến instance khác. Nếu mỗi agent giữ một dict Python riêng, request được chia sang ba instance có thể cho 0, 0, 0; sau đó số đếm tăng riêng trên từng instance, ví dụ 2 khi quay lại instance đã nhận lượt đầu. Restart instance đó còn làm mất phần lịch sử trong dict của nó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi kiểm tra `/ask` trên bản Railway, test trả `401 {"detail":"invalid or missing API key"}` dù tôi đã đặt `DEPLOY_API_KEY` cục bộ. Tôi đối chiếu mà không in secret và thấy `AGENT_API_KEY` trong `.env` hiện khác `AGENT_API_KEY` đang lưu trên Railway. Tôi đặt `DEPLOY_API_KEY` cục bộ bằng key thật của service Railway rồi chạy lại `tests/test_cp5.py::TestPublicDeployment::test_ask_hoat_dong_voi_key_that`; test đã pass. Giá trị key chỉ nằm trong `.env` bị Git bỏ qua, không đưa vào tài liệu hoặc ảnh.
