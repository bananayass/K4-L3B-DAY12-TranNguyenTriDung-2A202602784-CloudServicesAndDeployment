# DAY12 — Cloud Services and Deployment

## Yêu cầu môi trường

Bạn cần:

- Python 3.11 trở lên;
- Git và tài khoản GitHub;
- Docker Desktop hoặc Docker Engine cùng Docker Compose;
- tài khoản Railway hoặc Render cho CP5.

Clone repository cá nhân đã fork:

```bash
git clone <URL-repository-cá-nhân>
cd K4-L3B-DAY12-HoVaTen-MSSV-CloudServicesAndDeployment
```

### macOS/Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
cp .env.example .env
```

### Windows PowerShell

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
```

Sinh API key cá nhân:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

Dán giá trị vào `AGENT_API_KEY` trong `.env`.

> Không commit `.env` hoặc đưa secret vào screenshot, `DEPLOYMENT.md`, source hay prompt AI.

Bật Redis:

```bash
docker compose up -d redis
docker compose ps
```

Nếu chưa dùng được Docker ở CP0/CP1, tạm đặt:

```env
REDIS_URL=fake://
```

trong `.env`.

> CP2 và cloud deployment vẫn cần container thật.

---

# Tiến độ 240 phút

Ghi nhận thời điểm bắt đầu là `Start` và dùng các mốc tương đối sau:

| Giai đoạn | Thời gian | Trọng tâm | Điểm |
|---|---:|---|---:|
| CP0 | Start +0–20 phút | Setup, repo và baseline | — |
| CP1 | Start +20–60 phút | 12-Factor config, health và logging | 15 |
| CP2 | Start +60–105 phút | Docker production-ready | 15 |
| Giải lao | Start +105–115 phút | Nghỉ 10 phút | — |
| CP3 | Start +115–160 phút | Authentication, rate limit, cost guard | 20 |
| CP4 | Start +160–200 phút | Stateless, probes, graceful shutdown | 20 |
| CP5 | Start +200–230 phút | Cloud deployment | 15 |
| Wrap-up | Start +230–240 phút | Exercises, grade và nộp bài | 15 |

Nếu kẹt một checkpoint quá 10 phút, ghi lại lỗi, hỏi Lab Coach và chuyển sang phần tiếp theo.

Điểm checkpoint được tính theo tỷ lệ test pass nên không nên bỏ toàn bộ phần sau chỉ vì một lỗi chưa giải quyết được.

---

# CP0 — Setup và baseline

Đọc nhanh các file sau trước khi sửa code:

- `README.md`: mục tiêu, timeline và quy tắc đặt tên;
- `LAB_GUIDE.md`: hướng dẫn kỹ thuật chi tiết;
- `CHECKPOINTS.md`: điều kiện hoàn thành từng checkpoint;
- `RUBRIC.md`, `RULES.md`, `SUBMISSION.md`: cách chấm và nộp bài.

Chạy baseline:

```bash
pytest tests/ -v -m "not docker"
```

Ở CP0, phần lớn test rớt là bình thường vì starter còn `TODO` và `NotImplementedError`.

Bạn chỉ cần xác nhận:

- pytest chạy được;
- không có `ModuleNotFoundError` hoặc lỗi cài môi trường;
- `.env` đã có `AGENT_API_KEY` cá nhân;
- Redis chạy được hoặc đã dùng `fake://` tạm thời;
- repository cá nhân có tên đúng.

Commit checkpoint đầu tiên:

```bash
git add .
git commit -m "checkpoint 0: setup environment"
git push origin main
```

---

# CP1 — 12-Factor Config, Health và Logging

## 6.1. Cấu hình trong `app/config.py`

Khai báo đủ sáu trường trong `Settings`:

| Trường | Kiểu | Mặc định |
|---|---|---|
| `port` | `int` | `8000` |
| `agent_api_key` | `str` | không có mặc định |
| `redis_url` | `str` | `redis://localhost:6379/0` |
| `rate_limit_per_minute` | `int` | `10` |
| `monthly_budget_usd` | `float` | `10.0` |
| `log_level` | `str` | `INFO` |

`agent_api_key` phải là trường bắt buộc.

Nếu cloud thiếu secret, ứng dụng phải **fail fast** ngay khi khởi động thay vì chạy với khóa mặc định không an toàn.

## 6.2. Structured logging trong `app/logging_utils.py`

Hoàn thiện `log_event()` để mỗi event được in thành đúng **một dòng JSON**.

Log cần có tối thiểu:

- `event`;
- `level`;
- `timestamp`;
- các field bổ sung được truyền vào.

Không dùng `indent` vì nền tảng cloud thu log theo từng dòng.

Dùng UTF-8 / `ensure_ascii=False` để nội dung tiếng Việt không bị biến dạng.

## 6.3. Liveness endpoint `/health`

Hoàn thiện `/health` trong `app/main.py`:

```text
Bình thường  → 200 {"status":"ok", "service":..., "version":...}
Đang tắt    → 503 {"status":"shutting_down"}
```

`/health` không được gọi Redis, database hoặc dependency ngoài.

Endpoint này chỉ trả lời process còn sống hay cần restart.

Chạy thử:

```bash
uvicorn app.main:app --reload --port 8000
curl -i http://localhost:8000/health
```

Checkpoint:

```bash
pytest tests/test_cp1.py -v
```

Hoàn thành CP1 khi:

- test pass;
- log JSON nằm trên một dòng;
- `/health` không phụ thuộc Redis.

---

# CP2 — Docker production-ready

Starter Dockerfile chạy được nhưng chưa an toàn cho production.

## 7.1. Dockerfile

Yêu cầu:

- dùng multi-stage build với stage `builder` và `runtime`;
- dùng base image `python:3.11-slim` hoặc image gọn tương đương;
- copy `requirements.txt` và cài dependency trước khi copy source để tận dụng cache;
- tạo user thường và chuyển sang bằng `USER`;
- thêm `HEALTHCHECK` gọi `/health`;
- bind Uvicorn vào `0.0.0.0`;
- đọc cổng từ `${PORT:-8000}`;
- image cuối nhỏ hơn 500 MB.

## 7.2. `.dockerignore`

Không để các file/thư mục sau lọt vào build context:

- `.env`;
- `.git`;
- `.venv`;
- `__pycache__`;
- file thừa.

Không ignore nhầm:

- `app/`;
- `utils/`;
- `requirements.txt`.

## 7.3. `docker-compose.yml`

Thêm service `agent`:

- build từ Dockerfile hiện tại;
- map cổng `8000:8000`;
- đọc `AGENT_API_KEY` bằng nội suy biến môi trường, không hard-code;
- dùng `REDIS_URL=redis://redis:6379/0`;
- phụ thuộc service `redis`;
- có healthcheck.

Tên service `redis` chính là hostname bên trong mạng Compose.

> Không dùng `localhost` để agent kết nối Redis trong container.

Build và chạy thật:

```bash
docker build -t day12-agent:prod .
docker images day12-agent:prod
docker compose up -d
docker compose ps
curl http://localhost:8000/health
docker compose logs agent
```

Checkpoint:

```bash
pytest tests/test_cp2.py -v
```

Nếu cần kiểm tra nhanh mà chưa build image:

```bash
pytest tests/test_cp2.py -v -m "not docker"
```

> Test Docker bị skip không có nghĩa là phần đó đã đạt. Lab Coach có thể yêu cầu evidence build/chạy thực tế.

---

# CP3 — API Security và Cost Guard

Public URL sẽ bị người lạ hoặc bot gọi nếu không có lớp bảo vệ.

Bạn cần triển khai ba lớp:

| Lớp | Câu hỏi | HTTP status |
|---|---|---:|
| Authentication | Client có khóa hợp lệ không? | 401 |
| Rate limiting | User gọi có quá nhanh không? | 429 |
| Cost guard | User đã vượt ngân sách tháng chưa? | 402 |

## 8.1. Authentication

Trong `app/auth.py`:

- đọc `X-API-Key` và so sánh với `AGENT_API_KEY`;
- dùng `secrets.compare_digest`, không dùng `==`;
- sai hoặc thiếu key trả `401`;
- key hợp lệ trả về `X-User-Id`, hoặc `anonymous` nếu thiếu user ID.

## 8.2. Sliding-window rate limit

Trong `app/rate_limiter.py`, dùng Redis Sorted Set:

1. xóa request cũ hơn 60 giây;
2. đếm request còn trong cửa sổ;
3. nếu count `>= limit`, trả `429` và `Retry-After`;
4. nếu còn quota, thêm một member duy nhất rồi đặt TTL.

Phải kiểm tra trước và ghi nhận sau.

Member cần chứa timestamp cùng UUID để hai request đồng thời không ghi đè nhau.

## 8.3. Cost guard

Trong `app/cost_guard.py`:

- `spent()` trả `0.0` nếu key chưa tồn tại;
- `check()` trả `402` nếu chi phí hiện tại cộng chi phí ước tính vượt ngân sách;
- `record()` cộng dồn bằng `incrbyfloat` và đặt TTL;
- key phải tách theo user và tháng UTC.

## 8.4. Ghép `/ask` đúng thứ tự

```text
verify_api_key
→ limiter.check
→ guard.check
→ store.get_history
→ ask_llm
→ store.append user/assistant
→ guard.record
→ log_event
→ response
```

Authentication, rate limit và budget check phải chạy trước khi gọi LLM.

Kiểm tra:

```bash
# Thiếu key: mong đợi 401
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# Có key: mong đợi 200
curl -i -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv01" \
  -d '{"question":"Docker là gì?"}'
```

Checkpoint:

```bash
pytest tests/test_cp3.py -v
```

---

# CP4 — Scaling và Reliability

## 9.1. Đưa state ra khỏi process

Trong `app/store.py`, lưu history vào Redis thay vì dict Python:

- dùng Redis List theo từng user;
- serialize message thành JSON;
- dùng `ltrim` để chỉ giữ 20 message gần nhất;
- đặt TTL 7 ngày;
- `get_history()` trả list theo thứ tự cũ đến mới;
- `ping()` bắt mọi exception và trả `False` khi Redis không sẵn sàng.

State trong Redis cho phép nhiều container cùng thấy một lịch sử hội thoại và không làm mất dữ liệu tạm thời khi một instance restart.

## 9.2. Phân biệt liveness và readiness

| Endpoint | Câu hỏi | Dependency | Khi trả 503 |
|---|---|---|---|
| `/health` | Process còn sống không? | Không kiểm tra Redis | Orchestrator có thể restart container |
| `/ready` | Có nhận traffic được không? | Kiểm tra Redis | Load balancer ngừng gửi request |

`/ready` phải:

- trả `503` khi Redis lỗi hoặc service đang shutdown;
- trả `200` khi Redis sẵn sàng.

## 9.3. Graceful shutdown

Trong `app/lifecycle.py`:

- đăng ký `SIGTERM` và `SIGINT`;
- lưu handler cũ trước khi thay thế;
- khi nhận signal, đặt `shutting_down=True`;
- gọi lại handler cũ nếu callable;
- không thực hiện network hoặc tác vụ nặng trong signal handler.

Nếu không gọi lại handler của Uvicorn, service có thể chỉ đổi trạng thái sang “đang tắt” nhưng không thoát và cuối cùng bị `SIGKILL`.

Checkpoint:

```bash
pytest tests/test_cp4.py -v
```

Thử scale nếu Docker đã sẵn sàng:

```bash
docker compose up -d --scale agent=3
docker compose ps
```

Nginx/load balancing là phần mở rộng để học, không phải bonus riêng.

---

# CP5 — Deploy lên cloud

Chọn một trong hai nền tảng:

| Platform | Cách triển khai | Redis |
|---|---|---|
| Railway | Deploy từ Dockerfile/CLI, tạo Redis add-on | Có |
| Render | Tạo Blueprint từ `render.yaml` | Render Key Value |

## 10.1. Environment trên cloud

Đặt các biến trong dashboard hoặc secret store, không commit giá trị:

- `AGENT_API_KEY`;
- `REDIS_URL`;
- `RATE_LIMIT_PER_MINUTE=10`;
- `MONTHLY_BUDGET_USD=10.0`;
- `LOG_LEVEL=INFO`.

Không tự ghi đè `$PORT` nếu platform đã cấp.

Service phải bind `0.0.0.0` và đọc port do platform cung cấp.

## 10.2. Railway

Bạn có thể deploy qua dashboard hoặc Railway CLI:

```bash
railway login
railway init
railway add --database redis
railway up
railway domain
railway logs
```

Kiểm tra:

- `REDIS_URL` đã được gắn vào service agent;
- `AGENT_API_KEY` nằm trong Variables;
- secret không nằm trong repo.

## 10.3. Render

1. Push repository cá nhân lên GitHub.
2. Trên Render chọn **New → Blueprint**.
3. Chọn repository cá nhân; Render đọc `render.yaml`.
4. Điền `AGENT_API_KEY` khi được hỏi.
5. Chờ web service và Redis hoàn tất build/deploy.

## 10.4. Kiểm tra URL thật

```bash
URL=https://<domain-cua-ban>

curl -i $URL/health
curl -i $URL/ready
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

Kết quả mong đợi:

- `/health` trả `200` và `status: ok`;
- `/ready` trả `200` và `redis: true`;
- `/ask` không có API key trả `401`.

Điền đầy đủ `DEPLOYMENT.md`:

- họ tên, MSSV và repository;
- public URL và platform;
- tên các environment variables đã set, không ghi giá trị secret;
- output các lệnh kiểm tra;
- `screenshots/dashboard.png`;
- `screenshots/health.png`.

Nếu muốn test `/ask` có xác thực, đặt `DEPLOY_API_KEY` trong `.env` cục bộ.

> Đây là API key của chính service đã deploy, không phải token Railway hoặc Render.

Checkpoint:

```bash
pytest tests/test_cp5.py -v
```

## Phương án local fallback

Nếu không thể đăng ký cloud hoặc mạng bị chặn:

1. đặt `LOCAL_FALLBACK=true` trong `.env`;
2. chạy `docker compose up -d`;
3. kiểm tra `docker compose ps`, `/health`, `/ready` và auth;
4. lưu ảnh vào `screenshots/`;
5. ghi lý do trong `DEPLOYMENT.md`;
6. chạy lại:

```bash
pytest tests/test_cp5.py -v
```

> Phương án fallback làm CP5 bị giới hạn tối đa **9/15 điểm**.

---

# Exercises và tự chấm

Hoàn thiện đủ 10 câu trong `exercises.md` bằng lời của chính bạn và dựa trên quan sát khi chạy bài:

1. fail fast khi thiếu secret;
2. structured log;
3. kích thước image;
4. Docker layer cache;
5. container non-root;
6. sliding window;
7. rate limit và cost guard;
8. liveness và readiness;
9. stateless scaling;
10. lỗi thực tế khi deploy.

Không để lại dòng placeholder:

```text
> *Câu trả lời của bạn*
```

Chạy toàn bộ test và chấm điểm:

```bash
pytest tests/ -v
python grade.py
```

Nếu chưa làm bonus:

```bash
python grade.py --no-bonus
```

Thang điểm bắt buộc:

| Nội dung | Điểm |
|---|---:|
| CP1 — Config, Health & Logging | 15 |
| CP2 — Docker | 15 |
| CP3 — API Security | 20 |
| CP4 — Scaling & Reliability | 20 |
| CP5 — Cloud Deployment | 15 |
| `exercises.md` | 15 |
| **Tổng** | **100** |

Điểm checkpoint được tính theo tỷ lệ test pass.

Test bị skip do thiếu Docker hoặc điều kiện môi trường không tự động được xem là đạt.

---

# Bonus — CI/CD với GitHub Actions

Chỉ làm bonus sau khi CP1–CP5 đã ổn.

Tự tạo workflow trong `.github/workflows/` để:

- chạy khi push hoặc pull request;
- cài dependency;
- chạy tests;
- build Docker image;
- chỉ deploy nhánh chính sau khi tests xanh;
- hiển thị badge `passing` trong README.

Kiểm tra:

```bash
pytest tests/test_bonus_cicd.py -v
```

Bonus tối đa 10 điểm, nhưng `grade.py` vẫn giới hạn tổng cuối ở 100.

> Nginx/load balancing không thuộc bonus này.

---

# Kiểm tra và nộp bài

Repository phải ở chế độ **public** để Lab Coach truy cập được trong thời gian chấm.

Chạy kiểm tra cuối:

```bash
pytest tests/ -v
python grade.py
git status --short
git log --oneline --decorate -5
git ls-files | grep -E '(^|/)\.env$|\.(pem|key)$'
```

Checklist:

- [ ] Repository đúng tên `K4-L3B-DAY12-HoVaTen-MSSV-CloudServicesAndDeployment`.
- [ ] Không còn `NotImplementedError` trong phần bắt buộc của `app/`.
- [ ] CP1–CP5 đã được chạy và bạn biết test nào còn fail/skip cùng nguyên nhân.
- [ ] Dockerfile, `.dockerignore` và `docker-compose.yml` đã hoàn thiện.
- [ ] `exercises.md` đã trả lời đủ 10 câu bằng lời của bạn.
- [ ] `DEPLOYMENT.md` đã điền URL/platform/output và không chứa secret.
- [ ] `screenshots/dashboard.png` và `screenshots/health.png` mở được.
- [ ] Không commit `.env`, API key, token, private key hoặc credential.
- [ ] Có lịch sử commit theo từng checkpoint.
- [ ] Repository public và link truy cập được khi đăng xuất GitHub.

Nộp **URL repository GitHub cá nhân** trên Codelab trước deadline do Lab Coach công bố.

> Lab Coach có thể hỏi trực tiếp bất kỳ phần code hoặc nội dung phản ánh nào. Không giải thích được phần đã nộp có thể bị hủy điểm phần tương ứng; sao chép bài hoặc evidence có thể bị 0 điểm.
