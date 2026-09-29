# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vu Duc Thien  Mã học viên: 2A202602437

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống: khi tạo service trên Render mà quên nhập `AGENT_API_KEY`. Nếu có
mặc định `"changeme"`, app vẫn khởi động, `/health` vẫn 200 nên Render báo
"Live", và ai đoán được chữ `changeme` (chuỗi mặc định rất phổ biến) là gọi
`/ask` bằng tiền của mình. Mình chỉ phát hiện ra khi nhìn hóa đơn. Không có
mặc định thì deploy đỏ ngay, log ghi rõ thiếu biến gì, sửa trong 1 phút.

Mình còn tự gặp một biến thể của lỗi này. Ban đầu mình chỉ khai báo trường
bắt buộc, nhưng `Settings` chỉ được đọc lần đầu khi có request `/ask`. Chạy
`docker run --rm day12-agent:prod` (không truyền key) thì container **vẫn
chạy**, `/health` vẫn 200, chỉ tới `/ask` mới lỗi 500. Tức là "fail" nhưng
không "fast". Mình sửa bằng cách gọi `get_settings()` trong `lifespan` lúc
khởi động. Chạy lại thì container thoát ngay với:

```
pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
agent_api_key
  Field required [type=missing, input_value={'port': '8000'}, input_type=dict]
ERROR:    Application startup failed. Exiting.
```

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được khi gọi `/ask` qua `docker compose`:

```
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T13:26:53.929235+00:00", "user_id": "sv-scale-redis", "tokens_in": 44, "tokens_out": 46, "cost_usd": 3.42e-05}
```

Hai việc làm được mà `print` không làm được:

1. **Lọc và tìm theo trường.** Có thể lọc đúng `user_id = "sv-scale-redis"`
   để xem một user đã hỏi những gì, lúc nào. Chính mình đã làm việc này khi
   scale 3 container: lọc các dòng `"ask_completed"` để đếm mỗi container nhận
   bao nhiêu request. Với `print("đã trả lời xong")` thì không biết của ai.
2. **Cộng dồn số liệu để giám sát và cảnh báo.** Cộng `cost_usd` hoặc
   `tokens_in + tokens_out` theo user, theo giờ để vẽ biểu đồ chi phí, hoặc
   đặt cảnh báo khi một user tiêu bất thường. Chuỗi văn bản tự do thì phải viết
   regex riêng, mà đổi câu chữ một chút là regex hỏng.

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
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage | 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Chênh khoảng 1.4 GB. Xem `docker history` thì layer cài thư viện ở cả hai bản
đều khoảng 95 MB (`RUN pip install` ở bản 1 stage, `COPY /opt/venv` ở bản
multi-stage), code chỉ vài trăm KB. Vậy gần như toàn bộ phần chênh là **base
image**: `python:3.11` bản đầy đủ chứa sẵn cả một hệ Debian cho việc build
(gcc, make, header C, git, thư viện dev...), còn `python:3.11-slim` bỏ hết
những thứ đó.

Với project này, thư viện đều có wheel dựng sẵn nên không phải biên dịch gì.
Phần tiết kiệm chủ yếu đến từ việc đổi sang base `slim`. Multi-stage đảm bảo
nếu sau này có thư viện phải biên dịch, compiler chỉ nằm ở stage `builder` và
bị bỏ lại, không lọt vào image chạy thật.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Mình đổi `SERVICE_VERSION = "1.0.0"` thành `"1.0.1"` rồi build lại
(`--progress=plain`):

- **Dùng lại cache (CACHED):** `RUN python -m venv`, `COPY requirements.txt`,
  `RUN pip install`, `RUN groupadd/useradd`, `WORKDIR`, `COPY --from=builder
  /opt/venv`.
- **Chạy lại:** chỉ `COPY app/ ./app/` và các lệnh sau nó (`COPY utils/`).
- Thời gian build: **2.1 giây**.

Với Dockerfile gốc đặt `COPY . .` trước `RUN pip install`, sửa đúng một ký
tự đó thì layer `COPY . .` đổi, kéo theo **mọi layer sau nó phải chạy lại**,
kể cả `pip install`. Thời gian build: **269 giây**, tức chậm hơn khoảng 130
lần chỉ vì thứ tự hai dòng lệnh. Docker cache theo chuỗi: một layer thay đổi
thì mọi layer phía sau mất cache, nên thứ ít đổi (thư viện) phải đặt trước thứ
hay đổi (code).

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện khi chạy bằng root:

1. Code có lỗ hổng cho phép chạy lệnh tùy ý (ví dụ một thư viện có lỗi
   deserialize, hoặc lỡ đưa input của user vào `subprocess`).
2. Kẻ tấn công chạy lệnh shell **với quyền root trong container**: đọc mọi
   file, đọc biến môi trường chứa `AGENT_API_KEY`, cài công cụ tấn công.
3. Container chỉ là process được cô lập bằng namespace và **dùng chung kernel
   với host**. Root trong container và root trên host là cùng một UID 0, chỉ
   ngăn cách bởi lớp cô lập đó.
4. Chỉ cần một lỗi thoát container (lỗ hổng kernel hoặc runtime, hay cấu hình
   sai như mount `/var/run/docker.sock` hoặc chạy `--privileged`), kẻ tấn công
   trở thành root trên máy host và điều khiển mọi container khác.

`USER app` cắt chuỗi này **ở bước 2**: lệnh của kẻ tấn công chạy bằng một user
thường không có quyền root. Họ không ghi được vào file hệ thống, không cài
được gói, và nếu có thoát ra host thì cũng chỉ là một user không đặc quyền.
Mình kiểm tra `docker compose exec agent whoami` thì nhận được `app`, không
phải `root`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa **20 request trong 2 giây**:

- Lúc 10:00:59, gửi 10 request. Bộ đếm của phút 10:00 là 10/10, vẫn hợp lệ.
- Lúc 10:01:00, bộ đếm reset về 0.
- Lúc 10:01:00–10:01:01, gửi tiếp 10 request. Bộ đếm của phút 10:01 là
  10/10, vẫn hợp lệ.

Kết quả là 20 request trong khoảng 2 giây, gấp đôi hạn mức, mà không vi phạm
luật nào.

Với sliding window, ở giây 10:01:01 hệ thống đếm các request trong 60 giây
gần nhất (từ 10:00:01), vẫn thấy đủ 10 request lúc 10:00:59, nên request thứ
11 bị chặn 429. Kết quả chạy thật trên Render khớp với điều này: 15 request
liên tiếp cho `200` × 10 rồi `429` × 5.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- **Rate limit** giới hạn **số lượng request trong một khoảng thời gian ngắn**
  (10 request/60 giây). Nó bảo vệ hệ thống khỏi bị spam hoặc quá tải, và
  không quan tâm mỗi request tốn bao nhiêu.
- **Cost guard** giới hạn **tổng số tiền trong cả tháng** (10 USD/user/tháng).
  Nó bảo vệ ngân sách, và không quan tâm request đến nhanh hay chậm.

**Rate limit cho qua, cost guard chặn:** một user gửi đều đặn 5 request/phút
(dưới hạn mức 10), nhưng mỗi request là một câu hỏi rất dài kèm lịch sử 20
message, tốn nhiều token. Sau vài ngày tổng chi phí vượt 10 USD, nên cost
guard trả **402** dù tốc độ gọi hoàn toàn hợp lệ. Test
`test_qua_http_thi_tra_402` mô phỏng đúng việc này bằng cách đặt sẵn chi phí
999 USD cho user.

**Cost guard cho qua, rate limit chặn:** user mới đầu tháng, mới tiêu 0.0001
USD, nhưng một script lỗi gọi `/ask` 15 lần trong 1 giây. Ngân sách còn gần
đủ nhưng request thứ 11 bị **429**. Mình đã thấy đúng như vậy khi test trên
Render (`200` × 10 rồi `429` × 5), trong khi mỗi request chỉ tốn khoảng
0.00002 USD.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. **Giây 0:** Redis mất kết nối.
2. Lần kiểm tra tiếp theo, endpoint gộp gọi Redis thất bại nên **cả 3
   container** cùng trả 503, vì cả 3 dùng chung một Redis.
3. Orchestrator hiểu đây là tín hiệu liveness "process đã chết", nên sau vài
   lần thất bại liên tiếp nó **restart cả 3 container**, dù code Python bên
   trong không có lỗi gì.
4. Trong lúc restart, không còn container nào nhận request. Load balancer
   không có chỗ để gửi, **toàn bộ service sập** (502/503), kể cả những việc
   không cần Redis.
5. Container mới khởi động xong, nhưng nếu Redis vẫn chưa về thì health check
   lại fail, lại restart, thành vòng lặp restart liên tục.
6. **Giây 30:** Redis trở lại, nhưng các container còn phải chờ khởi động
   lại xong, nên thời gian gián đoạn **dài hơn 30 giây**. Request đang xử lý
   dở lúc bị restart cũng mất.

Tách riêng thì Redis mất 30 giây chỉ làm `/ready` trả 503. Load balancer tạm
ngừng gửi traffic, `/health` vẫn 200 nên không container nào bị restart, và
Redis về là `/ready` tự trở lại 200. Mình đã thử thật bằng `docker compose
stop redis`: `/health` vẫn **200**, `/ready` trả **503**
`{"status":"not ready","redis":false}`, bật lại Redis thì `/ready` về 200 ngay.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Mình chạy 3 container `agent` sau Nginx (round-robin), gọi `/ask` 6 lần với
cùng một user. Log cho thấy mỗi container nhận đúng 2 request.

**Lịch sử lưu trong Redis:**

```
req1: history_length=0
req2: history_length=2
req3: history_length=4
req4: history_length=6
req5: history_length=8
req6: history_length=10
```

Con số tăng đều thêm 2 mỗi lượt (1 câu hỏi + 1 câu trả lời), dù mỗi request
rơi vào một container khác nhau, vì cả 3 cùng đọc và ghi một Redis.

**Lịch sử lưu trong bộ nhớ của từng process:** để mô phỏng dict Python, mình
chạy lại với `REDIS_URL=fake://` (Redis giả nằm trong RAM của mỗi process):

```
req1: history_length=0
req2: history_length=0
req3: history_length=0
req4: history_length=2
req5: history_length=2
req6: history_length=2
```

3 request đầu rơi vào 3 container khác nhau, container nào cũng thấy lịch sử
trống. Từ vòng thứ 2, mỗi container chỉ nhớ được đúng 1 lượt nó đã tự xử lý.
Người dùng sẽ thấy agent lúc nhớ lúc quên tùy request rơi vào container nào,
và khi container restart thì mất hết.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

**Thông báo:** sau khi Render báo deploy thành công, mình mở URL
`https://day12-agent-qv1c.onrender.com` trên trình duyệt và nhận được:

```
{"detail":"Not Found"}
```

Ban đầu mình tưởng deploy hỏng.

**Tìm nguyên nhân:** mở tab **Logs** của service trên Render thì thấy:

```
INFO:     10.27.25.223:0 - "GET / HTTP/1.1" 404 Not Found
INFO:     10.213.24.140:52668 - "GET /health HTTP/1.1" 200 OK
==> Your service is live 🎉
```

Hai điểm quan trọng rút ra từ log:

- Request đã tới được app (có dòng log của uvicorn), và response là JSON của
  FastAPI chứ không phải trang lỗi của Render. Vậy service đang chạy.
- Cùng lúc đó health check `/health` của Render vẫn 200. Lỗi chỉ xảy ra ở
  đường dẫn `/`, mà app không định nghĩa route cho `/`, chỉ có `/health`,
  `/ready` và `/ask`.

Log còn có dòng `Detected service running on port 10000`, xác nhận app đọc
đúng `$PORT` do Render cấp (qua `${PORT:-8000}` trong Dockerfile), nên loại
được khả năng lỗi cổng.

**Sửa:** không cần sửa code. Mình dùng đúng đường dẫn: `/health` trả
`{"status":"ok",...}`, `/ready` trả `{"status":"ready","redis":true}`, và
`/docs` để thử API. Bài học là trước khi kết luận "deploy hỏng", phải đọc log
để phân biệt service chết với request gọi sai endpoint.
