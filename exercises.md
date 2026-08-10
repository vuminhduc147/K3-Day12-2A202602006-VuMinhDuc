# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bên dưới bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vũ Minh Đức  Mã học viên: 2A202602006

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy, nếu tôi quên đặt `AGENT_API_KEY`, ứng dụng dừng ngay và log cấu hình báo thiếu biến nên tôi phát hiện trước khi service nhận traffic. Nếu dùng mặc định `"changeme"`, service vẫn báo healthy nhưng mọi người biết giá trị mặc định đều có thể gọi `/ask`, làm phát sinh chi phí và lộ dữ liệu hội thoại.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi quan sát được có dạng `{"event":"request_completed","method":"POST","path":"/ask","status_code":401,"duration_ms":1}`. Với log JSON, tôi có thể lọc/đếm theo `status_code` để phát hiện nhiều request 401 và tính percentile của `duration_ms` để theo dõi độ trễ. Một câu `print` tự do không cung cấp các trường ổn định để máy truy vấn và tổng hợp như vậy.

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
| 1 stage (bản đầu) | 282 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Image multi-stage tôi đo được là 270 MB, nhỏ hơn bản một stage vì runtime không mang theo các tệp build trung gian, cache cài đặt và công cụ chỉ cần trong lúc cài dependency. Phần thư viện Python cần chạy ứng dụng vẫn được copy sang runtime nên không thể loại bỏ.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `WORKDIR`, `COPY requirements.txt` và `RUN pip install` được dùng lại từ cache; chỉ layer `COPY app`, các layer sau nó và bước export image phải làm lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi mã nguồn đều làm mất cache của layer cài dependency, khiến build lại chậm và phải tải/cài lại package dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng thực thi mã từ dữ liệu đầu vào có thể cho kẻ tấn công chạy lệnh trong container. Nếu tiến trình là root, họ có thể sửa file hệ thống trong container; kết hợp với volume/socket được mount sai hoặc lỗ hổng runtime, họ có thể tác động tới host với quyền cao. `USER appuser` cắt chuỗi ở bước sau khi chiếm tiến trình: mã độc chỉ có quyền của user thường, không được tùy ý ghi file hệ thống hay dùng tài nguyên đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong hai giây: gửi 10 request ở giây 59 của phút trước và 10 request ở giây 00 của phút sau. Bộ đếm theo phút đồng hồ reset đúng ranh giới nên mỗi nhóm vẫn hợp lệ, dù thực tế 20 request nằm trong một cửa sổ hai giây. Sliding window 60 giây ngăn được kiểu burst qua ranh giới này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất trong cửa sổ ngắn theo user, còn cost guard giới hạn tổng ngân sách sử dụng trong tháng. Một user gửi ít request nhưng mỗi request dùng nhiều token có thể chưa chạm 10 request/phút nhưng đã hết ngân sách nên cost guard chặn. Ngược lại, user gửi 11 request rất rẻ trong một phút có thể bị rate limit chặn dù ngân sách tháng vẫn còn nhiều.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint hợp nhất trả lỗi; liveness probe coi cả ba container là chết và lần lượt restart chúng. Container khởi động lại nhưng Redis vẫn mất nên probe tiếp tục thất bại, dẫn tới vòng lặp restart và toàn bộ cụm không phục vụ được kể cả các chức năng không phụ thuộc Redis. Tách `/health` giúp tiến trình vẫn được giữ sống, còn `/ready` loại các instance khỏi nhận traffic cho đến khi Redis phục hồi.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, `history_length` tăng đều theo các lượt hỏi dù request được chuyển tới instance nào. Nếu dùng một dict Python riêng trong từng container, mỗi instance chỉ thấy lịch sử của chính nó: con số sẽ lúc tăng, lúc quay về 0 hoặc một giá trị nhỏ tùy load balancer chuyển request tới container nào; restart container còn làm mất toàn bộ phần lịch sử đó.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy Render, `/health` trả 200 nhưng `/ready` và `/ask` trả 500. Tôi gọi riêng từng endpoint để khoanh vùng và thấy tiến trình web vẫn sống, còn dependency Redis lỗi; sau đó kiểm tra phần Environment của web service và phát hiện `REDIS_URL` chưa trỏ đúng Internal Redis URL. Tôi lấy connection string nội bộ của Render Key Value, đặt lại `REDIS_URL`, lưu cấu hình và redeploy để ứng dụng khởi tạo Redis client đúng.
