# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> [Câu trả lời của bạn]` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đào Ngọc Duy  Mã học viên: 2A202601780

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Giả sử khi deploy lên Railway, tôi quên set biến API_TOKEN trong dashboard. Nếu có mặc định "changeme", app vẫn khởi động bình thường và bất kỳ ai biết token mặc định đó đều có thể gọi API, gây ra hóa đơn LLM không kiểm soát. Với fail fast, app chết ngay lúc khởi động, pydantic ném ValidationError rõ ràng "api_token Field required" — tôi thấy lỗi ngay trên log deploy và sửa được trước khi có request nào đi qua. Lỗi xuất hiện khi tôi còn đang nhìn màn hình, thay vì âm thầm chạy với token mặc định cho đến khi nhìn hóa đơn.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thu được: `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T10:30:00+00:00", "client_id": "sv01", "prompt_tokens": 45, "completion_tokens": 12, "usd_cost": 0.0001}`
>
> Hai việc làm được: (1) Lọc log theo client_id để tìm ra "client nào tiêu nhiều tiền nhất hôm nay?" bằng truy vấn trên Cloud Logging hoặc jq — với print() thì phải đọc từng dòng text bằng mắt. (2) Tạo cảnh báo tự động khi tỷ lệ lỗi (severity = ERROR) vượt ngưỡng trong 5 phút gần nhất — vì severity là trường có cấu trúc, hệ thống giám sát đọc được nó để lọc và đếm, còn print() thì chỉ là text tự do không lọc được.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1.8 GB |
| Multi-stage | ~300 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần chênh lệch khoảng 1.5GB chủ yếu là: (1) compiler và build tools (gcc, build-essential, các header file C) cần thiết để biên dịch các thư viện Python có native extension, (2) cache pip và các file tạm trong quá trình cài đặt, (3) base image python:3.11 đầy đủ chứa rất nhiều package hệ thống không cần cho runtime. Với multi-stage, stage builder cài hết rồi bị vứt đi — chỉ kết quả (các package đã biên dịch) được COPY sang stage runtime dùng python:3.11-slim gọn nhẹ hơn nhiều.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại (COPY requirements.txt → pip install → COPY app): khi sửa 1 ký tự trong main.py, các layer FROM, COPY requirements.txt, RUN pip install đều được dùng lại từ cache (vì requirements.txt không đổi). Chỉ layer COPY app ./app trở đi phải chạy lại — rất nhanh vì chỉ copy file. Nếu đặt COPY . . trước pip install: mỗi lần sửa bất kỳ file nào (kể cả 1 dấu phẩy trong main.py), Docker thấy context thay đổi → invalidate cache từ COPY . . trở đi → phải cài lại toàn bộ thư viện từ đầu, mất vài phút mỗi lần build.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) Code Python có lỗ hổng RCE (ví dụ injection qua input không sanitize) → kẻ tấn công thực thi lệnh tùy ý bên trong container. (2) Container chạy root → lệnh chạy với quyền root, kẻ tấn công đọc/ghi mọi file trong container, mount filesystem host, hoặc khai thác lỗ hổng kernel escape (ví dụ CVE container breakout) để thoát ra máy host với quyền root. (3) Lệnh `USER appuser` cắt đứt chuỗi ở bước 2: dù kẻ tấn công chạy được lệnh, nó chỉ có quyền của user thường (uid 10001), không đọc được file nhạy cảm, không mount được, và không khai thác được các lỗ hổng yêu cầu quyền root để escape ra host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Header WWW-Authenticate: Bearer là bắt buộc theo chuẩn HTTP RFC 7235 — khi server trả 401, nó phải nói cho client biết cần xác thực kiểu gì (Bearer, Basic, Digest...) để client hoặc thư viện HTTP tự xử lý. Trả cùng một thông báo lỗi cho cả 3 trường hợp là để tránh rò rỉ thông tin (information leakage): nếu ta nói rõ "sai scheme" vs "sai token" vs "thiếu header", kẻ tấn công biết được mình đã đi đúng hướng đến đâu — ví dụ biết scheme đã đúng thì chỉ cần brute force token. Thông báo chung "invalid or missing bearer token" không cho biết sai ở bước nào, làm tăng đáng kể chi phí tấn công.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Gửi được 10 request trước khi bị 429. Dù im lặng 10 phút (tích lũy 10×10=100 token lý thuyết), nhưng `min(capacity, tokens)` giới hạn tối đa ở 10 — đúng bằng sức chứa xô. Nếu bỏ `min(capacity, ...)`: client im lặng 10 phút sẽ tích được 100 token, gửi được 100 request liên tiếp trước khi bị 429. Xô không có giới hạn trên nên thời gian im lặng càng dài, "burst" cho phép càng lớn — client im lặng 1 ngày sẽ tích 14.400 token và bắn hết trong vài giây, vô hiệu hóa hoàn toàn mục đích rate limit.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức $30/tháng: sự cố từ 2h sáng có thể đốt hết $30 trong một ngày (hoặc vài giờ nếu tốc độ gọi cao). Service chỉ hồi phục đầu tháng sau — tức phải chờ đến hết tháng, hoặc cần người can thiệp thủ công reset. Thiệt hại tối đa: $30. Hạn mức $1/ngày: sự cố chỉ đốt được tối đa $1 (phần còn lại trong ngày). Sang 0h UTC ngày hôm sau, key Redis mới (theo ngày) tự động reset ngân sách, service tự hồi phục mà không cần ai can thiệp. Thiệt hại tối đa: $1 — nhỏ hơn 30 lần so với hạn mức tháng.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> (1) Redis mất kết nối. (2) Cả 3 container gọi ping() đều thất bại → endpoint gộp trả 503. (3) Orchestrator (K8s/Docker) thấy liveness check fail → bắt đầu restart cả 3 container cùng lúc. (4) Trong lúc restart, không có container nào phục vụ request → downtime toàn bộ. (5) Redis quay lại sau 30 giây, nhưng các container đang restart, chưa sẵn sàng. (6) Container khởi động xong → lại healthy → nhận traffic. Kết quả: một sự cố Redis tạm thời 30 giây biến thành downtime toàn hệ thống vài phút. Nếu tách riêng: /healthz không kiểm tra Redis nên vẫn trả 200 → container không bị restart. /readyz trả 503 → LB ngừng gửi request mới nhưng không kill container. Redis quay lại → /readyz trả 200 → LB gửi traffic trở lại. Không có downtime.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp: Container khởi động rồi tắt ngay với log `pydantic.ValidationError: 1 validation error for Settings, api_token: Field required`. Nguyên nhân: quên set biến API_TOKEN trong dashboard của platform — Settings yêu cầu bắt buộc mà container không có file .env (bị .dockerignore loại trừ đúng cách). Cách tìm ra: chạy `docker compose logs chat` thấy ngay dòng ValidationError. Cách sửa: vào dashboard platform → Variables → thêm API_TOKEN với giá trị token đã sinh bằng `python -c "import secrets; print(secrets.token_urlsafe(32))"` → redeploy → container khởi động thành công.
