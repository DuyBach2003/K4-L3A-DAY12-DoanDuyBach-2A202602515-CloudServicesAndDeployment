# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder ở mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đoàn Duy Bách  Mã học viên: 2A202602515

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Giả sử tôi deploy lên Railway nhưng quên khai báo `AGENT_API_KEY` trong dashboard. Vì `agent_api_key` không có mặc định, `Settings()` ném `ValidationError` ngay lúc khởi động, health check không bao giờ xanh, và Railway giữ nguyên bản cũ đang chạy. Tôi thấy lỗi trong log deploy sau vài giây. Nếu để mặc định `"changeme"`, app vẫn lên và báo healthy, nhưng ai biết khóa `changeme` (nó nằm ngay trong source public của repo) cũng gọi được `/ask` miễn phí và đốt tiền LLM của tôi. Tôi chỉ phát hiện khi hóa đơn tới.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log thu được khi gọi `/ask` (chạy uvicorn thật):

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T10:01:16.558534+00:00", "user_id": "u1", "tokens_in": 1, "tokens_out": 35, "cost_usd": 2.115e-05}
```

Hai việc làm được mà `print("đã trả lời xong")` không làm được: (1) lọc và tổng hợp theo trường, ví dụ `jq 'select(.user_id=="u1") | .cost_usd'` rồi cộng lại để biết một user tốn bao nhiêu, hoặc đưa vào hệ thống log để vẽ biểu đồ token theo thời gian; (2) lọc theo `level` và `event` để đặt cảnh báo khi có `level=error`, còn dòng print thì không có cấu trúc, muốn tách số phải viết regex và vỡ khi đổi câu chữ.

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

Số đo thật trên máy tôi (`docker images`):

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu, `python:3.11`) | 1.2 GB |
| Multi-stage (`python:3.11-slim`) | 234 MB |

Phần chênh lệch (~1 GB) chủ yếu là base image: `python:3.11` đầy đủ mang theo gcc, compiler, header và nhiều công cụ build, trong khi `-slim` chỉ có Python runtime. Multi-stage còn giúp runtime chỉ copy virtualenv `/opt/venv` đã cài xong, không mang theo cache pip hay công cụ build của stage builder. Chúng chỉ phục vụ lúc cài đặt mà lúc chạy không dùng đến.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Tôi thêm một dòng vào `app/main.py` rồi build lại. Kết quả: các layer `FROM`, `venv`, `COPY requirements.txt` và **`RUN pip install`** đều `CACHED` (bước tốn thời gian nhất). Chỉ `COPY app/` và `COPY utils/` phải chạy lại, vì nội dung thư mục đã đổi. Lý do là tôi copy `requirements.txt` riêng và cài dependency trước rồi mới copy source. Nếu đặt `COPY . .` lên trước `RUN pip install`, sửa một ký tự trong code sẽ làm layer copy đổi hash, kéo theo mọi layer sau nó mất cache, và `pip install` chạy lại từ đầu mỗi lần build dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: (1) code có lỗ hổng, ví dụ deserialize dữ liệu không tin cậy hoặc command injection, cho kẻ tấn công thực thi lệnh trong process Python; (2) nếu container chạy root thì process đó là root trong container: đọc được mọi file, ghi được vào filesystem, cài công cụ, đọc biến môi trường và secret; (3) kẻ tấn công lợi dụng cấu hình thừa như socket Docker mount vào, volume ghi được hoặc lỗ hổng kernel/runtime để thoát ra host, và root trong container có UID 0 trùng với root của host nên thoát ra là có quyền cao ngay. Lệnh `USER app` cắt chuỗi ở bước (2): process chạy bằng user thường không có shell, không có home, không ghi được ngoài thư mục được `--chown`, nên đến bước (3) thì thiếu quyền để tiếp tục, và thiệt hại nếu có cũng bị giới hạn trong phạm vi nhỏ.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với hạn mức 10/phút và cách đếm theo phút đồng hồ, người dùng có thể gửi **20 request trong khoảng 2 giây**: 10 request ở giây 59 của phút N (dùng hết hạn mức phút đó), rồi bộ đếm reset lúc giây 00 của phút N+1 và họ gửi thêm 10 request ngay lập tức. Cả 20 đều hợp lệ vì mỗi lần nằm trong một cửa sổ khác nhau, tức gấp đôi hạn mức thật. Sliding window 60 giây tránh được điều này vì với mỗi request nó đếm số request trong 60 giây gần nhất tính từ hiện tại, nên 10 request ở giây 59 vẫn còn được tính lúc giây 00 và request thứ 11 bị chặn.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn **số lượng request** theo thời gian (10/phút, trả 429), còn cost guard giới hạn **tổng tiền** đã chi trong tháng (trả 402). Rate limit cho qua nhưng cost guard chặn: một user gửi đều 5 request/phút, không bao giờ vượt hạn mức, nhưng mỗi request là prompt rất dài tốn nhiều token, cuối tháng tổng chi vượt ngân sách 10 USD và bị chặn 402. Ngược lại, rate limit chặn nhưng cost guard cho qua: một script gửi 100 request/giây với câu hỏi rất ngắn, mỗi cái tốn vài phần triệu USD, tổng tiền còn xa ngân sách nhưng vượt hạn mức 10/phút nên bị 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp làm một endpoint kiểm tra cả Redis và dùng nó làm liveness probe, khi Redis mất kết nối 30 giây thì: (1) Redis chết, `/health` của cả 3 container cùng trả lỗi (vì chúng cùng phụ thuộc một Redis); (2) sau vài lần fail liên tiếp (`retries=3`), orchestrator coi cả 3 container là chết và restart chúng đồng loạt; (3) không còn container nào phục vụ, toàn bộ service sập dù code không hỏng gì; (4) container mới khởi động lại vẫn không nối được Redis nên tiếp tục fail và restart, tạo vòng lặp; (5) khi Redis trở lại sau 30 giây, service còn phải chờ container lên lại. Tách ra thì `/health` chỉ báo process còn sống (không restart oan), còn `/ready` trả 503 khi Redis mất nên load balancer chỉ ngừng gửi traffic, container vẫn sống và tự nhận traffic lại ngay khi Redis về.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Với Redis, cả 3 replica đọc chung một kho nên `history_length` tăng liên tục 0, 1, 2, 3... dù request rơi vào container nào. Nếu lưu trong dict Python, mỗi container có dict riêng trong RAM. Load balancer phân request xoay vòng giữa 3 container nên số này sẽ nhảy lung tung: request 1, 2, 3 đều thấy `history_length` = 0 (mỗi container gặp user lần đầu), rồi request 4, 5, 6 thấy 1, và cứ thế, chỉ tăng bằng khoảng 1/3 tốc độ thật. Mất container (restart, deploy bản mới) thì lịch sử của nó mất luôn. Đó là lý do phải stateless và đưa trạng thái ra ngoài.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi tôi gặp là ở bước smoke test sau deploy trong CI/CD (GitHub Actions gọi vào bản vừa deploy lên Railway). Thông báo: `curl: (3) URL rejected: Malformed input to a URL function`, lặp 12 lần rồi job đỏ. Cách tìm nguyên nhân: đọc log của step, dòng đầu in ra `URL="https://agent-production-302f.up.railway.app "`, có một **dấu cách thừa ở cuối** dấu nháy. Nó đến từ variable `PUBLIC_URL` tôi dán vào GitHub kèm khoảng trắng, nên `curl` từ chối URL. Cách sửa: thêm `URL="$(echo "$URL" | tr -d '[:space:]')"` để workflow tự cắt khoảng trắng, và sửa lại giá trị variable. Trước đó bước deploy còn báo `Unauthorized. Please login with railway login`, và bản Railway CLI ghim `@3` quá cũ, tôi đổi sang `@railway/cli@5` thì deploy chạy được.
