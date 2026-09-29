# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu ngay dưới mỗi câu bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Nguyễn Trí Dũng  Mã học viên: 2A202602784

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ lúc deploy lên Railway, mình quên khai báo `AGENT_API_KEY`. Nếu app fail fast thì bản deploy báo lỗi ngay và mình biết cần bổ sung biến môi trường. Nếu dùng mặc định `changeme`, service vẫn chạy với một khóa dễ đoán, người khác có thể gọi API trước khi mình phát hiện.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log mình thu được khi gọi `/ask`:
>
> ```json
> {"method":"POST","path":"/ask","status":200,"latency_ms":266,"user_id":"sv-test","history_length":20,"cost_usd":0.0001056,"tokens_in":520,"tokens_out":46}
> ```
>
> Hai việc mình làm được với dòng log này mà print("đã trả lời xong") không làm được: heo dõi hiệu năng và chi phí: mình biết request /ask mất 266 ms, dùng 520 input tokens, 46 output tokens và tốn $0.0001056. Từ nhiều log có thể tính latency trung bình, tổng token hoặc tổng chi phí.Tìm kiếm và phân tích lỗi/request cụ thể: vì log có các field như path, status, user_id, mình có thể filter các request lỗi, xem request của một user cụ thể hoặc thống kê số lần gọi /ask. Trong khi print("đã trả lời xong") chỉ cho biết chương trình đã chạy tới đó, không có đủ dữ liệu để phân tích.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```
>
| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.73GB|
| Multi-stage | 271 MB |

> Image multi-stage mình build được là 271 MB.giảm khoảng 1.46 GB
 (xấp xỉ 84%). Bản đầu dùng base `python:3.11` đầy đủ và giữ toàn bộ package,
công cụ hệ thống cùng layer cài đặt trong image chạy. Bản mới dùng
`python:3.11-slim` cho runtime và chỉ copy thư viện Python từ builder, nên
không mang các thành phần chỉ phục vụ build sang production

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, `requirements.txt` không đổi nên các layer cài dependency được dùng lại từ cache. Khi `app/main.py` đổi, bước `COPY app ./app` và các bước sau nó phải chạy lại; các layer trước đó vẫn được giữ. Nếu đặt `COPY . .` trước `pip install`, thay đổi code cũng làm layer `COPY` đổi nên bước cài thư viện sẽ chạy lại mỗi lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu kẻ tấn công khai thác lỗ hổng trong ứng dụng, tiến trình app có thể bị điều khiển và dùng quyền của user đang chạy container để đọc hoặc sửa dữ liệu. Nếu tiến trình đó là root thì quyền bên trong container cao nhất, và lỗ hổng cấu hình hoặc thoát container có thể khiến ảnh hưởng lan tới host. Lệnh `USER appuser` giới hạn tiến trình ở quyền của user thường, nên kẻ tấn công không nhận quyền root chỉ nhờ xâm nhập app.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với hạn mức 10 request/phút, người dùng có thể gửi 10 request ngay trước khi phút kết thúc rồi gửi thêm 10 request ngay sau khi phút mới bắt đầu. Như vậy có thể thành 20 request trong khoảng 2 giây dù mỗi phút đồng hồ đều không vượt 10.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm số request trong một khoảng thời gian; cost guard cộng tiền đã dùng trong tháng. Một người còn quota request nhưng đã gần hết ngân sách có thể bị cost guard chặn với 402. Ngược lại, một người gửi quá 10 request trong một phút nhưng vẫn tiêu ít tiền sẽ bị rate limit chặn với 429 dù ngân sách tháng còn.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu health check gộp chung và kiểm tra Redis, Redis mất kết nối thì cả ba container đều báo health thất bại. Orchestrator có thể lần lượt đánh dấu chúng không khỏe rồi restart chúng, dù tiến trình API vẫn sống. Trong lúc Redis bị mất 30 giây, các lần restart không sửa được Redis và có thể làm cả cụm cùng ngừng phục vụ; tách `/health` (process) khỏi `/ready` (Redis) tránh restart vì một dependency tạm thời.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với lịch sử trong Redis, mọi instance đọc cùng một danh sách nên `history_length` tăng thêm 2 sau mỗi lượt hỏi: 0, 2, 4, ... Nếu dùng dict trong RAM thì mỗi container có lịch sử riêng; khi request chuyển sang instance khác, nó có thể trả về lịch sử ngắn hơn hoặc bắt đầu lại từ 0. Nếu dùng một dictpython, mỗi replica chỉ có lịch sử riêng: kết quả có thể nhảy như `0, 0, 2,0, 2...` tùy request được cân bằng vào container nào; restart container cònlàm phần lịch sử của nó trở về 0.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Đó là khi sai redis_Url nó thông báo /bin/sh: 1: exec: docker-entrypoint.sh: not found/bin/sh: 1: exec: docker-entrypoint.sh: not found tìm ra nguyên nhân bằng cách trace nó lại từ vlearn xem và lab guide để mình thiếu bước nào cũng như nghiên cứu tìm hiểu về nó, sau đó sửa lỗi bằng cách sửa ở phần railway varibles REDIS_URL=${{<ten-service-redis>.REDIS_URL}} và đặt tên riêng REDIS_URL=${{day12-redis.REDIS_URL}} vào cả redis và agent 
