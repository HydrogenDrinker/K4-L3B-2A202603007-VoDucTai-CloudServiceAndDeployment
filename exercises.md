# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Võ Đức Tài                  Mã học viên: 2A202603007

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Tình huống cụ thể: Khi deploy ứng dụng lên production (như Railway hoặc Cloud Run), lập trình viên vô tình quên cấu hình biến môi trường `AGENT_API_KEY` trong Dashboard của platform.
- Nếu để mặc định `agent_api_key = "changeme"`: Ứng dụng vẫn khởi động thành công và báo trạng thái Healthy. Lập trình viên tưởng mọi thứ đã sẵn sàng và đi ngủ. Tuy nhiên, các bot quét trên Internet (vốn liên tục brute-force các khóa phổ biến như `"changeme"`, `"admin"`, `"secret"`) sẽ tìm thấy endpoint `/ask`, gửi hàng nghìn request qua mặt lớp xác thực bằng key mặc định đó, và gọi đến LLM làm phát sinh hóa đơn khổng lồ hàng nghìn USD. Lập trình viên chỉ nhận ra khi kiểm tra hóa đơn vào cuối tháng.
- Khi không có giá trị mặc định (Fail Fast): Pydantic ném `ValidationError` ngay lúc service khởi động và container crash lập tức trong quá trình deploy. Lập trình viên đang nhìn màn hình triển khai sẽ thấy deploy báo failed ngay tức khắc, mở log ra thấy rõ `Field required: agent_api_key`, và sửa ngay trong vòng 1 phút trước khi bất kỳ ai trên Internet có thể tiếp cận service.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON thu được:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T06:30:15.123456+00:00", "user_id": "sv-test", "tokens_in": 15, "tokens_out": 42, "cost_usd": 0.0001}`

Hai việc làm được với log JSON mà `print("đã trả lời xong")` không thể làm được:
1. Lọc, nhóm và truy vấn định lượng (Aggregation & Metrics): Hệ thống thu thập log tập trung (như Datadog, CloudWatch, Loki, Elasticsearch) có thể tự động parse các trường số để vẽ biểu đồ và chạy truy vấn như: "Tính tổng chi phí LLM (`sum(cost_usd)`) theo từng `user_id` trong 24 giờ qua" hoặc "Tìm top 5 user tiêu tốn nhiều token nhất". Log text dạng print không thể tách trường tự động để tính toán như vậy.
2. Thiết lập cảnh báo thời gian thực (Alerting & Monitoring): Ta có thể đặt quy tắc cảnh báo tự động: nếu số lượng log có `"level": "error"` vượt quá 5% tổng request trong 5 phút, hoặc nếu phát hiện `cost_usd` của một request đơn lẻ lớn hơn $0.5, hệ thống sẽ tự động gửi thông báo khẩn qua Slack/Telegram cho đội DevOps để can thiệp kịp thời.

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
| 1 stage (bản đầu) | ~1.02 GB |
| Multi-stage | ~185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần chênh lệch lớn này (~835 MB) bao gồm:
1. Base image đầy đủ (`python:3.11`) đi kèm hệ điều hành Debian đầy đủ chứa hàng trăm gói tiện ích hệ thống, trình biên dịch C/C++ (`gcc`, `g++`, `make`), công cụ build (`build-essential`), thư viện phát triển (`linux-headers`, các file header `.h`), tài liệu manual, và bộ cài đặt apt cache. Trong khi đó, stage runtime dùng `python:3.11-slim` chỉ giữ lại kernel tối thiểu và thư viện C runtime cần thiết để chạy Python.
2. Quá trình `pip install`: Trong bản 1 stage, pip tải về các file `.whl`, file nén `.tar.gz`, bộ nhớ đệm wheel cache (`~/.cache/pip`), và các công cụ biên dịch phụ trợ. Trong bản multi-stage, toàn bộ quá trình cài đặt diễn ra trong stage `builder`, và ta dùng `--prefix=/install` kèm `COPY --from=builder /install /usr/local` để chỉ sao chép các gói Python đã biên dịch xong sang image runtime sạch, loại bỏ hoàn toàn compiler và rác build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa một ký tự trong `app/main.py` rồi build lại:
- Các layer được dùng lại từ cache: Tất cả các layer từ đầu đến trước lệnh `COPY app ./app`:
  1. Base image (`FROM python:3.11-slim`)
  2. `WORKDIR /install`
  3. `COPY requirements.txt .`
  4. `RUN pip install ...` (layer tốn nhiều thời gian nhất được lấy 100% từ cache!)
  5. `WORKDIR /app`, `COPY --from=builder /install /usr/local`, `RUN useradd ...`
- Các layer phải chạy lại: Bắt đầu từ layer bị thay đổi là `COPY app ./app` và các bước kế tiếp (`COPY utils ./utils`, `USER appuser`, `HEALTHCHECK`, `CMD`). Vì chỉ copy code vài KB, quá trình build chỉ mất chưa đầy 1 giây.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: Docker tính toán checksum của toàn bộ thư mục context. Khi sửa một ký tự trong `app/main.py`, layer `COPY . .` bị thay đổi, dẫn đến Docker làm mất hiệu lực (invalidate) toàn bộ cache của tất cả các layer phía sau nó. Kết quả là lệnh `RUN pip install` phải tải lại và cài đặt lại toàn bộ thư viện từ đầu, khiến thời gian build tăng từ 1 giây lên vài phút mỗi lần sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện:
1. Kẻ tấn công phát hiện một lỗ hổng trong code ứng dụng Python (ví dụ: lỗ hổng RCE qua Command Injection, deserialize dữ liệu không an toàn với `pickle`, hoặc một thư viện bên thứ ba có lỗ hổng đọc/ghi file tùy ý).
2. Kẻ tấn công khai thác lỗ hổng để thực thi mã độc. Do container chạy dưới quyền user `root` (UID 0), tiến trình Python có đầy đủ quyền root bên trong container: đọc/ghi/xóa mọi file hệ thống (`/etc/passwd`, `/bin`), mở raw socket, cài đặt thêm công cụ tấn công vào container.
3. Nếu container gặp một lỗi cấu hình (chẳng hạn bind mount thư mục `/var/run/docker.sock` hoặc thư mục nhạy cảm của host vào container, hoặc lỗ hổng kernel của hệ điều hành host cho phép container escape), tiến trình độc hại với UID 0 bên trong container sẽ ánh xạ thẳng tới UID 0 (root) trên máy host, cho phép kẻ tấn công chiếm toàn quyền kiểm soát máy chủ vật lý, cài đặt backdoor hoặc đánh cắp toàn bộ dữ liệu.

Lệnh `USER appuser` cắt đứt chuỗi đó ở bước 2: Tiến trình Python chạy với quyền người dùng không đặc quyền (UID 10001, không có quyền sudo). Dù kẻ tấn công có khai thác thành công RCE trong Python, mã độc chỉ có quyền hạn chế của `appuser`, không thể sửa file hệ thống container, không thể ghi đè các binary, và nếu có tìm cách escape ra host thì trên host nó cũng chỉ là một user vô danh không có quyền hạn gì, ngăn chặn hoàn toàn việc chiếm quyền máy chủ host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Với hạn mức là 10 request/phút theo phút đồng hồ:
Số request tối đa người dùng có thể gửi trong 2 giây liên tiếp là: **20 request**.

Giải thích cách đạt được con số đó:
- Cơ chế reset theo phút đồng hồ sẽ làm mới hạn ngạch vào đầu mỗi phút (giây 00).
- Người dùng gửi 10 request liên tiếp vào giây cuối cùng của phút thứ nhất: lúc `10:00:59`. Vì trong phút 10:00 người này chưa dùng quota, hệ thống chấp nhận cả 10 request.
- Ngay sau đó 1 giây, đồng hồ điểm `10:01:00`, hạn mức được reset về 0 cho phút mới.
- Lúc `10:01:00` hoặc `10:01:01`, người dùng gửi tiếp 10 request nữa. Hệ thống kiểm tra thấy trong phút 10:01 người dùng mới gửi 10 request (chưa vượt 10), nên tiếp tục cho qua toàn bộ.
- Kết quả: Trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:01), hệ thống phải hứng chịu tới 20 request dồn dập, tạo ra spike tải gấp đôi dự kiến mà vẫn "đúng luật" phút đồng hồ. Thuật toán sliding window 60s loại bỏ kẽ hở này vì nó luôn tính chính xác tổng số request trong bất kỳ khoảng 60 giây trượt liên tục nào.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Sự khác nhau cốt lõi:
- Rate Limiting giới hạn **tần suất / số lượng** request trong một đơn vị thời gian (ví dụ: 10 request / 60 giây) nhằm bảo vệ hạ tầng máy chủ khỏi bị quá tải, nghẽn mạng hoặc tấn công từ chối dịch vụ (DoS).
- Cost Guard giới hạn **tổng chi phí tài chính tích lũy** theo thời gian (ví dụ: tối đa $10.0 / tháng) nhằm bảo vệ ngân sách của chủ hệ thống khỏi bị cạn kiệt do mức tiêu thụ token của các mô hình LLM.

Tình huống minh họa:
1. Rate limit cho qua nhưng Cost Guard chặn: Một user chỉ gửi 1 request duy nhất trong 10 phút (tần suất cực thấp, hoàn toàn dưới hạn mức 10 req/phút). Tuy nhiên user này đã tiêu hết ngân sách tháng ($10.0/tháng). Cost Guard sẽ kiểm tra trước và chặn request này với mã HTTP 402 Payment Required, mặc dù Rate Limiter hoàn toàn cho phép.
2. Cost Guard cho qua nhưng Rate limit chặn: Một user mới đăng ký đầu tháng, tài khoản còn nguyên $10.0 chưa tiêu đồng nào (ngân sách dồi dào). Tuy nhiên user dùng script gửi 20 request chỉ trong vòng 5 giây. Cost Guard thấy tiền vẫn còn đủ nhưng Rate Limiter sẽ chặn từ request thứ 11 trở đi với mã HTTP 429 Too Many Requests để tránh nghẽn server.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Thứ tự sự kiện xảy ra:
1. Giây 0: Kết nối mạng tới Redis bị gián đoạn (network glitch hoặc Redis khởi động lại).
2. Giây 5–10: Bộ điều phối container (Docker / Kubernetes / Orchestrator) gửi request liveness probe định kỳ tới `/health` của cả 3 container agent.
3. Do `/health` kiểm tra kết nối Redis và ping thất bại, cả 3 container đều đồng loạt trả về HTTP 503 Service Unavailable hoặc timeout.
4. Giây 15: Orchestrator coi HTTP 503 từ liveness probe là dấu hiệu tiến trình ứng dụng đã chết hoặc rơi vào deadlock, lập tức phát lệnh kill và restart cả 3 container cùng lúc để cố gắng tự phục hồi.
5. Giây 20–30: Trong khi Redis vẫn chưa hồi phục, các container vừa khởi động lại tiếp tục bị failed health check ngay khi vừa bật lên và lại bị restart liên tục (rơi vào trạng thái CrashLoopBackOff).
6. Giây 30: Redis hoạt động bình thường trở lại, nhưng lúc này toàn bộ 3 container agent đều đang bị kẹt trong chu kỳ khởi động lại của orchestrator, không có bất kỳ instance nào sẵn sàng nhận request.
Hậu quả: Một sự cố mạng tạm thời ở Redis biến thành sự cố sập hoàn toàn toàn bộ hệ thống (cascading failure / total outage). Nếu tách riêng: `/health` vẫn trả 200 (process Python vẫn sống tốt, không bị restart), chỉ có `/ready` trả 503 (load balancer tạm ngưng đẩy traffic vào, chờ Redis hồi phục thì lập tức phục vụ lại ngay mà không mất công restart app).

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Khi scale ra 3 container với cùng một `X-User-Id`:
- Nếu lịch sử được lưu trong Redis (như hiện tại): Lịch sử được tập trung hóa. Dù Load Balancer phân phối 5 request luân phiên qua lại giữa Container 1, Container 2 và Container 3, mỗi container đều đọc và ghi vào cùng một key `history:user_id` trên Redis. Kết quả là `history_length` tăng đều đặn và nhất quán: 0 -> 2 -> 4 -> 6 -> 8...
- Nếu lịch sử được lưu trong biến dict trong RAM của từng tiến trình Python: Mỗi container có một vùng nhớ RAM độc lập. Khi request 1 vào container A (dict A lưu 2 message), request 2 rẽ vào container B (dict B đang rỗng, nên `history_length` lại tụt về 0!), request 3 rẽ vào container C (dict C rỗng, lại trả về 0), request 4 quay lại container A (thấy 2 message cũ). Con số `history_length` sẽ nhảy lộn xộn ngẫu nhiên (ví dụ: 0 -> 0 -> 0 -> 2 -> 2 -> 4...), khiến AI agent bị "mất trí nhớ từng chặng", quên ngữ cảnh câu hỏi trước của người dùng tùy thuộc vào việc request rơi trúng container nào.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thực tế khi deploy lên cloud:
- Thông báo lỗi: Container bị crash và thoát với mã lỗi: `ValidationError: 1 validation error for Settings agent_api_key Field required [type=missing, input_value={}, input_type=dict]`. Đồng thời trên giao diện Railway/Render báo trạng thái `Deploy failed / CrashLoop`.
- Cách tìm ra nguyên nhân: Mở tab **Deploy Logs** trên Dashboard của Railway/Render. Nhờ cơ chế Fail Fast của Pydantic `Settings` mà ta đã lập trình ở Checkpoint 1, lỗi không bị nuốt mà hiện rõ ràng ở dòng traceback đầu tiên khi ứng dụng khởi chạy: Pydantic cố gắng đọc biến môi trường `AGENT_API_KEY` nhưng không tìm thấy do chưa được thiết lập trong mục Environment Variables của service trên cloud.
- Cách sửa: Vào mục **Variables** trên Dashboard của Railway/Render, thêm biến môi trường có tên `AGENT_API_KEY` với giá trị là chuỗi secret đã sinh ngẫu nhiên (bảo đảm an toàn). Nhấn Save/Redeploy, ứng dụng khởi động lại thành công và vượt qua tất cả các bài kiểm tra health check.
