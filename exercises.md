# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `>Câu trả lời của bạn` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Ngô Minh Phong  Mã học viên: 2A202602025
---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> *Câu trả lời:* Tình huống: Chạy trên production nhưng quên cung cấp `AGENT_API_KEY` trong `.env`. Nếu có mặc định "changeme", app vẫn khởi động bình thường (`health` và `ready` pass). Ai đó trên mạng có thể dò ra key "changeme" và gọi API miễn phí làm tốn hàng đống tiền. Việc "chết sớm" (fail fast) lúc khởi động khiến container crash liên tục, mình sẽ phát hiện ra lỗi ngay lập tức qua logs (hoặc app không lên nổi) thay vì phát hiện khi nhận hoá đơn đám mây khổng lồ.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> *Câu trả lời:*
> Log mẫu: `{"event": "ask_completed", "user_id": "sv-test", "tokens_in": 150, "tokens_out": 50, "cost_usd": 0.0002, "timestamp": "2026-08-10T04:36:06Z"}`
> Hai việc làm được:
> 1. Dùng các hệ thống (như Datadog, Kibana) để **truy vấn và tính tổng** lượng `cost_usd` hoặc tokens của một `user_id` cụ thể trong tháng, vì JSON là dạng key-value dễ dàng bóc tách để tính toán.
> 2. Có thể **thiết lập cảnh báo tự động (alert)** nếu trường `cost_usd` trong 1 dòng log vượt quá một ngưỡng nào đó (ví dụ > $0.5), thay vì phải viết regex lằng nhằng để tách chuỗi chữ thập cẩm của `print`.

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
| 1 stage (bản đầu) | 1GB |
| Multi-stage | 164 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> *Câu trả lời:*
> - 1 stage (bản đầu): ~1.0 GB
> - Multi-stage: ~164 MB
> Phần dung lượng chênh lệch khổng lồ (hơn 800MB) chính là hệ điều hành base nặng nề, các công cụ dùng để biên dịch (gcc, curl), trình quản lý gói, và các file rác sinh ra trong quá trình cài đặt thư viện (cache của pip/poetry). Multi-stage loại bỏ toàn bộ môi trường "xây dựng" (build stage) đó, chỉ nhặt đúng thành phẩm cuối (thư viện đã cài và code) để bỏ sang một môi trường "chạy" (runtime stage) siêu mỏng nhẹ (như alpine hoặc slim).

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> *Câu trả lời:*
> - Khi sửa `app/main.py`: Các layer từ `COPY . .` trở về sau sẽ bị mất cache và phải chạy lại. Các layer trước đó (như `COPY requirements.txt` và `RUN pip install`) vẫn sử dụng lại cache cũ.
> - Nếu đặt `COPY . .` lên trước `RUN pip install`: Bất cứ khi nào bạn sửa dù chỉ 1 ký tự trong file code Python, layer `COPY . .` sẽ bị coi là thay đổi. Do nó đứng trước, nó làm sụp đổ toàn bộ cache của các layer phía sau. Hậu quả là Docker sẽ phải chạy lại `RUN pip install` từ đầu (tốn thời gian tải lại toàn bộ package từ mạng) dù file `requirements.txt` không hề thay đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> *Câu trả lời:*
> Lỗ hổng trong code Python bị khai thác (ví dụ RCE) -> Hacker chạy được mã độc từ xa -> Vì container chạy quyền `root`, mã độc cũng chạy quyền `root` -> Hacker khai thác lỗ hổng thoát khỏi container (container breakout) -> Hacker thoát ra ngoài và kiểm soát toàn bộ máy Host của công ty với quyền `root` cao nhất.
> Lệnh `USER nonroot` cắt đứt chuỗi ở đoạn: Mã độc khi bị RCE sẽ bị ép chạy dưới quyền của user `nonroot` bên trong container. Do không có đặc quyền (như sửa file hệ thống, đổi cấu hình mạng), hacker rất khó để khai thác tiếp nhằm leo thang và thoát ra máy Host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> *Câu trả lời:*
> Tối đa 20 request trong 2 giây liên tiếp.
> Giải thích: Giả sử hạn mức là 10/phút đếm theo đồng hồ. Người dùng có thể cố ý gửi dồn dập 10 request lúc `10:00:59` (chưa vượt hạn mức). Vừa sang giây tiếp theo `10:01:00`, bộ đếm của hệ thống tự động reset về 0. Người dùng lập tức gửi tiếp 10 request lúc `10:01:00` (cũng chưa vượt hạn mức mới). Kết quả là hệ thống phải hứng chịu 20 request chỉ trong vòng 2 giây ngắn ngủi. Cửa sổ trượt (sliding window) khắc phục được lỗ hổng này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> *Câu trả lời:*
> - Sự khác biệt: Rate limit giới hạn **tần suất/số lượng** (chống ddos/spam), còn Cost guard giới hạn **ngân sách/tiền** (chống cháy túi).
> - Rate limit cho qua, Cost guard chặn: User gọi rất từ tốn (1 request/phút) nên không bị chặn rate limit, nhưng request đó mang theo câu hỏi cực dài tốn tới 11 USD (vượt ngân sách 10 USD), lúc này Cost guard sẽ chặn lại (402).
> - Rate limit chặn, Cost guard cho qua: User spam nhắn 15 câu "hi" ngắn ngủn trong 1 giây. Chi phí cực kỳ rẻ (Cost guard cho qua), nhưng vì spam dồn dập vượt 10 request/phút nên Rate limit sẽ chặn lại (429).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> *Câu trả lời:*
> Nếu /health kiểm tra cả Redis:
> 1. Redis mất kết nối.
> 2. Endpoint `/health` của cả 3 container đồng loạt báo lỗi (trả về 503).
> 3. Hệ thống quản lý container (như Docker swarm/Kubernetes/Render) liên tục ping `/health` và lầm tưởng cả 3 container app bị treo/đứng máy.
> 4. Hệ thống tự động Kill (giết) cả 3 container và khởi động lại chúng liên tục (CrashLoop).
> 5. Hậu quả: Dịch vụ bị sập hoàn toàn. Đáng lẽ ra app vẫn có thể sống để phục vụ các API không cần Redis (nếu có) hoặc chờ Redis phục hồi, thì nay lại bị giết chết oan uổng. Mất Redis là lỗi dependency, không phải lỗi liveness của process.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> *Câu trả lời:*
> Nếu lưu bằng dict Python: Bộ nhớ RAM của 3 container hoàn toàn độc lập.
> Khi gửi request nhiều lần, Nginx Load Balancer sẽ chia đều (Round Robin) request của bạn vào 3 container khác nhau. Lần 1 vào agent-1, lần 2 vào agent-2, lần 3 vào agent-3... Do đó, `history_length` sẽ không tăng đều đặn (0, 2, 4, 6...) mà sẽ nhảy loạn xạ (ví dụ: 0, 0, 0, 2, 2, 2). Mỗi container chỉ "nhớ" được những câu hỏi vô tình rơi vào trúng nó. Việc dùng Redis (State Store) giải quyết được vì mọi container đều đọc ghi chung một nơi.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> *Câu trả lời:*
> - Lỗi gặp: Request trên Render bị trả về `{"detail":"invalid or missing API key"}` (Mã 401).
> - Cách tìm nguyên nhân: Kiểm tra test log của pytest và chạy tay bằng `curl`, phát hiện ra request báo 401 dù có gửi `X-API-Key`. Đối chiếu lại với cấu hình thì nhận ra biến môi trường cấu hình sai.
> - Cách sửa: Truy cập vào trang Dashboard của Render, vào mục Environment, copy đúng giá trị thật của `AGENT_API_KEY`, mang về máy và dán vào file `.env` (gán cho `DEPLOY_API_KEY`), rồi lưu lại và chạy test là xanh toàn bộ.
