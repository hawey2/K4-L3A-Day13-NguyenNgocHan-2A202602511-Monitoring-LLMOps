# Template Alert và Runbook

Mỗi alert phải dựa trên triệu chứng người dùng hoặc SLO, không dựa trực tiếp vào tên implementation nội bộ.

## Alert 1: High Latency (P95 > 3s)

- Tên: high_latency_p95
- Severity: critical
- Duration: 5m
- Kênh thông báo: Slack
- SLI/SLO liên quan: fast_successful_requests (latency_ms <= 3000 at P95)
- Điều kiện và thời gian duy trì: P95 latency vượt 3000ms trong 5 phút liên tục
- Ảnh hưởng tới người dùng: Trải nghiệm chat chậm, timeout client, giảm tỷ lệ hoàn thành request
- Ba bước kiểm tra đầu tiên:
  1. Xem dashboard panel Latency - xác định P95/P99 có spike không
  2. Lọc logs trong khoảng thời gian spike, tìm correlation_id có latency cao
  3. Traces trên Langfuse với correlation_id đó, so sánh span retrieve vs generate
- Mitigation tạm thời: Scale up API instances, tắt feature non-critical, kiểm tra incident rag_slow/cost_spike
- Owner: oncall-engineer

## Alert 2: High Error Rate (> 2%)

- Tên: high_error_rate
- Severity: warning
- Duration: 2m
- Kênh thông báo: Slack
- SLI/SLO liên quan: fast_successful_requests (error budget), guardrails.error_rate_pct_max
- Điều kiện và thời gian duy trì: Tỷ lệ request_failed / request_received vượt 2% trong 2 phút
- Ảnh hưởng tới người dùng: Request thất bại, người dùng không nhận được câu trả lời
- Ba bước kiểm tra đầu tiên:
  1. Xem dashboard panel Errors - xem error_type breakdown
  2. Lọc logs event=request_failed, nhóm theo error_type và correlation_id
  3. Traces các correlation_id failed, xem span nào error (retrieve hay generate)
- Mitigation tạm thời: Disable incident tool_fail nếu đang bật, rollback prompt version nếu lỗi sau deploy
- Owner: oncall-engineer

## Alert 3: Retrieval Failure Spike (> 10%)

- Tên: retrieval_failure_spike
- Severity: warning
- Duration: 3m
- Kênh thông báo: Slack
- SLI/SLO liên quan: guardrails.retrieval_success_rate_pct_min (90%)
- Điều kiện và thời gian duy trì: Tỷ lệ tool_success=false trong retrieval vượt 10% trong 3 phút
- Ảnh hưởng tới người dùng: RAG không trả về context, chất lượng câu trả lời giảm mạnh (quality_score xuống)
- Ba bước kiểm tra đầu tiên:
  1. Xem dashboard panel Errors - retrieval success rate
  2. Lọc logs tool_name=retrieval AND tool_success=false, lấy correlation_id
  3. Traces correlation_id đó, xem span retrieve có error/timeout không
- Mitigation tạm thời: Disable incident tool_fail, kiểm tra vector store health, fallback sang local docs
- Owner: oncall-engineer
