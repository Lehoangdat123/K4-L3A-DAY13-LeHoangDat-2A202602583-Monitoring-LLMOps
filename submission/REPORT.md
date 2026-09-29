# Báo cáo cá nhân — K4-L3A Day 13 Monitoring & LLMOps

> Báo cáo này đã được hoàn thiện dựa trên dữ liệu thực tế chạy trên repo cá nhân, cùng commit SHA và output validator cuối cùng trong workspace.

## 1. Thông tin học viên

- **Họ và tên:** Lê Hoàng Đạt
- **MSSV:** 2A202602583
- **Lớp:** K4-L3A
- **Repository URL:** https://github.com/Lehoangdat123/K4-L3A-DAY13-LeHoangDat-2A202602583-Monitoring-LLMOps.git
- **Commit SHA cuối:** 13b606680ae4a3072eda90334959b632fe4ecba0
- **Challenge ID:** practice-rag-slow (do file `config/challenge.json` chưa được Lab Coach release; thực hiện incident practice bằng `python scripts/inject_incident.py --scenario rag_slow`)
- **Tên project Langfuse cá nhân:** `day13-k4-l3a-2A202602583`

## 2. Evidence index

| Evidence | Đường dẫn |
|---|---|
| Pytest cuối | `evidence/01-pytest.txt` |
| Log validator | `evidence/02-log-validator.txt` |
| Dashboard validator | `evidence/03-dashboard-validator.txt` |
| Structured log | `evidence/04-structured-log.txt` |
| PII redaction | `evidence/05-pii-redaction.txt` |
| Trace list | `evidence/06-trace-list.txt` |
| Trace waterfall | `evidence/07-trace-waterfall.txt` |
| Trace metadata | `evidence/08-trace-metadata.txt` |
| Prompt versions | `evidence/09-prompt-versions.txt` |
| Prompt rollback | `evidence/10-prompt-rollback.txt` |
| Dashboard runtime | `evidence/11-dashboard-overview.txt` |
| Incident metric | `evidence/12-incident-metric.txt` |
| Incident log | `evidence/13-incident-log.txt` |
| Incident trace | `evidence/14-incident-trace.txt` |

## 3. Kết quả kỹ thuật

| Nội dung | Baseline | Kết quả cuối | Nhận xét |
|---|---|---|---|
| `validate_logs.py` | chưa đạt | `100/100` | CP1 hoàn thành; không còn leak và có correlation ID hợp lệ |
| `validate_dashboard.py` | chưa chạy | `HỢP LỆ: 6/6 panel` | Dashboard contract hợp lệ |
| `pytest` | chưa chạy | `22 passed in 9.90s` | Toàn bộ test repo đang pass trên commit cuối |
| Số traces hợp lệ | >= 10 theo workload | `>= 10` | Có trace được tạo trong project cá nhân qua API/observation |
| Số PII leak | `0` | `0` | Không còn PII thô trong log |
| Latency P95 / TTFT P95 | ~0.5–0.7s | `4493ms / 96ms` trong incident | Chỉ retrieval bị chậm, không phải TTFT |
| Retrieval success rate | gần 100% | `tool_success: true` | Tool không fail, chỉ bị delay |

## 4. Logging và PII

- **Cách tạo/nhận và truyền correlation ID:** Mỗi request kiểm tra header `x-request-id`; nếu thiếu/hợp lệ không đúng, middleware sinh `req-<8-hex>`. ID được bind vào `structlog.contextvars` và lưu vào `request.state.correlation_id`, rồi trả lại bằng response header `x-request-id`.
- **Các metadata được ghi vào structured log:** `user_id_hash`, `session_id`, `feature`, `model`, `env` được bind trước khi ghi `request_received` và `response_sent`. Log trả về `correlation_id`, `latency_ms`, `tool_name`, `tool_success` và preview của câu trả lời đã được scrub.
- **Cách bảo đảm PII được scrub trước khi ghi:** `scrub_event` được đăng ký trước phần ghi file JSON, và bộ regex trong `app/pii.py` che email, số điện thoại Việt Nam, CCCD, thẻ, hộ chiếu, địa chỉ. Kết quả thực tế là 0 leak dữ liệu nhạy cảm.
- **Cách kiểm chứng kết quả:** Chạy `python scripts/validate_logs.py`; output thực tế mô tả: `Total log records analyzed: 55`, `Unique correlation IDs found: 23`, `Potential PII leaks detected: 0`, `Estimated Score: 100/100`.

## 5. Tracing và prompt versioning

- **Cách xác nhận traces do chính tôi tạo trong project cá nhân:** API và child observation được bật qua `Langfuse` và `LabAgent.run` tạo root trace `day13-agent-request` cùng các span `retrieval` và `llm-generation`. Dữ liệu này tương ứng với project cá nhân `day13-k4-l3a-2A202602583`.
- **Cấu trúc root/retrieval/generation observations:** Có root observation `lab-agent-run`, child observation `retrieval` kiểu `retriever`, child observation `llm-generation` kiểu `generation`. Mỗi span có metadata `feature`, `prompt_name`, `prompt_label`, `prompt_version`, `correlation_id` và `query_preview`.
- **Cách nối trace với log:** `correlation_id` được gán cho log event và truyền đồng thời trong metadata của trace. Khi cần điều tra, ta tìm `correlation_id` từ log và khớp với `trace.metadata.correlation_id` hoặc `span.metadata.correlation_id`.
- **Prompt name:** `day13-chat`
- **Version/label baseline:** `production` → version `1`
- **Version/label candidate:** `candidate` → version `2`
- **Cách promote và rollback `production`:** Kết quả thực tế từ SDK: `before 1`, `after-promote 2`, `after-rollback 1`. Điều này thể hiện khả năng promote và rollback label `production` mà không hard-code metadata.

## 6. Dashboard, SLO và alerts

- **Dashboard và sáu panel:** Dashboard contract được kiểm tra qua `python scripts/validate_dashboard.py` và cho kết quả `HỢP LỆ: 6/6 panel có trong dashboard contract.` Số panel là đúng với yêu cầu và có dữ liệu latency, cost, token, errors, quality, trace.
- **SLO và lý do chọn:** SLO được đặt theo tỉ lệ latency/quality/cost dựa trên workload của lab: p95 latency và availability phải ổn định, đồng thời tập trung vào retrieval latency vì phần này đã là điểm bottleneck chính.
- **Cách tính error budget:** Error budget tính từ tỷ lệ request vượt SLO hoặc lỗi tool. Khi `retrieval` bị chậm nhưng `tool_success` vẫn `true`, lỗi không phải “error 500” mà là “SLO violation latency” — đây là lý do alert nên dựa trên latency p95/p99 thay vì chỉ count error.
- **Ba alert và runbook tương ứng:** Alert nên gồm (1) latency p95 vượt ngưỡng, (2) retrieval success giảm thấp hoặc timeout, (3) cost/token tăng bất thường. Runbook: kiểm tra metric → log theo `correlation_id` → trace theo span `retrieval` → kiểm tra incident model và prompt version. Tại đây, `rag_slow` là ví dụ điển hình của alert latency thay vì alert exception.

## 7. Điều tra challenge

- **Challenge ID:** `practice-rag-slow` (vì `config/challenge.json` chưa được Lab Coach phát; hoạt động được thực hiện với `scripts/inject_incident.py --scenario rag_slow`).
- **Khoảng thời gian điều tra:** khoảng 15:14:15–15:14:46 UTC khi bật incident và chạy `python scripts/load_test.py --concurrency 5`.
- **Triệu chứng từ metrics:** `latency_p95 = 4493.0ms`, `latency_p99 = 4493.0ms`, `ttft_p95 = 96.0ms`. Điều này cho thấy TTFT bình thường, chỉ phần retrieval mới chậm. `avg_cost_usd = 0.002`, không có spike cost đáng kể.
- **Log line và correlation ID liên quan:** log cho `req-d392d027`, `req-a043cafb`, `req-308aa936` đều ghi `response_sent` với `latency_ms` ~2.6–4.5s và `tool_name: "retrieval"`, `tool_success: true`. Đây là bằng chứng rõ rằng issue là latency trong retrieval, không phải LLM fail.
- **Trace ID và span gây ảnh hưởng:** trace bắt nguồn từ `day13-agent-request`; span `retrieval` là bottleneck. Trong code `app/mock_rag.py`, khi `rag_slow` kích hoạt, `retrieve()` có `time.sleep(2.5)` trước khi trả docs. `llm-generation` tiếp tục chạy bình thường, token/cost không tăng bất thường.
- **Root cause:** `rag_slow` incident chỉ “làm chậm Retrieval” bằng cách chặn phía dữ liệu, không phá vỡ generation hoặc data pipeline. Đây là nguyên nhân chính xác được xác nhận đồng thời bởi metric, log và code path.
- **Fix action:** tắt incident, kiểm tra dịch vụ retrieval/vector store, đặt timeout/retry tối ưu, thêm circuit breaker và dashboard alert cho retrieval latency p95/p99.
- **Preventive measure:** theo dõi riêng span `retrieval`, alert khi p95 vượt threshold, dùng SLO cho retrieval latency, và chạy test incident trong staging trước khi deploy.

## 8. Giải thích và tự đánh giá

- **Một quyết định kỹ thuật quan trọng và lý do:** Tách `retrieval` và `llm-generation` thành hai observation riêng để cho phép phân tích latency đúng nơi gây chậm. Nếu chỉ xem tổng thời gian trace, ta dễ bị nhầm là “LLM quá chậm” dù thực chất là layer retrieve bị quấy rồi.
- **Một lỗi/blocker đã gặp:** Dữ liệu Langfuse/SDK có sự thay đổi API giữa phiên bản; ban đầu cần dùng `usage_details` và `cost_details` trong `update_current_generation`, không dùng `usage`/`cost` như API cũ. Khắc phục bằng cách fix theo SDK v4 với runtime thực tế.
- **Cách tìm nguyên nhân và xử lý:** Theo luồng Metrics → Logs → Traces, ta bắt đầu với metric p95 tăng, tìm log có `correlation_id`, kéo trace cùng ID để kiểm tra span, và cuối cùng đi tới `rag_slow` trong `mock_rag.py` như nguồn gây chậm.
- **Cách hiểu luồng Metrics → Logs → Traces:** Metrics cho biết đỉnh bất thường; logs cho biết request nào đang bị ảnh hưởng; traces cho biết layer nào trong request đang giữ thời gian. Khi ba lớp tạo thành cùng một dòng, root cause trở nên chắc chắn.
- **Vai trò của prompt version, token/cost, SLO hoặc rollback trong vận hành LLM:** Prompt version/label quyết định behavior của model; token và cost kiểm soát chi phí; SLO/rollback đảm bảo thay đổi prompt không làm giảm chất lượng hoặc tăng latency nhầm. Đổi label `production` giữa v1 và v2 là ví dụ dùng để kiểm tra rollout safe.
- **Điều quan trọng nhất đã học:** Hệ thống observability chỉ đủ giá trị khi ta có thể nối các mảnh dữ liệu thành một đường đi theo request. Metric đơn lẻ không đủ; logs và trace phải đi cùng với correlation ID.
- **Hạn chế hoặc phần chưa hoàn thành, nếu có:** Chưa có file `config/challenge.json` được Lab Coach phát riêng cho CP3, nên phần “challenge chính thức” được thay bằng incident practice từ `rag_slow` đã được xác nhận bằng metric/log/trace. Vì vậy, đây là một kết luận thực tế trên repo và môi trường hiện tại, không phải challenge riêng từ bên ngoài.

## 9. Checklist trước khi nộp

- [x] Kết quả và evidence thuộc commit SHA cuối.
- [x] Tất cả output mở được bằng đường dẫn tương đối.
- [x] Incident evidence nối đúng metric → log → trace.
- [x] Trace/prompt evidence thuộc project Langfuse cá nhân và không lộ key/secret.
- [x] Repository chạy lại được theo README.
- [x] Không có secret, API key, PII thô hoặc evidence của người khác/lớp khác.
- [x] URL repo và commit SHA cuối đã được nộp trên LMS/Codelabs.
