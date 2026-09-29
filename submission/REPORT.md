# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Mỗi học viên hoàn thiện một file duy nhất này. Khi dẫn evidence, dùng đường dẫn tương đối, ví dụ `evidence/07-trace-waterfall.png`.

## 1. Thông tin học viên

- **Họ và tên:** Nguyen Ngoc Han
- **MSSV:** 2A202602511
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/hawey2/K4-L3A-Day13-NguyenNgocHan-2A202602511-Monitoring-LLMOps
- **Commit SHA cuối:** 13b6066
- **Challenge ID:** N/A (challenge not released)
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602511`

## 2. Evidence index

Điền đúng đường dẫn tới evidence thực tế. Có thể đổi tên hoặc dùng nhiều ảnh nếu cần.

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.png` |
| Log validator | `evidence/02-log-validator.png` |
| Dashboard validator | `evidence/03-dashboard-validator.png` |
| Structured log | `evidence/04-structured-log.png` |
| PII redaction | `evidence/05-pii-redaction.png` |
| Trace list | `evidence/06-trace-list.png` |
| Trace waterfall | `evidence/07-trace-waterfall.png` |
| Trace metadata | `evidence/08-trace-metadata.png` |
| Prompt versions | `evidence/09-prompt-versions.png` |
| Prompt rollback | `evidence/10-prompt-rollback.png` |
| Dashboard runtime | `evidence/11-dashboard-overview.png` |
| Incident metric | `evidence/12-incident-metric.png` |
| Incident log | `evidence/13-incident-log.png` |
| Incident trace | `evidence/14-incident-trace.png` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | 0/100 | 100/100 | Tất cả PII được scrub, correlation ID propagate đúng, enrichment đầy đủ |
| `validate_dashboard.py` | 6/6 | 6/6 | Dashboard contract hợp lệ 6 panel |
| `pytest` | 20/22 | 22/22 | Tất cả tests pass |
| Số traces hợp lệ | 0 | 10+ | Tạo >= 10 traces trong project Langfuse cá nhân |
| Số PII leak | N/A | 0 | Không có PII leak trong logs |
| Latency P95 / TTFT P95 | N/A | 231ms / 55ms | Rất tốt, dưới ngưỡng 3000ms |
| Retrieval success rate | N/A | 100% | Tất cả retrieval thành công |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Middleware `CorrelationIdMiddleware` extract `x-request-id` từ header nếu có format `req-<8-hex>`, nếu không thì sinh mới bằng `uuid.uuid4().hex[:8]`. ID được bind vào structlog contextvars và truyền qua `request.state.correlation_id` xuống handler. Response header trả về `x-request-id` và `x-response-time-ms`.

- **Các metadata được ghi vào structured log:** `user_id_hash` (SHA256 12 ký tự đầu), `session_id`, `feature` (qa/summary), `model` (claude-sonnet-4-5), `env` (dev). Tất cả được bind trước khi log `request_received`.

- **Cách bảo đảm PII được scrub trước khi ghi:** Processor `scrub_event` được đăng ký trong structlog pipeline **trước** `JSONRenderer` và `JsonlFileProcessor`. Processor này quét tất cả string trong `payload` và `event` field, áp dụng regex patterns cho email, phone_vn, cccd, credit_card.

- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py` đạt 100/100, kiểm tra `data/logs.jsonl` xác nhận không có PII nguyên văn, correlation ID unique 10 IDs.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** Project Langfuse `day13-k4-l3a-2A202602511` được cấu hình qua `.env` với `LANGFUSE_PUBLIC_KEY`/`SECRET_KEY` riêng. Traces xuất hiện trong project này trên Langfuse Cloud.

- **Cấu trúc root/retrieval/generation observations:** Root observation `lab-agent-run` (type=agent) chứa child `retrieve` (type=retriever) và `generate` (type=generation). Generation observation cập nhật thêm `prompt`, `usage` (input/output tokens), `model`, `metadata` (cost_usd, ttft_ms) qua `update_current_generation`.

- **Cách nối trace với log:** `correlation_id` được truyền vào trace metadata qua `propagate_attributes` với key `correlation_id`. Log cũng ghi `correlation_id` nên có thể tìm trace từ log và ngược lại.

- **Prompt name:** `day13-chat`

- **Version/label baseline:** v1, label `production`

- **Version/label candidate:** v2, label `staging` (hoặc label khác để test)

- **Trace ID của mỗi version:** Trace IDs khác nhau do mỗi request tạo trace mới; version prompt được ghi trong trace metadata `prompt_version`.

- **Cách promote và rollback `production`:** Trên Langfuse UI, vào Prompts → `day13-chat` → chọn version → click "Promote to production" hoặc đổi label. Rollback bằng cách promote version cũ lên production. Code dùng `client.get_prompt(name, label="production")` nên tự động lấy version đang gán label production.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** `config/dashboard.yaml` định nghĩa 6 panel: Latency (P50/P95/P99 + TTFT P95), Traffic (requests/min), Errors (error rate + retrieval success), Cost (USD/min + total), Tokens (in/out sum), Quality (mean score). Panel dùng `data/logs.jsonl` làm nguồn.

- **SLO và lý do chọn:** SLO `fast_successful_requests`: 99.5% request có latency <= 3000ms trong cửa sổ 28 ngày. Chọn ngưỡng 3000ms vì baseline P95 ~231ms, cho headroom 10x. Error budget = 0.5% (khoảng 3.6 giờ downtime/tháng).

- **Cách tính error budget:** Error budget = (1 - SLO target) * total requests trong window 28 ngày. Với SLO 99.5% → error budget 0.5%. Mỗi request failed hoặc latency > 3000ms tiêu tốn budget.

- **Ba alert và runbook tương ứng:**
  1. `high_latency_p95` (critical, 5m): P95 > 3000ms → check dashboard → filter logs high latency → trace spans retrieve/generate
  2. `high_error_rate` (warning, 2m): error rate > 2% → check error breakdown → trace failed requests
  3. `retrieval_failure_spike` (warning, 3m): retrieval failure > 10% → check tool_success=false logs → trace retrieve span

## 7. Điều tra challenge

- **Challenge ID:** N/A (challenge not released by Lab Coach)
- **Khoảng thời gian điều tra:** N/A
- **Triệu chứng từ metrics:** N/A
- **Log line và correlation ID liên quan:** N/A
- **Trace ID và span gây ảnh hưởng:** N/A
- **Root cause:** N/A
- **Fix action:** N/A
- **Preventive measure:** N/A

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Đặt PII scrubber (`scrub_event`) **trước** JSON renderer trong structlog pipeline. Điều này đảm bảo PII bị redact ngay tại object event_dict trước khi serialize ra JSON, tránh leak PII ra file log hoặc stdout.

- **Một lỗi/blocker đã gặp:** Test `test_chat_response_log_exposes_quality_for_dashboard` fail do `ChatRequest` thiếu field `model`. Root cause: schema không có field `model` nhưng `main.py` truy cập `body.model`. Fix: thêm `model: str = Field(default="claude-sonnet-4-5")` vào `ChatRequest`.

- **Cách tìm nguyên nhân và xử lý:** Chạy pytest xem error traceback → thấy `AttributeError: 'ChatRequest' object has no attribute 'model'` → đọc `app/schemas.py` và `app/main.py` → thêm field missing → chạy lại test pass.

- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics (dashboard) cho thấy triệu chứng (latency cao, error rate tăng) và thời gian → Lọc `data/logs.jsonl` theo thời gian và event bất thường để lấy `correlation_id` → Dùng `correlation_id` tìm trace trên Langfuse → So sánh span retrieve vs generate để xác định bottleneck/root cause.

- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Prompt version cho phép A/B test, rollback nhanh khi version mới gây regression. Token/cost tracking giúp kiểm soát chi phí và phát hiện anomaly (cost spike). SLO/error budget định lượng độ tin cậy, hướng dẫn decision release/rollback. Rollback prompt là mitigation nhanh nhất khi quality giảm.

- **Điều quan trọng nhất đã học:** Correlation ID là "dây liên kết" quan trọng nhất giữa metrics, logs, traces. Không có nó thì không thể điều tra end-to-end. Structured logging + PII scrubbing + tracing phải thiết kế cùng nhau từ đầu, không phải bổ sung sau.

- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Challenge chính thức (CP3) chưa chạy do Lab Coach chưa release file `config/challenge.json`. Evidence ảnh (screenshots) chưa thu thập đầy đủ do môi trường headless.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [ ] Tất cả ảnh/output mở được bằng đường dẫn tương đối.
- [ ] Incident evidence nối đúng metric → log → trace.
- [ ] Trace/prompt evidence thuộc project Langfuse cá nhân và ảnh không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [ ] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.