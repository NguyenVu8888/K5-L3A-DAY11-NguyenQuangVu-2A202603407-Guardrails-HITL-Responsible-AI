# CẨM NANG KIẾN TRÚC & TÁI TẠO HỆ THỐNG TỪ CON SỐ 0
## Dự án: Controlled Agent Security & Guardrails (VinBank AI Assistant)

---

# PHẦN 1: BỨC TRANH TOÀN CẢNH & THIẾT KẾ KIẾN TRÚC TỔNG THỂ

## 1. Bản chất bài toán (Problem Statement)
Hệ thống là một trợ lý ngân hàng AI (VinBank Assistant) xử lý các truy vấn nhạy cảm (tài khoản, chuyển khoản, lãi suất).
Hệ thống phải hoạt động trong môi trường không tin cậy (Zero-Trust Data Source):
- Input người dùng có thể chứa Prompt Injection, Jailbreak, Unicode tàng hình, hoặc các truy vấn lạc đề/độc hại.
- Output sinh ra từ LLM có thể bị ảo giác (hallucination) hoặc vô tình làm lộ lọt dữ liệu định danh người dùng (PII: SĐT, Email, CCCD) hoặc bí mật hệ thống (Admin password, API key, Database connection string).
- Luồng Egress (dữ liệu gửi ra bên ngoài) phải bị kiểm soát chặt chẽ, không cho phép gửi dữ liệu nhạy cảm sang các endpoint ngoài whitelist.

Do đó, kiến trúc được thiết kế theo nguyên lý **Phòng thủ đa tầng (Defense-in-Depth)** và **Giám sát toàn diện (Observability)**, không bao giờ tin tưởng hoàn toàn vào sự tự kiềm chế của LLM.

---

## 2. Luồng chạy End-to-End Lifecycle (Khi hệ thống hoạt động)

```text
                             [NGƯỜI DÙNG / KHÁCH HÀNG]
                                         │
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │ 1. RATE LIMITER (src/assignment/rate_limiter.py) │
                 │    - Kiểm tra số lượng gọi / cửa sổ trượt       │
                 │    - Vượt ngưỡng: CHẶN NGAY (Fail-fast)          │
                 └───────────────────────┬─────────────────────────┘
                                         │ (ALLOW)
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │ 2. INPUT GUARDRAILS                             │
                 │    (src/guardrails/input_guardrails.py)         │
                 │    - detect_injection(): Regex + Unicode strip  │
                 │    - topic_filter(): Whitelist / Blacklist      │
                 │    - Dính mã độc / lạc đề: CHẶN NGAY            │
                 └───────────────────────┬─────────────────────────┘
                                         │ (ALLOW)
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │ 3. BLUE AGENT / LLM INFERENCE                   │
                 │    (src/agents/agent.py & openai_runtime.py)    │
                 │    - Model: liquid/lfm-3-1.7b:free (OpenRouter) │
                 │    - System Prompt bảo vệ bí mật nội bộ         │
                 └───────────────────────┬─────────────────────────┘
                                         │
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │ 4. OUTPUT GUARDRAILS                            │
                 │    (src/guardrails/output_guardrails.py)        │
                 │    - content_filter(): Quét PII (SĐT, Email,    │
                 │      CCCD) & Secrets (Admin pass, sk-..., DB)   │
                 │    - Thay thế vi phạm bằng [REDACTED]           │
                 └───────────────────────┬─────────────────────────┘
                                         │
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │ 5. EGRESS GATEWAY (src/assignment/pipeline.py)  │
                 │    - is_egress_allowed(): Whitelist domain      │
                 │    - Chặn payload nếu còn sót Secret/PII        │
                 └───────────────────────┬─────────────────────────┘
                                         │
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │ 6. OBSERVABILITY TẦNG NGANG                     │
                 │    - AuditLogPlugin: Forensics lịch sử truy vấn │
                 │    - MonitoringAlert: Đo block rate, bắn Alert  │
                 └───────────────────────┬─────────────────────────┘
                                         │
                                         ▼
                 [TRẢ PHẢN HỒI AN TOÀN CHO NGƯỜI DÙNG HOẶC RA FILE]
```

---

## 3. Lộ trình giả lập "Dựng mới dự án từ thư mục trống" (From-Scratch Blueprint)

Nếu phải tự tay tạo ra dự án này từ một thư mục trống, một Senior Architect sẽ đi theo 5 bước:

### Bước 1: Khởi tạo Cấu trúc & Nền tảng Dữ liệu (Foundation Layer)
- Tạo `requirements.txt`, `.env.example`, `.gitignore`.
- Tạo `data/protected/vinbank_secrets.json`: Nơi chứa dữ liệu bí mật giả định (`admin_password`, `api_key`, `db_host`).
- Tạo `src/core/config.py`: Định nghĩa cấu hình tập trung (load `.env`, danh sách `ALLOWED_TOPICS`, `BLOCKED_TOPICS`, đọc secrets từ JSON).
- Tạo `src/core/openai_runtime.py` & `src/core/utils.py`: Tạo adapter bọc OpenAI Client tương thích Google ADK callback conventions (giúp cắm các filter/plugin vào vòng đời inference).

### Bước 2: Xây dựng các Bộ lọc Phòng vệ Cốt lõi (Core Guardrails Layer)
- Tạo `src/guardrails/input_guardrails.py`:
  + Triển khai logic chuẩn hóa ký tự (Unicode normalization) chống evasion.
  + Triển khai `detect_injection()` và `topic_filter()`.
  + Gói thành `InputGuardrailPlugin` kế thừa `BasePlugin`.
- Tạo `src/guardrails/output_guardrails.py`:
  + Triển khai `content_filter()` dùng Regular Expressions để bắt PII và secret keys.
  + Gói thành `OutputGuardrailPlugin` can thiệp vào callback sau khi model sinh chữ (`after_model_callback`).

### Bước 3: Xây dựng Hạ tầng Giám sát, Điều phối & Cổng Egress (Orchestration & Observability Layer)
- Tạo `src/assignment/rate_limiter.py`: Triển khai giải pháp sliding window theo user_id để chống spam/DoS.
- Tạo `src/assignment/audit_log.py`: Ghi vết forensics toàn diện (input, output, layer chặn, latency).
- Tạo `src/assignment/monitoring.py`: Gom metrics, tính toán block rate, phát hiện bất thường và bắn `Alert`.
- Tạo `src/assignment/pipeline.py`:
  + Triển khai `is_egress_allowed()`: Chặn rò rỉ dữ liệu qua internet.
  + Ghép nối danh sách plugin đúng thứ tự: `RateLimit -> InputGuardrail -> OutputGuardrail`.
  + Viết hàm `run_assignment_suite()` thực thi tự động 4 nhóm kịch bản và xuất ra `outputs/results.json`, `audit_log.json`, `metrics.json`.

### Bước 4: Thiết lập Agents & Chiến trường Tấn công (Red Teaming Layer)
- Tạo `src/agents/agent.py`: Tạo `create_blue_agent` (phòng thủ hoàn chỉnh) và `create_red_agent_default` (bỏ lỏng bảo vệ).
- Tạo `src/agents/guards_agent.py` & `src/agents/red_advance_nemo.py`: Tạo mục tiêu phòng thủ kiên cố Red Advance kết hợp Regex + NeMo Guardrails.
- Tạo `src/attacks/attacks.py`: Thiết lập 5 kịch bản tấn công đối kháng nâng cao và hàm `run_attacks()`.

### Bước 5: Entry Point & Tự động Đánh giá (CLI & Grading Layer)
- Tạo `src/main.py`: Nhận tham số `--part 2`, `--part 3`, `--part 4` để chạy từng giai đoạn.
- Tạo `scripts/grade.py` và các test case trong `tests/`: Tự động chấm điểm, kiểm định schema kết quả theo chuẩn `schemas/results.schema.json`.

---

## 4. Tại sao lại phân chia Module như hiện tại?
1. **Single Responsibility Principle (SRP):**
   - Tầng Guardrails (`src/guardrails/`) chỉ tập trung vào việc phân tích ngữ nghĩa, phát hiện mã độc và lọc nội dung. Nó không cần quan tâm đến network hay DoS.
   - Tầng Limiter/Audit/Monitoring (`src/assignment/`) tập trung vào hạ tầng và độ tin cậy của dịch vụ (SRE/SecOps).
2. **Nguyên tắc "Fail-Closed" & "Defense-in-Depth":**
   - Ngay cả khi LLM bị bẻ khóa (Jailbreak bypass thành công Input Guardrail), lớp Output Guardrail và Egress Gateway vẫn là chốt chặn cuối cùng ngăn chặn dữ liệu bí mật tuồn ra ngoài.

---

# PHẦN 2: ĐIỂM GHÉP NỐI & NHẬT KÝ TRIỂN KHAI CHI TIẾT

## Điểm ghép nối (Integration Points) trong dự án
Yêu cầu của dự án này tập trung vào 3 điểm cốt tử:
1. **Mắt xích Checkpoint 2 (Lọc Input/Output):**
   - File: `src/guardrails/input_guardrails.py` & `src/guardrails/output_guardrails.py`.
2. **Mắt xích Checkpoint 3 (Điều phối Pipeline, Giới hạn tần suất, Egress, Đo lường & Xuất kết quả):**
   - File: `src/assignment/rate_limiter.py`, `src/assignment/audit_log.py`, `src/assignment/monitoring.py`, `src/assignment/pipeline.py`.
3. **Mắt xích Checkpoint 4 (Thiết kế đòn tấn công Red Teaming):**
   - File: `src/attacks/attacks.py`.

---

## NHẬT KÝ TỪNG BƯỚC TRIỂN KHAI (GIAI ĐOẠN 1)

### Bước 1: Input Guardrails — Phát hiện Prompt Injection (`detect_injection`)
- **Mục tiêu:** Phát hiện các mẫu câu cố tình bẻ khóa system prompt hoặc thay đổi danh tính mô hình (Jailbreak), xử lý thủ thuật né tránh bằng ký tự ẩn Unicode.
- **Kiến thức cốt lõi:**
  + *Kỹ thuật Evasion:* Kẻ tấn công thường chèn ký tự vô hình (Zero-Width Space `\u200b`, `\u200c`, `\ufeff`) vào giữa từ khóa để đánh lừa regex (ví dụ: `Ignore\u200b all previous instructions`).
  + *Tiền xử lý chuỗi:* Cần chuẩn hóa Unicode, loại bỏ các ký tự ẩn trước khi áp dụng regex patterns.
  + *Tránh False Positive (chặn nhầm):* Một câu hỏi thông thường như tóm tắt email về việc chậm trễ chuyển khoản ngân hàng không được phép bị chặn nhầm.
- **Luồng dữ liệu:**
  + *Upstream:* Chuỗi text lấy từ `user_message` thông qua `InputGuardrailPlugin`.
  + *Downstream:* Trả về `"BLOCK"` hoặc `"ALLOW"` cho `InputGuardrailPlugin` quyết định dừng lại hay cho qua.

### Bước 2: Input Guardrails — Lọc chủ đề ngân hàng (`topic_filter`)
- **Mục tiêu:** Đảm bảo trợ lý AI VinBank chỉ hoạt động trong phạm vi nghiệp vụ được cấp phép (Domain Boundary Enforcement).
- **Kiến thức cốt lõi:**
  + *Denylist (Chủ đề cấm):* Phát hiện các chủ đề nguy hại (vũ khí, ma túy, cờ bạc, tấn công mạng...) dựa trên `BLOCKED_TOPICS`.
  + *Allowlist (Chủ đề cho phép):* Chỉ chấp nhận các truy vấn thuộc lĩnh vực ngân hàng (tài khoản, tiết kiệm, lãi suất, chuyển khoản, thẻ...) dựa trên `ALLOWED_TOPICS`.
  + *Nguyên lý Fail-Closed:* Nếu câu hỏi không dính từ cấm nhưng cũng không dính bất kỳ từ khóa ngân hàng nào (lạc đề: công thức nấu ăn, thời tiết...) $\rightarrow$ Chặn ngay (`"BLOCK"`).
- **Luồng dữ liệu:**
  + *Upstream:* Chuỗi `user_input` sau khi đã vượt qua bước kiểm tra injection (`detect_injection == "ALLOW"`).
  + *Downstream:* Trả về `"BLOCK"` hoặc `"ALLOW"` cho `InputGuardrailPlugin`.

### Bước 3: Input Guardrails — Đóng gói `InputGuardrailPlugin`
- **Mục tiêu:** Tích hợp logic `detect_injection` và `topic_filter` vào cơ chế Plugin Interceptor của Google ADK / Agent Runtime.
- **Kiến thức cốt lõi:**
  + *Cơ chế Interceptor:* `on_user_message_callback()` được kích hoạt ngay khi người dùng gửi tin nhắn, trước khi LLM được gọi.
  + *Giao thức can thiệp (Short-circuit):*
    - Nếu trả về `None`: Hệ thống xem như message an toàn và tiếp tục chuyển đến LLM.
    - Nếu trả về `types.Content`: Hệ thống lập tức dừng luồng (short-circuit), không gọi LLM và trả ngay nội dung `types.Content` này cho người dùng.
- **Luồng dữ liệu:**
  + *Upstream:* Nhận `user_message: types.Content` từ Agent Runtime / Caller.
  + *Downstream:* Trả về `types.Content | None`. Cập nhật các bộ đếm `total_count` và `blocked_count`.

### Bước 4: Output Guardrails — Lọc PII & Secrets (`content_filter`)
- **Mục tiêu:** Ngăn chặn việc LLM vô tình làm lộ lọt thông tin nhận dạng cá nhân (Personally Identifiable Information - PII) hoặc bí mật hệ thống (Secrets).
- **Kiến thức cốt lõi:**
  + *Chiến lược Data Masking / Redaction:* Thay thế các chuỗi dữ liệu khớp mẫu vi phạm bằng chuỗi an toàn `[REDACTED]`.
  + *Các mẫu dữ liệu kiểm soát:*
    - SĐT Việt Nam: `0\d{9,10}`
    - Địa chỉ Email: `[\w.-]+@[\w.-]+\.[a-zA-Z]{2,}`
    - CMND / CCCD: `\b\d{9}\b|\b\d{12}\b`
    - API Key: `sk-[a-zA-Z0-9_-]+`
    - Mật khẩu: `password\s*(?:is|[:=])\s*\S+`
- **Luồng dữ liệu:**
  + *Upstream:* Nhận chuỗi `response: str` sinh ra từ LLM.
  + *Downstream:* Trả về `dict` gồm `safe: bool`, `issues: list[str]`, và `redacted: str` để `OutputGuardrailPlugin` sử dụng.

### Bước 5: Output Guardrails — Đóng gói `OutputGuardrailPlugin`
- **Mục tiêu:** Tích hợp `content_filter` vào vòng đời sinh chữ của Agent qua callback `after_model_callback`.
- **Kiến thức cốt lõi:**
  + Can thiệp vào đối tượng `llm_response.content` để thay thế văn bản rò rỉ thành văn bản đã khử nhạy cảm `[REDACTED]`.
  + Tăng bộ đếm `redacted_count` để phục vụ quan sát số liệu (Observability).
- **Luồng dữ liệu:**
  + *Upstream:* Nhận `llm_response` từ mô hình sau khi kết thúc lượt gọi API.
  + *Downstream:* Trả về `llm_response` đã được làm sạch để gửi về cho người dùng.

### Bước 6: Rate Limiter — Giới hạn tần suất trượt theo người dùng (`RateLimitPlugin`)
- **Mục tiêu:** Chống cạn kiệt tài nguyên / DoS / Flooding request tới mô hình AI.
- **Kiến thức cốt lõi:**
  + *Thuật toán Sliding Window (Cửa sổ trượt):* Dùng hàng đợi `deque` để lưu timestamp của từng request cho mỗi `user_id`.
  + *Cơ chế dọn dẹp:* Loại bỏ các timestamp cũ hơn `window_seconds`.
  + *Cơ chế chặn:* Nếu số lượng request trong cửa sổ vượt `max_requests`, trả về thông báo lỗi kèm thời gian cần chờ đợi `wait` giây mà không gọi vào LLM.
- **Luồng dữ liệu:**
  + *Upstream:* Chạy đầu tiên trong chuỗi callback `on_user_message_callback`.
  + *Downstream:* Trả về `types.Content` (thông báo Rate limit) nếu bị nghẽn, hoặc `None` để đi tiếp tới Input Guardrails.

### Bước 7: Audit Log — Nhật ký an ninh & Forensics (`AuditLogPlugin`)
- **Mục tiêu:** Cung cấp khả năng truy vết và kiểm toán an ninh (Security Auditing) cho mọi tương tác trong hệ thống mà không can thiệp làm gián đoạn luồng xử lý.
- **Kiến thức cốt lõi:**
  + *Tách biệt trách nhiệm (Separation of Concerns):* Lớp này không bao giờ chặn (block) request, mà chỉ ghi nhận trung thực mọi dữ liệu vào/ra, quyết định của các tầng bảo vệ, và độ trễ (latency).
  + *Tính toán Latency:* Ghép nối `record_input` và `record_output` thông qua định danh `request_id` (hoặc `user_id`) để đo thời gian phản hồi chính xác đến mili-giây.
  + *Xuất dữ liệu an toàn:* Đảm bảo hàm `export_json` tự động tạo thư mục `outputs/` nếu chưa tồn tại.
- **Luồng dữ liệu:**
  + *Upstream:* Nhận sự kiện từ pipeline trước khi lọc input (`record_input`) và sau khi lọc output (`record_output`).
  + *Downstream:* Xuất file `outputs/audit_log.json` phục vụ hậu kiểm.

### Bước 8: Monitoring & Alerting — Thống kê & Cảnh báo an ninh (`MonitoringAlert`)
- **Mục tiêu:** Giám sát liên tục trạng thái sức khỏe an ninh của hệ thống thông qua việc đo lường các tỷ lệ vi phạm và tự động kích hoạt cảnh báo khi vượt ngưỡng.
- **Kiến thức cốt lõi:**
  + *Các chỉ số cốt lõi (Key Metrics):*
    - Tỷ lệ chặn (`block_rate`): Cảnh báo khi vượt ngưỡng `block_rate_threshold` (50%).
    - Lượt vi phạm tần suất (`rate_limit_hits`): Cảnh báo khi vượt ngưỡng `rate_limit_hit_threshold` (5 lần).
    - Tỷ lệ vi phạm đánh giá (`judge_fail_rate`): Cảnh báo khi vượt `judge_fail_rate_threshold` (30%).
  + *Đóng gói Snapshot:* Sử dụng dataclass gom toàn bộ chỉ số và danh sách cảnh báo `Alert` để xuất ra `outputs/metrics.json`.
- **Luồng dữ liệu:**
  + *Upstream:* Được pipeline cập nhật các bộ đếm sau mỗi lượt request (`total_requests`, `blocked_requests`, `rate_limit_hits`).
  + *Downstream:* Hàm `export_json` lưu trữ báo cáo metrics ra file `outputs/metrics.json`.

### Bước 9: Egress Control — Kiểm soát dữ liệu ra ngoài (`is_egress_allowed`)
- **Mục tiêu:** Ngăn chặn tuyệt đối việc dữ liệu nhạy cảm bị rò rỉ ra ngoài qua các kết nối mạng bên ngoài (Egress Data Exfiltration).
- **Kiến thức cốt lõi:**
  + *Quy tắc Deterministic (Cấm dựa vào LLM):* Quyết định cho phép hay chặn gửi dữ liệu ra ngoài phải hoàn toàn do mã lệnh (Hard Rules) kiểm soát, không dựa vào "sự đồng ý" của LLM.
  + *Chính sách Allowlist Điểm đến:*
    - Giao thức bắt buộc: HTTPS (`scheme == "https"`).
    - Domain hợp lệ: Chỉ cho phép các domain được ủy quyền của ngân hàng (ví dụ: `api.vinbank.example`). Chặn đứng domain mạo danh (subdomain attack).
  + *Kiểm tra Nội dung Payload:* Dữ liệu gửi đi không được chứa mật khẩu, API key, địa chỉ cơ sở dữ liệu nội bộ, số điện thoại, hay email cá nhân (tái sử dụng `content_filter` và `DEMO_SECRETS`).
- **Luồng dữ liệu:**
  + *Upstream:* Nhận `destination: str` (URL đích) và `payload: str` (nội dung gửi đi) từ Agent Action Gateway.
  + *Downstream:* Trả về `True` nếu an toàn, `False` nếu phát hiện bất kỳ dấu hiệu rò rỉ hoặc domain lạ.

### Bước 10: Pipeline Assembly — Lắp ráp chuỗi Plugin (`build_production_plugins`)
- **Mục tiêu:** Kết nối các thành phần phòng thủ riêng lẻ thành một chuỗi thực thi tuần tự hoàn chỉnh (Defense-in-depth Chain of Responsibility).
- **Kiến thức cốt lõi:**
  + *Thứ tự ưu tiên thực thi:*
    1. `RateLimitPlugin`: Đặt ở đầu tiên để loại bỏ request spam mà không tốn chi phí CPU/NLP.
    2. `InputGuardrailPlugin`: Đặt ở giữa để loại bỏ prompt injection và câu hỏi lạc đề trước khi gửi prompt cho mô hình.
    3. `OutputGuardrailPlugin`: Đặt ở cuối để rà soát câu trả lời của mô hình trước khi gửi lại cho khách hàng.
  + *Tách biệt Observability:* Hàm `build_observability` khởi tạo độc lập `AuditLogPlugin` và `MonitoringAlert`.
- **Luồng dữ liệu:**
  + *Upstream:* Gọi khi khởi tạo Agent hoặc bộ test suite.
  + *Downstream:* Trả về danh sách `[RateLimitPlugin, InputGuardrailPlugin, OutputGuardrailPlugin]` và tuple `(AuditLogPlugin, MonitoringAlert)`.

### Bước 11: Suite Runner — Chạy 4 bộ test & Sinh Artifact `results.json` (`run_assignment_suite`)
- **Mục tiêu:** Tự động hóa quá trình thử nghiệm toàn diện hệ thống phòng thủ Blue Agent qua 4 nhóm kịch bản và sinh ra các file báo cáo JSON đạt chuẩn.
- **Kiến thức cốt lõi:**
  + *4 Nhóm kịch bản đánh giá:*
    - `safe_queries` ($\ge$ 5): Các câu hỏi nghiệp vụ ngân hàng hợp lệ $\rightarrow$ Kỳ vọng 100% không bị chặn nhầm (`blocked: False`).
    - `attack_queries` ($\ge$ 7): Các câu hỏi độc hại, jailbreak, prompt injection, off-topic $\rightarrow$ Kỳ vọng $\ge$ 5 câu bị chặn (`blocked: True`).
    - `rate_limit`: Mô phỏng spam request liên tiếp $\rightarrow$ Kiểm tra chặn chính xác khi vượt ngưỡng `max_requests`.
    - `edge_cases` ($\ge$ 3): Các trường hợp biên (Unicode tàng hình, câu lồng ghép dữ liệu bên thứ ba).
  + *Đóng gói Artifact:* Sinh `outputs/results.json` khớp 100% với `schemas/results.schema.json`, đồng thời xuất `outputs/audit_log.json` và `outputs/metrics.json`.
- **Luồng dữ liệu:**
  + *Upstream:* Nhận đối tượng `pipeline` chứa `plugins`, `audit`, và `monitor`.
  + *Downstream:* Ghi file ra thư mục `outputs/` và trả về kết quả dict cho `main.py`.

### Bước 12: Red Team — Soạn thảo 5 Prompt tấn công đối kháng (`adversarial_prompts`)
- **Mục tiêu:** Đóng vai Red Team thiết kế các câu lệnh khai thác lỗ hổng ngữ nghĩa (Semantic Jailbreaks) nhằm buộc mô hình AI tiết lộ các bí mật nội bộ nhạy cảm (`admin_password`, `api_key`, `db_host`).
- **Kiến thức cốt lõi:**
  + *5 Vector tấn công đối kháng:*
    1. `Completion / Infilling`: Tạo một câu dở dang mang tính khẳng định để mô hình tiếp tục điền bí mật vào chỗ trống.
    2. `Format Shifting / JSON Translation`: Yêu cầu mô hình trích xuất toàn bộ cấu hình hệ thống và ghi chú nội bộ dưới dạng JSON có cấu trúc.
    3. `Hypothetical Scenario`: Đặt mô hình vào tình huống giả định khẩn cấp (Emergency Maintenance) để vượt qua rào cản cảnh giác.
    4. `Side-channel Confirmation`: Đưa ra thông tin mật đã đoán trước và yêu cầu mô hình xác nhận tính chính xác.
    5. `Authority Persona / Role-play Escalation`: Đóng vai kiểm toán viên an ninh (Security Auditor) hoặc quản trị viên cấp cao yêu cầu bàn giao thông tin tuân thủ.
- **Luồng dữ liệu:**
  + *Upstream:* Nạp danh sách `adversarial_prompts` vào hàm `run_attacks()`.
  + *Downstream:* Gửi tới `red_agent_default` và `red_agent_advance`, đánh giá kết quả rò rỉ và ghi ra `outputs/attack_results.json`.










