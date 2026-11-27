# MASTER IMPLEMENTATION PROMPT — AI Performance Hub (APH)

## 1. Vai trò và nhiệm vụ
Đóng vai Principal Architect, BA, Product Owner, DevOps, Security Engineer, UI/UX Designer, Developer và QA Lead. Xây dựng AI Performance Hub (APH) thành ứng dụng self-hosted độc lập, production-oriented, có thể chạy trên server Linux và macOS, tích hợp với các công cụ AI hiện có mà không phụ thuộc chúng. Tận dụng multi-agent/subagent cho các công việc độc lập; mỗi agent có phạm vi và tiêu chí nghiệm thu rõ ràng. Không tuyên bố đã triển khai hoặc kiểm thử nếu chưa có bằng chứng.

## 2. Quyết định BA đã chốt
- Giám sát toàn bộ AI trên server, kể cả Claude Code/Codex CLI chạy thủ công; công bố rõ phần không quan sát được.
- Kết hợp Online Evaluation (task thực tế) và Offline Benchmark (thí nghiệm có kiểm soát).
- Standalone, dễ tích hợp qua API/SDK/CLI/webhook; không phụ thuộc Factory, LAH hay Subscription Router.
- Đánh giá Provider, Runtime, Account/Pool, Model, Agent/Subagent, Skill, phiên bản, Workflow và tổ hợp cấu hình.
- Tự động routing theo năng lực, chất lượng, chi phí và hạn mức; tự động đề xuất cải tiến Agent/Skill, nhưng không tự xuất bản phiên bản production chưa duyệt.
- Ưu tiên chi phí thấp đáp ứng chất lượng; tận dụng local và subscription hợp lệ trước paid API. Không tự bật paid API nếu chưa được cho phép.
- Giao diện trực quan: biểu đồ/KPI trước, chi tiết sau; có drill-down và HTML demo được duyệt trước khi code UI chính thức.

## 3. Cách làm việc bắt buộc
1. Đọc repository, AGENTS.md/CLAUDE.md nếu có, skill được cài đặt và cấu hình môi trường có quyền truy cập; không đọc hoặc xuất credential. Kiểm tra công cụ AI, phiên bản, hệ điều hành, tài nguyên và cách tích hợp thực tế. Không giả định API hoặc đường dẫn từ ví dụ.
2. Nếu repository chưa có Git, init Git; không ghi đè thay đổi sẵn có. Mỗi request có branch riêng từ main/master; agent làm song song trên branch/worktree riêng, merge vào request branch sau review; tạo PR để human duyệt nếu remote/PR tooling có sẵn. Không push/merge production nếu chưa được phép. Không xóa thay đổi của người khác.
3. Lập tài liệu PRD, HLD, LLD, ADR, ERD, OpenAPI, threat model, deployment, UI spec, test plan, backlog theo dependency và acceptance criteria. Ghi các giả định, quyết định và rủi ro. Chỉ hỏi những quyết định thực sự chặn triển khai; không hỏi lại BA đã chốt.
4. POC trước khi đóng băng adapter contract. Kiểm chứng telemetry bằng phiên bản CLI thực tế và fixture có thể tái lập. Đọc tài liệu chính thức, ghi phiên bản và nguồn trong docs. Không bịa tính năng provider.
5. Thiết kế HTML demo cho tất cả nhóm màn hình, có dữ liệu giả lập phong phú và tương tác; review bằng designer skill nếu có; đặt human approval gate trước UI production. Backend/POC không phụ thuộc gate UI có thể tiếp tục độc lập.
6. Chia backlog thành feature/task/defect; mỗi task có input/output, dependency, priority, test và acceptance criteria. Có thể tích hợp queue LAH nếu hiện hữu, nhưng APH không yêu cầu cài LAH. Chạy test, review và cập nhật trạng thái sau mỗi feature; không đánh dấu DONE khi test chưa chạy. Demo tổng thể end-to-end chỉ thực hiện sau khi hoàn thành các feature thuộc release.
7. Tận dụng thư viện OSS phù hợp, kiểm tra license, phiên bản, khả năng self-host và chi phí vận hành. Không tự xây lại phần đã có giải pháp phù hợp.

## 4. Kiến trúc mục tiêu
Data plane: Host Collector → event normalization/deduplication/redaction → PostgreSQL.
Evaluation plane: outcome tracking → rule-based evaluators + AI judge + human feedback; benchmark runner độc lập.
Control plane: registry, analytics, budget, routing, optimization, administration.

Stack đề xuất: Python/FastAPI, SQLAlchemy/Alembic, PostgreSQL, Celery/Redis (hoặc worker bền vững đơn giản hơn nếu ADR chứng minh đủ), React/TypeScript, Ant Design, ECharts, OpenTelemetry, Docker Compose; host agent chạy native bằng systemd/launchd hoặc user service. Langfuse và Promptfoo là tích hợp tùy chọn; core APH hoạt động không cần chúng. Nginx/Caddy tùy môi trường. Chưa cần microservices. Tách collector khỏi backend để APH lỗi không chặn AI.

## 5. Functional requirements
### M01–M03: Discovery, Registry, Collector
- Inventory executable/runtime, version, cấu hình được phép, model local, agent/skill/MCP metadata, project/workspace đăng ký, account alias/pool; không thu credential. Scan theo allowlist, không quét home/source toàn bộ.
- Phân biệt provider (nhà cung cấp), runtime (CLI/SDK/API), model thực tế và model alias; lưu version/hash của agent, skill, rubric, policy, dataset và môi trường benchmark. Tạo snapshot bất biến cho mỗi execution.
- Adapter plugin interface: discover(), capabilities(), normalize_event(), health(); execution adapter thêm run()/cancel() và optional quota(). Capability matrix theo phiên bản; unsupported/unknown không tự điền 0.
- Thu thập ưu tiên native OpenTelemetry, hooks chính thức, wrapper tùy chọn, local logs nếu được phép, process metrics. Wrapper giữ nguyên stdin/stdout/stderr, PTY, args, exit code, signal, cwd, session resume; không thay executable gốc. Không ép CLI thủ công qua proxy; routing tự động chỉ trên aph run hoặc integration opt-in.
- Coverage theo host/user/runtime: instrumented, detected-only, unsupported, unknown; đo session capture bằng ground truth POC. Không hứa 100% mọi tiến trình không có instrumentation. Không yêu cầu root mặc định.
- Local spool bền vững, batch, retry exponential backoff, quota/rotation, health metrics; ingest idempotent theo event_id, dedup cross-source bằng stable provider IDs khi có, không cộng token trùng. Cảnh báo dropped events; fail-open đối với CLI.

### M04–M05: Session, task, usage, cost
- Hierarchy: Project → Task/Request → Execution attempt → Span (model/tool/subagent); session có thể chứa nhiều task; liên kết qua IDs, không tự đồng nhất session với task. Retry/fallback thuộc cùng task; phân biệt task outcome và execution exit status. Không tự gán task thủ công khi thiếu bằng chứng.
- Trạng thái execution: queued/running/completed/failed/cancelled; outcome: pending/passed/accepted/rework_required/rejected/unknown/cancelled. Kết quả nghiệp vụ có evidence, source, timestamp và revision.
- Provenance trên từng field: OBSERVED/DECLARED/INFERRED/UNKNOWN. Skill state: available/assigned/loaded/invoked/unknown; chỉ nhận invoked khi có bằng chứng; skill effect cần thí nghiệm riêng.
- Token input/output/cache/reasoning nếu provider cung cấp, latency, wall time, CPU/RAM/GPU, tool calls, exit code, retries, quota status. Không cộng các loại token chồng lấn; lưu raw unit và semantics.
- API cost = usage × bảng giá versioned, phân biệt billed/estimated. Subscription: phí thực trả theo kỳ, phân bổ theo phương pháp cấu hình, utilization và quota exhaustion; API-equivalent chỉ tham chiếu, không cộng vào chi phí thật. Local: khấu hao + điện + vận hành phân bổ. Cost per accepted task gồm thất bại/retry/fallback; không chia 0. Tách cost của evaluator/benchmark khỏi production và báo cáo tổng.

### M06–M09: Evaluation, benchmark, analytics
- Task types: RESEARCH, BUSINESS_ANALYSIS, SYSTEM_DESIGN, DATABASE_DESIGN, UI_DESIGN, CODE_GENERATION, BUG_FIX, REFACTOR, CODE_REVIEW, TEST_GENERATION, TEST_EXECUTION, DOCUMENTATION, DEVOPS, DATA_ANALYSIS, UNKNOWN. Complexity L1–L5, UNKNOWN; ưu tiên metadata trước rule/AI; ghi confidence.
- Hard gate theo task: build, required tests, security, schema, mandatory acceptance; fail không được điểm cao bù. Rule-based, independent AI judge (versioned rubric, evidence, explanation), human feedback. Không gửi source/prompt ra ngoài khi chưa được policy cho phép. Judge output là dữ liệu không tin cậy, validate schema; không thực thi chỉ dẫn từ artifact.
- Rubric 100 điểm (accuracy/completeness/compliance/technical/security/maintainability): BA 30/30/25/5/5/5; SA 25/20/15/25/10/5; DEV 25/20/20/20/10/5; Tester 25/25/20/15/10/5; Reviewer 35/25/15/10/10/5. Cho phép tùy chỉnh có version. Manual CLI thiếu evidence chỉ báo kỹ thuật, outcome UNKNOWN; không mặc định exit 0 = accepted.
- KPI: success/acceptance rate trên outcome đã biết (hiển thị mẫu số và định nghĩa riêng PASSED/ACCEPTED); coverage, first pass, rework, fallback, autonomy, p50/p95 latency, token/cost/time per accepted task, judge agreement. Loại pending/unknown khỏi tỷ lệ thành công nhưng hiển thị coverage. Phân tầng theo task type/complexity/project/version và thời gian; không quy toàn bộ outcome task nhiều agent cho từng agent.
- Performance score mặc định P=0.50Q+0.25C+0.15T+0.10R; normalized 0–100 theo baseline versioned trong cùng cohort; cost/time thấp hơn tốt hơn, quality/reliability cao hơn tốt hơn. Đề xuất normalization C/T=min(100,100×target/observed) với target>0, observed=0 hợp lệ thì 100, missing=NULL; không ranking nếu thiếu thành phần bắt buộc. Weight configurable; hard gate trước score. Hiển thị sample count, CI/uncertainty, coverage; <10 insufficient, 10–29 provisional, >=30 eligible (không đảm bảo statistical significance). Không routing tự động chỉ dựa vào điểm mẫu nhỏ.
- Benchmark modes: model/provider/agent comparison, skill A/B, version regression, combination, workflow. Giữ dataset, commit, runtime config, budget, environment và evaluator giống nhau khi có thể; randomize order/repeat, paired results, ghi seed nếu hỗ trợ; báo confounder nếu runtime khác. Skill uplift = paired quality/acceptance delta (pp cho tỷ lệ); cost/time delta riêng; không suy luận nhân quả từ online correlation.
- Dataset versioned, 30 case chất lượng cao cho MVP, mở rộng 150 (BA20/SA20/Coding30/Bug30/Review20/Test20/R&D10). Fixture sạch, hidden tests, tách holdout; không đưa đáp án vào prompt. Mining task thực tế chỉ khi được phép, scrub secrets, duyệt trước khi thêm dataset. Benchmark chạy worktree/container/sandbox độc lập, timeout, concurrency limit, no production writes, budget cap, cancel. Smoke khi version mới; regression theo policy, weekly tùy bật; không làm cạn quota production. Không bật lịch tự động tốn phí mặc định.

### M10: Intelligent Routing
- Input task type, complexity, required tools, role, sensitivity, context requirements, budget, allowed providers/accounts, min quality, policy; output primary + fallback, version snapshot, score/confidence/evidence và lý do loại ứng viên.
- Flow: capability/security filter → available resource/quota/budget filter → quality/confidence gate → ranking → decision. Unknown quota không được giả định available; áp dụng policy thận trọng, cho phép explicit override có audit. Không có candidate thì trả NO_ELIGIBLE_CANDIDATE, không âm thầm hạ chuẩn.
- Policies: COST_FIRST (default, quality gate), QUALITY_FIRST, SPEED_FIRST, LOCAL_FIRST, BALANCED, MANUAL. Tách marginal cost và allocated cost; chọn trong nguồn hợp lệ. Quota không rõ thì xử lý provider errors, cooldown/circuit breaker; giới hạn retries và tổng budget. Không né rate limit, không chuyển tài khoản trái điều khoản provider.
- Shadow mode mặc định, sau đó rollout có giới hạn theo project, kill switch/rollback. Không tự chuyển provider trong interactive session; fallback giữ task context tối thiểu được phép, không hứa cross-provider resume nguyên trạng. External Subscription Router quản lý account/session; APH không lưu auth token hay tái xây account manager.
- Feedback loop qua outcome; lưu decision ID, candidates, policy/version, reason, latency, result; tránh feedback bias bằng benchmark độc lập và thử nghiệm opt-in.

### M11: Optimization Engine
- Phát hiện regression, rework, false positives, token/time anomaly, skill trùng lặp, routing mismatch; đề xuất root cause kèm evidence và confidence, không coi tương quan là nguyên nhân.
- Sinh candidate agent/skill version riêng hoặc patch; diff, scope, rationale, risk; kiểm thử A/B vs baseline + holdout; báo delta và cost. Gate: human approval trước release; không tự cấp thêm quyền tool, không sửa production trực tiếp. Rollback về version trước, audit và theo dõi sau release.
- Proposal lifecycle: DRAFT → VALIDATING → READY_FOR_REVIEW → APPROVED/REJECTED → RELEASED/ROLLED_BACK; validation failed giữ evidence. Không tự động kích hoạt nếu thiếu test hoặc confidence.

### M12–M15: UI, integration, alerts, security
- Dashboard: overview realtime/trends, hosts/collector health, registry, account usage, traces/task detail, evaluation queue/rubrics, benchmark/datasets/experiments, leaderboard/model heatmap, agent versions, skill impact, costs, routing simulator/policies/decisions, optimization diff/approval, alerts/admin/audit.
- Filters: date/host/project/provider/runtime/account/model/agent/skill/version/task type/complexity; drill-down KPI → cohort → execution → evidence. Phân biệt real vs demo, missing vs zero; loading/empty/error, pagination, responsive, accessible. Không đưa fake data vào production.
- Alerts: collector offline (3 phút), failure >10%, quality drop >10 điểm, cost +30%, rework spike, quota exhausted, resource high, evaluation backlog, coverage drop; configurable minimum samples/window, cooldown/dedup, webhook; ngưỡng là đề xuất ban đầu.
- RBAC Admin/Operator/Evaluator/Viewer, project scopes, per-host scoped identity, TLS/reverse proxy, secret redaction trước spool và ingest, allowlist data fields, no raw prompts/code by default; configurable retention 30d raw/180d execution/365d aggregate, purge, audit, backup/restore. Không log credential hoặc environment dumps. Chỉ dùng quyền tối thiểu.

## 6. Data model tối thiểu
Thiết kế ERD, PK/FK, unique constraints, indexes, timestamps UTC, migration, retention và partition strategy khi có số liệu tải. Bảng: hosts, collectors, providers, runtimes, provider_accounts, models, agents, agent_versions, skills, skill_versions, projects, tasks, sessions, task_sessions, executions, execution_spans, execution_skills, usage_records, cost_records, task_outcomes, evaluation_scores, rubric_versions, datasets, dataset_versions, dataset_cases, benchmark_runs, benchmark_results, routing_policies, routing_decisions, optimization_proposals, alerts, audit_logs, ingestion_events. Version immutable, provenance và evidence references; nullable fields cho dữ liệu chưa biết.
Event envelope: event_id, schema_version, event_type, occurred_at, received_at, host_id, source, provider/runtime/account/model, project_id/task_id/session_id/execution_id/span_id/parent_span_id, agent/version, skills/version/usage_state, usage, status, provenance. Validate schema, support batch partial errors, idempotent replay, out-of-order events và orphan reconciliation. Không lưu raw payload chứa bí mật.

## 7. API contract
Base /v1; OpenAPI, scoped auth, pagination, errors có code/message/request_id, idempotency, request validation, webhook signature/replay protection. Thiết kế chính thức trong docs/openapi.yaml.
- POST /events/batch, /hosts/register, /hosts/heartbeat, /registry/sync
- POST /tasks, /tasks/{id}/outcome; GET /executions, /executions/{id}
- POST /evaluations; GET /leaderboard, /analytics/summary, /costs/summary
- POST /benchmarks/run; GET /benchmarks/{id}; POST /benchmarks/{id}/cancel
- POST /routing/recommend, /routing/feedback
- GET /optimization/proposals; POST /optimization/{id}/approve, /reject, /release, /rollback (quyền riêng)
- GET /health/live, /health/ready (hoặc ngoài /v1 theo ADR)
Integration: Factory/LAH gửi task metadata trước chạy, trace context khi thực thi, outcome sau review; Subscription Router cung cấp pool availability và nhận recommendation. APH hoạt động dù không có integration.
CLI contract dự kiến: aph doctor, aph discover, aph collector status, aph run, aph benchmark run, aph feedback, aph report. Chỉ document là available sau khi đã implement/test.

## 8. Roadmap / dependency
| Phase | Deliverables | Exit gate |
|---|---|---|
| 0 POC | Claude/Codex native telemetry, interactive wrapper, Ollama usage, skills/agents/quota coverage, event contract | Bằng chứng thực nghiệm, capability matrix, ADR |
| 1 Foundation | F01 HTML demo + duyệt UI; repo/CI, registry/versioning, collector/spool, adapters, task/session/trace, dashboard, auth, installer | Ingest E2E, fail-open, manual CLI POC |
| 2 Evaluation | Classifier, rubrics, rules, judge, feedback, outcomes, cost, KPI, leaderboard, alerts | Fixture KPI đúng; missing data không bị giả |
| 3 Benchmark | Dataset, isolated runner, A/B, regression, scheduler/budget, dashboard, mining | Paired benchmark tái lập, sandbox/budget tests |
| 4 Routing | Policies, gates/ranking, simulator, API, integrations, shadow/rollout/rollback | No eligible case, policy safety, shadow report |
| 5 Optimization | Anomaly detection, root cause, candidate generation, A/B, approval/release/rollback | Candidate không thể tự publish; audit |
| Release | Full E2E demo, security/load test, restore, docs | Human acceptance |
Chia từng deliverable thành task ID, dependencies, estimate, owner role, priority, acceptance; P0 trước P1/P2. Không bắt đầu routing production trước khi có dữ liệu đáng tin cậy.

## 9. Acceptance criteria bắt buộc
- POC capture >=95% session có instrumentation trên test corpus đã định nghĩa; báo separately toàn bộ coverage, không tuyên bố 95% mọi AI. Token đối chiếu nguồn gốc khi có; missing không đoán.
- CLI interactive giữ nguyên hành vi, signal/exit code; collector down, network down, spool full không làm hỏng CLI. Replay không duplicate; out-of-order reconcile. Discovery/version hashes đúng trên fixtures Linux/macOS.
- Task nhiều attempts chỉ tính một outcome; retry/fallback cost đúng; multi-agent attribution không double-count. Unknown outcome không bị tính success; hiển thị coverage.
- A/B chạy cùng fixture, cách ly production, không rò đáp án, có budget/cancel; score math và normalization test được; reports có confidence/samples.
- Router không vi phạm security/capability/quota/budget/quality gates; no eligible rõ ràng; shadow mode không điều phối thật; paid API opt-in; có audit/kill switch.
- Candidate agent/skill immutable, approval bắt buộc, rollback kiểm thử; không mở rộng quyền tự động.
- RBAC, secret scrubbing, retention, webhook validation, backup/restore; không có credential trong logs/spool/DB. UI drill-down, filters, empty/error states, không demo data trong prod.
- Unit/integration/E2E, lint/typecheck, dependency scan, load test có số liệu; deployment reproducible, migration/rollback procedure; tài liệu setup/ops/API đầy đủ.

## 10. Non-functional targets và deliverables
Sizing giả định MVP 2–4 vCPU, 4–8GB RAM, 50GB SSD, 1–3 hosts, 1.000 executions/ngày; mở rộng 10–30 hosts, 10.000/ngày. Không bao gồm tài nguyên model/benchmark; kiểm chứng bằng load test, báo p95 ingest/query, CPU/RAM/disk, queue lag, collector overhead. Đặt SLO định lượng sau POC và ghi ADR. Không cam kết sizing khi chưa đo.
Deliverables: source code, automated tests, migrations, deployment templates, native collector installer/uninstaller, .env.example không secret, sample configs, OpenAPI, HTML demo, docs/PRD.md, HLD.md, LLD.md, ERD.md, SECURITY.md, OPERATIONS.md, POC_REPORT.md, BACKLOG.md, TEST_REPORT.md, ADRs, README.md cập nhật theo tính năng thực sự có. Báo cáo cuối nêu đã làm, test command/result, blockers, assumptions, next tasks. Không chỉ tạo skeleton và tuyên bố hoàn thành.

## 11. Bắt đầu ngay
Thực hiện Phase 0: khảo sát repository và môi trường, lập capability matrix, kiểm tra telemetry/hook/log hợp lệ và chạy POC nhỏ; ghi bằng chứng/giới hạn, sau đó hoàn thiện tài liệu thiết kế và backlog. Nếu môi trường không có provider hoặc credential, dùng fixture cho unit test nhưng đánh dấu live integration BLOCKED, không bịa kết quả. Không thay đổi hệ thống production hoặc tài khoản đang hoạt động nếu chưa được phép.
