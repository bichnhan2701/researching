# Dữ liệu đầu vào để nghiên cứu việc chuyển bớt tác vụ khỏi Qwen3.6

> **Mục đích.** Hiện hầu hết tác vụ LLM của hệ thống `staging/` chạy trên một model chung (Qwen3.6-35B-A3B). **Hướng nghiên cứu hiện tại: không dùng một model riêng cho mỗi tác vụ; chuyển một phần tác vụ sang một vài đích khác** (mã/bộ mã hóa, một endpoint nhỏ dùng chung, làn nền) để Qwen không phải gánh toàn bộ khối lượng đầu vào. Mục 4 xếp tác vụ vào bậc giải pháp và đích chuyển, ưu tiên theo khối lượng đầu vào; mục 11 là bước đầu (Decision). Tài liệu này **trích từ repo** những thông tin cần cho việc đó: yêu cầu của từng tác vụ, ràng buộc hệ thống, các model đã thử và số đo, dữ liệu đánh giá sẵn có, và mức độ tách rời hiện tại của cấu hình.
>
> **Không có** khuyến nghị chọn model cụ thể: tài liệu chỉ cung cấp dữ kiện. Chỗ nào là suy luận của tôi, tôi ghi "(suy luận)". Tên model ứng viên bên ngoài repo chưa được nghiên cứu ở đây. Mục 4 và mục 11 là **đề xuất quy trình** (suy luận), không phải lựa chọn model.
>
> **Liên quan:** `LLM_TASKS_PROMPTS_VI.md` (prompt nguyên văn của 9 tác vụ), `STAGING_OVERVIEW_VI.md` (tổng quan; mục 18 là cấu hình `.env`). Ba tác vụ phụ #10–#12 (mục 3) chưa có prompt nguyên văn trong `LLM_TASKS_PROMPTS_VI.md`.
>
> **Quy ước.** `O/` = `staging/services/voice-ai-orchestrator/src/voice_ai_orchestrator/`; `E/` = `staging/artifacts/eval/`. Số liệu độ trễ là **trung vị (p50)** nếu không ghi khác, đơn vị ms. Chưa chạy lại bất kỳ phép đo nào; mọi số lấy từ báo cáo đã có trong repo.

---

## 1. Tóm tắt những điều quan trọng nhất

1. **Không có tác vụ nào ngoài #5 (trích xuất trường) hiện dùng model chuyên biệt.** #5 là ví dụ duy nhất trong repo cho thấy một model nhỏ chuyên dụng (NuExtract-2.0-2B) đã được đo đối đầu với Gemini trên bộ vàng: nhanh hơn 3,9–4,1 lần, bịa giá trị (invention) tương đương, nhưng bỏ sót nhiều hơn.
2. **Chỉ Decision có bộ đánh giá nhãn thật** (`E/decision_corpus`, 207 dòng người duyệt + 52 ca tự soạn). Nhưng bộ 207 dòng thật **không có kind an toàn**: 0 `reject`, 0 `escalate`, 0 `courtesy`; kind an toàn chỉ được kiểm bằng dữ liệu tự soạn (55 dòng `reject` trong 16 họ). Với anger judge, bridge, grounded reply, sentiment (text), call purpose và post-call **không thấy** bộ nhãn hoặc benchmark riêng trong `E/` (tìm bằng từ khóa; có thể sót, xem mục 9).
3. **Độ chi tiết cấu hình hiện tại hạn chế việc tách.** Decision, Reply, Sentiment-text, Extraction có biến môi trường riêng. Nhưng **anger judge, grounded reply, call purpose, post-call, KB step validation, intent fallback, publish wording dùng chung MỘT endpoint** `LOCAL_JSON_LLM_*`; **bridge dùng chung endpoint với Reply** (`LOCAL_LLM_*`). Tách các tác vụ này cần sửa code nhỏ (mục 6).
4. **Đường găng (critical path) của một lượt chỉ có vài tác vụ**: Decision (~290–350 ms trên Qwen), Reply đến câu đầu (~740–940 ms), và Bridge (che thời gian chờ). Anger judge chạy song song nhưng cùng GPU. Hai tác vụ phụ nằm trên đường găng **có điều kiện**: Intent fallback (#11, chỉ ở lượt chọn workflow mà từ khóa không khớp) và KB step validation (#10, chỉ ở workflow có bước kiểu `knowledge_base`). Các tác vụ nền (sentiment, purpose, post-call) không nằm trên đường găng nhưng **vẫn làm chậm reply vì cùng một GPU** (sentiment text: +204–640 ms ttfa). Publish wording (#12) chạy ngoài cuộc gọi, lúc tenant publish workflow.
5. **Rào cản đo lường**: tất cả đo trên GPU dùng chung; báo cáo ghi nhiều lần rằng tải từ stack khác làm số đo dao động cả bậc (Decision từ 0,23 s đến 1,9 s). Ngoài ra Decision ở nhiệt độ 0 vẫn không lặp lại hoàn toàn (khác nhau 11/217 yêu cầu giữa hai lần chạy giống hệt).
6. **Yêu cầu tương thích với máy chủ** là ràng buộc cứng với model ứng viên: tắt thinking (`enable_thinking`), tool calling, `continue_final_message` (bridge), `structured_outputs`/`response_format` (JSON), streaming có `usage`. Danh sách ở mục 7.
7. **Cách làm hiện tại: chuyển bớt tác vụ khỏi Qwen sang vài đích dùng chung, ưu tiên theo khối lượng đầu vào (mục 4); Decision làm trước.** Ước lượng đầu vào Qwen nhận mỗi lượt từ 5 tác vụ chạy mỗi lượt là khoảng 5,9–6,6k token (mục 3). Reply chiếm ~4,1–4,4k và khó chuyển nhất; Decision là phần lớn nhất có thể chuyển (~1,1–1,3k, không bao giờ trúng prefix cache). Một biến `LOCAL_JSON_LLM_BASE_URL` chuyển cùng lúc cả họ tác vụ phụ (mục 6). Với Decision, đường rẻ nhất là đổi model qua `DECISION_LLM_BASE_URL/MODEL` (không sửa code) và chạy lại harness baseline đã có (mục 11). Đường đầu cục bộ (`hybrid`) chưa dùng được: README ghi đầu đã giao đạt **0,0 trên `reject`**, đặc trưng `workflow_active` bị lỗi ở dữ liệu D1, lệch huấn luyện/chạy cho v7 chưa sửa, và thư mục `assets/decision/` rỗng có chủ ý trong git.

---

## 2. Bối cảnh hệ thống ảnh hưởng đến việc chọn model

### 2.1 Luồng một lượt và thứ tự phụ thuộc

Xem `LLM_TASKS_PROMPTS_VI.md` mục 12. Tóm tắt các phụ thuộc:

- **Decision** chạy trước; **Reply chờ Decision** (câu trả lời cần biết kind). Decision và Reply là hai request tuần tự trên đường găng.
- **Anger judge** chạy song song với Decision (cùng GPU, nên tranh chấp).
- **Bridge** bắt đầu ở mỗi STT segment final / early final, đợi tối đa 700 ms ở cuối lượt.
- **Extraction** chạy ngay sau Decision, chỉ khi capability sở hữu cuộc gọi chạy; một lượt có thể có 2 lần gọi (lần hai khi trường vừa hỏi bị để trống).
- **Sentiment-text** bắt đầu sau khi reply sinh xong; **Call purpose** mỗi 4 lượt; **Post-call** khi kết thúc.

### 2.2 Phần cứng và máy chủ model (đã thấy trong repo)

| Hạng mục | Thông tin | Nguồn |
|---|---|---|
| Staging chính | "GB10" — **một GPU dùng chung** cho Decision, Reply, TTS, STT, emotion, extraction; các stack khác (`truong-*`, `van-*`, `hqvu-*`) cũng dùng chung vLLM và NuExtract | `E/llm-stage-latency-2026-10-11/report.md`, `E/extraction-wait-2026-10-19/RESULT.md` |
| Máy chủ LLM chính | vLLM 0.30.0, model `nvidia/Qwen3.6-35B-A3B-NVFP4` (tên phục vụ `qwen3.6-35b-a3b`), cổng host 8900 | `E/decision-baseline-2026-10-13/report.md` |
| Máy chủ LLM cũ | `mistralai/Ministral-3-8B-Instruct-2512` FP8, tên phục vụ `ministral`, `--max-model-len 16384`, `--max-num-seqs 32`, `--gpu-memory-utilization 0.60`, `--enable-prefix-caching`, tool calling bằng parser mistral | `selfhosted-ai/llm/docker-compose.vllm.yml` |
| Trích xuất trường | NuExtract-2.0-2B FP8 (vLLM) hoặc Q4_K_M GGUF (llama.cpp), trọng số ~2,5 GB, `max-model-len 4096` | `selfhosted-ai/extractor/` |
| Prefix cache của Qwen3.6 | Model lai (hybrid): vLLM chỉ tái dùng **khối nguyên 2096 token** giống nhau từ đầu. Reply lượt 1: 0 token từ cache; các lượt sau: 2096. Decision (1,1–1,3k token) **không bao giờ** trúng cache vì ngắn hơn một khối | `E/gb10-latency-context-2026-10-12.md` mục 4 |
| Máy phát triển cũ (đo wave 2) | RTX 5060 Ti 16 GB và RTX 3060 12 GB; Ollama chạy `llama3.2:3b` và `llama3.1:8b` (4-bit) | `E/wave2-local-generation-results-2026-08-22.md`, `docs/archive/docs/latency-architecture/runbook-gpu-placement.md` |
| Bộ nhớ GPU còn lại | Ghi chú trong compose: "vLLM dùng chung đã giữ ~60% GPU; hạ `SH_VLLM_GPU_MEM_UTIL` hoặc dừng cái kia trước khi chạy model thứ hai" | `selfhosted-ai/llm/docker-compose.vllm.yml` |

(Suy luận: nhiều model chuyên biệt nhỏ sẽ cạnh tranh VRAM và băng thông với Qwen 35B; ngân sách GPU là biến số chính của nghiên cứu. Chưa có số đo VRAM hiện tại trong tài liệu này.)

### 2.3 Ràng buộc chung

| Ràng buộc | Chi tiết | Nguồn |
|---|---|---|
| Mục tiêu độ trễ | PRD: **1,5 s** từ khi người gọi ngừng nói đến audio đầu tiên. Ngân sách đề xuất theo bước: endpointing 200–300; Decision 300–400; Reply tới câu đầu 300–500; TTS câu đầu 150–250; tổng 1100–1500 | `E/gb10-latency-context-2026-10-12.md` mục 3 |
| Trạng thái hiện tại (setup D: Qwen3.6 + Kokoro, text-emotion tắt) | endpointing 1024; Decision 286; Reply→câu đầu 736; Kokoro 140; **người gọi chờ 2215** | như trên |
| Ngôn ngữ | Các thử nghiệm và spec ghi "không cần xét tiếng Việt lúc này" (phê duyệt 2026-10-15); sản phẩm vẫn có cờ `vi` cho agent. Dữ liệu đánh giá hiện là tiếng Anh | `artifacts/spec/orchestrator/turn-pipeline.yaml` (1.3.0) |
| Đầu ra phải máy đọc được | Decision: 1 token/dòng; extraction: JSON theo template; sentiment/purpose/post-call: JSON; parse thất bại → fallback an toàn | `O/conversation_orchestrator/decision/prompt.py` |
| Fail-safe | Mọi tác vụ trả `None`/câu dự phòng khi lỗi; Decision lỗi → `fallback`; không có confidence (RIA-9) | `O/ai/grounded_reply.py`, `producer.py` |
| Xác định (determinism) | Nhiệt độ 0 vẫn không lặp lại hoàn toàn (xem 5.1) | `E/decision-baseline-2026-10-13/report.md`, `E/decision-latency-ideas-2026-10-19/RESULT.md` |
| PII | Transcript có thể chứa PII; prompt của call purpose/post-call gửi transcript thô; mask chỉ áp dụng khi lưu (`textMasked`) | `O/ai/caller_purpose.py` (docstring `build_prompt`) |
| Quy tắc ngôn ngữ agent | Không hứa chuyển máy / kết nối người / gọi lại (RP4) — ràng buộc lên **mọi** model sinh câu nói | `artifacts/spec/orchestrator/turn-pipeline.yaml` |

---

## 3. Hồ sơ yêu cầu từng tác vụ

Kích thước token lấy từ báo cáo khi có; chỗ khác ước lượng từ prompt trong code, ghi "(ước lượng)".

| # | Tác vụ | Đường găng? | Đầu vào (token) | Đầu ra | Loại bài toán | Chỉ tiêu đã dùng / hậu quả sai |
|---|---|:-:|---|---|---|---|
| 1 | **Decision** | **Có** (Reply chờ) | **1,1–1,3k** (persona + 11 kind + hội thoại 12 lượt + facts) | **~2–3 token** (`kind` hoặc `kind\|ref`), `max_tokens` 32, timeout 3 s | Phân loại 11 lớp theo ngữ cảnh, có tham chiếu từ tập đóng | Đúng kind/action; tính nhất quán; sai kind = sai lượt hội thoại (`reject`, `escalate` là kind an toàn) |
| 2 | **Anger judge** | Song song Decision | ~150–250 (ước lượng) | **1 từ** `yes`/`no`, `max_tokens` 2, timeout 2 s | Phân loại nhị phân | Dương tính giả = **khóa cuộc gọi vào hold đến hết**; âm tính giả = bỏ lỡ khách giận |
| 3 | **Bridge** | **Có** (che độ trễ), chờ ≤700 ms | ~400–500 (ước lượng; khối prompt 278 từ / 1404 ký tự, cộng persona và user message) | **1–4 từ** (≤5), `max_tokens` 12, stream, temp 0.7 | Sinh rất ngắn, đa dạng, đúng giọng | Không số, không câu hỏi, không hứa, không tự nhận là người; text token đầu mất **170–360 ms** (đo) |
| 4 | **Reply** | **Có** | **~4,1–4,4k** (system ~2014 + tools 362 + knowledge 1683–1855 + lịch sử + lời gọi) | Tối đa 320 token; câu đầu đến TTS ngay | Sinh hội thoại có căn cứ + tool calling | Không bịa (invention), bám knowledge, ngắn, đúng quy tắc điện thoại |
| 5 | **Field extraction** | Phần lớn **không**; có thể chặn lượt sau | Template + câu người gọi (nhỏ, <4096 ctx) | JSON các trường (verbatim) | Trích xuất có cấu trúc | **Invention ≤ baseline**, bỏ sót, nhầm trường, chữ số |
| 6 | **Grounded reply** | Trong workflow `loan_status` và câu hold | ~150–250 (ước lượng) | 1–2 câu, `max_tokens` 90/60, timeout 4 s | Sinh ngắn chỉ từ facts cho trước | Không thêm sự kiện ngoài danh sách |
| 7 | **Sentiment-text** | Không (nền) | ~100–150 (ước lượng) | JSON 6 nhãn, `max_tokens` 200 | Phân loại 6 lớp (xác suất) | Cấp cho `[tone: X]` (ngưỡng 0,55) và leo thang (ngưỡng 0,8) |
| 8 | **Call purpose** | Không (nền) | Toàn bộ transcript đến lúc đó + hợp đồng (tăng theo cuộc gọi) | JSON `{purpose, relation}`, `max_tokens` 120, timeout 8 s | Tóm tắt có điều kiện (giữ/làm mịn/đổi) | Không dùng filler chung chung; mặc định giữ giá trị cũ |
| 9 | **Post-call analysis** | Không (khi kết thúc) | Toàn bộ transcript + tín hiệu | JSON 6 trường, `max_tokens` 600 | Tóm tắt có cấu trúc, phong cách "supervisor" | Cụ thể, không kể chuyện, không bịa; tags 3–6 |

**Ba tác vụ phụ khác dùng chung `LOCAL_JSON_LLM_*`** (tất cả qua `GeminiCaller`, chỉ chạy trên Qwen khi `LOCAL_JSON_LLM_BASE_URL` được đặt; nguồn là code, chưa đo):

| # | Tác vụ | Đường găng? | Khi nào chạy | Đầu vào | Đầu ra | Ghi chú |
|---|---|:-:|---|---|---|---|
| 10 | **KB step validation** (`workflows/field_validation/knowledge_base.py`) | **Có, có điều kiện** (đồng bộ trong thread engine; câu hỏi kế tiếp chờ kết quả) | Chỉ ở bước cấu hình `source: knowledge_base, comparison: custom`, sau khi giá trị qua kiểm tra kiểu; một lần cho mỗi giá trị (giá trị không đổi dùng lại kết quả) | `instruction` của doanh nghiệp + các đoạn truy xuất từ KB + giá trị người gọi (đều bị cắt độ dài); chưa đo token | JSON `{verdict, confidence}`, `max_tokens` 64, temp 0, thinking tắt, có `response_schema` | **Hai bước tuần tự**: truy xuất KB (timeout 3 s) rồi LLM (timeout 4 s), xấu nhất ~7 s. Trong lượt suy đoán (early lane) nó **không chạy** mà bỏ lượt suy đoán, nên early lane không che được độ trễ này. Lỗi hoặc timeout → trường không được chấp nhận |
| 11 | **Intent fallback** (`workflows/llm_router.py`, gọi từ `ai/routing_engine.py::_select_intent`) | **Có, có điều kiện** (bọc `asyncio.to_thread`, tính vào `phases.routing`) | Chỉ khi chọn workflow và đồng thời: không có workflow ép từ campaign, không phải câu chấp nhận một lời đề nghị, **từ khóa không khớp** | Catalog intent (roster workflow đã publish + mô tả enum) + lời người gọi; chưa đo token | JSON `{intent, confidence, reason}`, temp 0; điểm gọi không đặt `max_tokens` | Timeout 4 s. Lỗi → `None` → `UNKNOWN_INTENT`, không bao giờ route một id không có trong roster. Cuộc gọi chỉ hỏi đáp có thể không chạy lần nào |
| 12 | **Publish wording** (`management/api.py::_question_model`, endpoint `/workflows/publish-check`) | **Không** (ngoài cuộc gọi) | Khi tenant publish workflow trên dashboard; một lần gọi cho mỗi bước cần sinh câu hỏi | Prompt sinh câu hỏi cho từng bước | Mảng JSON các chuỗi (`list[str]`), `max_tokens` 1024, thinking tắt, có `response_schema` | Timeout 20 s mỗi lần gọi. **Fail-closed**: không sinh được thì từ chối publish. Vẫn tranh GPU với cuộc gọi đang chạy nếu dùng chung. Caller không truyền `usage`/`component` nên chi phí không vào bucket nào |

**Khối lượng gọi (theo lượt):** #1, #2, #3, #4, #7 mỗi lượt người gọi (#4 chỉ khi cần sinh); #5 khi có workflow; #8 mỗi 4 lượt; #6, #9 hiếm; #10 và #11 theo điều kiện ở bảng trên; #12 ngoài cuộc gọi. Mỗi lượt có thể có 3–4 request đồng thời tới cùng GPU.

**Đầu vào Qwen nhận (ước lượng từ các bảng trên, đơn vị token; chưa đo token thật):**

| Tác vụ | Chạy khi nào | Đầu vào ước lượng | Trúng prefix cache? | Chuyển khỏi Qwen |
|---|---|---|---|---|
| #4 Reply | Khi cần sinh | 4,1–4,4k | Lượt 1: 0; các lượt sau: 2096 token | Khó (tool calling, knowledge, streaming) |
| #1 Decision | Mọi lượt | 1,1–1,3k | **Không bao giờ** (ngắn hơn một khối 2096) | Chỉ cần đổi `DECISION_LLM_*` |
| #3 Bridge | Mỗi segment final | 0,4–0,5k | Chưa rõ | Cần sửa code (chung endpoint Reply) |
| #2 Anger | Mọi lượt | 0,15–0,25k | Chưa rõ | Cần sửa code nhỏ hoặc đổi `LOCAL_JSON_LLM_*` |
| #7 Sentiment-text | Mọi lượt (nền) | 0,1–0,15k | Chưa rõ | Chỉ cần đổi `EMOTION_TEXT_*` |
| #8 Purpose | Mỗi 4 lượt | Cả transcript, tăng theo cuộc gọi | Chưa rõ | Đổi `LOCAL_JSON_LLM_*` |
| #9 Post-call | Một lần khi kết thúc | Cả transcript + tín hiệu | Chưa rõ | Đổi `LOCAL_JSON_LLM_*` |

Năm tác vụ chạy mỗi lượt (#4, #1, #3, #2, #7) cộng lại khoảng **5,9–6,6k token** mỗi lượt; chưa tính #5, #8, #9. Vì Reply được cache 2096 token từ lượt 2, phần prefill thật của Decision lớn hơn tỷ lệ token của nó cho thấy.

---

## 4. Bậc giải pháp và thứ tự tách từng tác vụ (suy luận)

Mục tiêu là **giảm khối lượng đầu vào Qwen phải xử lý**, không phải mỗi tác vụ một model. Vì vậy chỉ chọn một vài **đích chuyển dùng chung** (mục 4.2), rồi xếp từng tác vụ vào đích rẻ nhất mà nó còn đạt. Mỗi tác vụ vẫn cần bộ đo riêng trước khi chuyển: thử giải pháp rẻ nhất trước và chỉ nâng bậc khi bậc thấp hơn không đạt. Các bậc dưới đây là khung phân loại, không phải kết luận về model.

### 4.1 Các bậc

| Bậc | Giải pháp | Tác vụ ứng viên | Căn cứ trong repo |
|---|---|---|---|
| 0 | Mã cố định, từ khóa | Một phần #5 (số điện thoại, ngày, enum); từ khóa của #11 | `postprocess.py` đã chuyển ba việc sang mã (mục 5.3): semantic match 84,6% → 87,0%, p50 250 → 144 ms |
| 1 | Bộ mã hóa nhỏ + đầu phân loại, không sinh văn bản | #1 Decision, #2 Anger, #11 Intent; có thể #7 | Đầu tuyến tính trên `multilingual-e5-small`: 9/10, p50 7,5 ms, **chỉ trên 10 ca và so với Gemini, chưa so với Qwen** (mục 5.1) |
| 2 | Model sinh nhỏ, đầu ra ngắn, có bộ kiểm tra bên ngoài | #3 Bridge, #6 Grounded reply | Bridge ≤5 từ có `generated_line_checks`; 3B + few-shot + tự sinh lại đạt 0% vi phạm, nhưng 3B thuần sai 19% ở lượt đặt lịch sâu (mục 5.2) |
| 3 | Model chính | #4 Reply | Cần tool calling, prompt ~4,3k token, knowledge, streaming (mục 7) |
| 4 | Model trích xuất chuyên dụng | #5; có thể #10 | NuExtract 2B: invention 1,57%, p50 250 ms, nhưng omission 10,81% (mục 5.3) |
| 5 | Model lớn hơn, hàng đợi, ngoài đường găng | #8, #9, #12 | Không ràng buộc độ trễ cuộc gọi; vẫn tranh GPU nếu chạy cùng lúc |

Mẫu thiết kế đáng giữ: bậc thấp **tự từ chối** khi không chắc rồi đẩy lượt lên bậc cao (đầu Decision trả `None` khi tập hợp lệ có kind an toàn mà nó không phát được; chế độ `hybrid`).

### 4.2 Đích chuyển dùng chung

Thay vì N model cho N tác vụ, chỉ có ba đích. Các đích không loại trừ nhau: một tác vụ chỉ vào đúng một đích.

| Đích | Là gì | Tác vụ có thể vào | Cách chuyển | Ràng buộc |
|---|---|---|---|---|
| **X0** | Không dùng LLM: mã cố định, từ khóa, hoặc bộ mã hóa + đầu phân loại (bậc 0–1) | #1 Decision, #2 Anger, #11 Intent, một phần #5 | Decision: `DECISION_PRODUCER_MODE=hybrid`; còn lại cần code | Phải có bộ nhãn; kind an toàn cần xử lý riêng (mục 5.1) |
| **X1** | **Một** endpoint LLM nhỏ dùng chung cho nhiều tác vụ nhẹ (bậc 2–5) | #2, #3, #6, #8, #9, #10, #11, `intent_gate`, và Decision/#7 nếu trỏ cùng URL | Đặt `LOCAL_JSON_LLM_BASE_URL/MODEL` sang endpoint mới: một biến chuyển cả họ (mục 6). `DECISION_LLM_*` và `EMOTION_TEXT_*` có biến riêng nên trỏ cùng URL; #3 cần sửa code | JSON mode, tắt thinking (mục 7); #8/#9 cần context đủ dài; một endpoint dùng chung phải chịu tải gộp, cần đo tranh chấp |
| **X2** | Làn nền: hàng đợi, GPU hoặc thời điểm khác | #8, #9, #12, có thể #7 | Cần code hàng đợi, hoặc cùng X1 nhưng trên GPU khác | Chỉ cho tác vụ không trên đường găng |
| Ở lại Qwen | Qwen3.6 | #4 Reply (và #3 cho tới khi tách endpoint) | — | Tool calling, knowledge, prompt ~4k |

Lưu ý khi dùng X1: `LOCAL_JSON_LLM_BASE_URL` còn được đọc bởi `intake/extraction.py` và `appointment_booking/extraction.py` (đường trích xuất qua LLM). Chưa xác định hai đường này có chạy trên staging hay không (nuextract là mặc định); đổi biến này sẽ chuyển cả chúng.

### 4.3 Bốn câu hỏi để xếp một tác vụ vào bậc

1. **Đầu ra có phải một nhãn trong tập đóng không?** Có thì thử bậc 1 trước.
2. **Sai thì hậu quả gì?** Anger dương tính giả khóa cuộc gọi vào hold đến hết; Decision sai kind là sai cả lượt và phải giữ kind an toàn; Sentiment sai chỉ lệch tone. Hậu quả nặng đòi ngưỡng riêng, không chỉ độ chính xác trung bình.
3. **Đầu ra có kiểm tra được bằng quy tắc không?** Có thì model nhỏ an toàn hơn (Bridge). Không thì nên giữ model lớn (Reply).
4. **Có nằm trên đường găng không?** Có thì độ trễ là tiêu chí chính; không thì chất lượng và tranh GPU là tiêu chí chính.

### 4.4 Ưu tiên chuyển theo khối lượng đầu vào

| Ưu tiên | Tác vụ | Đầu vào mỗi lượt (ước lượng) | Đích đề xuất | Lý do |
|---|---|---|---|---|
| 1 | **#1 Decision** (mục 11) | ~1,1–1,3k, mọi lượt, không trúng cache | X0 hoặc X1 | Phần lớn nhất có thể chuyển và nằm trên đường găng; có bộ nhãn, harness baseline, cấu hình riêng, chế độ `shadow` |
| 2 | #3 Bridge | ~0,4–0,5k, mỗi segment | X1 | Có bộ kiểm tra đầu ra; cần sửa code vì chung endpoint Reply |
| 3 | #2 Anger, #7 Sentiment | ~0,1–0,25k, mọi lượt | X0 hoặc X1 | Nhỏ nhưng chạy mỗi lượt và tranh GPU; #7 chỉ cần đổi env; #2 cần bộ nhãn trước (mục 8.3) |
| 4 | #8 Purpose, #9 Post-call | Tăng theo transcript | X2 (hoặc X1 nếu đủ context) | Không trên đường găng; lớn khi cuộc gọi dài |
| 5 | #10, #11, #6, #12 | Thấp, có điều kiện | Đi theo X1 | Chuyển miễn phí cùng họ `LOCAL_JSON_LLM_*` (mục 6) |
| Ở lại Qwen | #4 Reply | ~4,1–4,4k | Giữ | Khó chuyển nhất; đòn bẩy là cache và rút gọn prompt, không phải đổi model (mục 5.5) |
| Đã ngoài Qwen | #5 Extraction | nhỏ | NuExtract | Đã ở model riêng |

Thứ tự là đề xuất, sẽ xác định lại sau khi Decision có kết quả. Giảm kích thước prompt (system ~2k + knowledge ~1,7–1,9k của Reply, 12 lượt hội thoại của Decision) là đòn bẩy bổ sung chưa nghiên cứu ở đây.

### 4.5 Nhóm theo đặc tính tải (tham khảo, không còn là đơn vị làm việc)

Cách nhóm trước đây dựa trên **đặc tính tải** (kích thước vào/ra, vị trí trên đường găng, loại đầu ra), không phải kết luận về model. Giữ lại để tra cứu:

| Nhóm | Tác vụ | Đặc tính chung | Câu hỏi nghiên cứu |
|---|---|---|---|
| **A. Phân loại đầu ra tí hon trên đường găng** | #1 Decision, #2 Anger; **#11 Intent fallback** (có điều kiện) | Đầu ra 1–3 token (#11: JSON ngắn), quyết định rẻ nhưng tuần tự; độ trễ model ≈ toàn bộ chi phí (harness 0,7 ms) | Model nhỏ/độ chính xác ngang model lớn? Có cần LLM không (bộ mã hóa + đầu phân loại, ví dụ đường `local` đã có)? Quản lý "kind an toàn". Riêng #11: chỉ chạy khi từ khóa trượt, nên đo tần suất trượt trước |
| **B. Sinh ngắn trên đường găng** | #3 Bridge, #4 Reply (câu đầu), #6 Grounded (hold) | Độ trễ token đầu và tốc độ decode quyết định; cần prefix cache; reply còn cần tool calling + knowledge | Model nào cho TTFT thấp với prompt 4k? Model dày đặc nhỏ so với MoE? Bridge tách riêng được không (prompt rất ngắn, ~5 từ)? |
| **C. Trích xuất có cấu trúc** | #5 Extraction; **#10 KB step validation** (verdict JSON có điều kiện, trên đường găng, cần truy xuất KB trước) | JSON/verbatim, ưu tiên không bịa; chạy sau Decision | Đã có NuExtract 2B làm cơ sở so sánh; ứng viên khác có hơn về omission/chữ số không? Với #10: có cần LLM sinh, hay đủ một model đánh giá nhỏ? Độ tự tin do model tự báo (không có log-prob) |
| **E. Ngoài cuộc gọi** | #12 Publish wording | Không có ràng buộc độ trễ cuộc gọi; cần đúng schema, không bịa, fail-closed | Có thể chạy trên model lớn hơn hoặc ngoài giờ cao điểm để không tranh GPU với cuộc gọi? |
| **D. Phân tích nền** | #7 Sentiment-text, #8 Purpose, #9 Post-call | Không trên đường găng; có thể batch/hàng đợi/model lớn hơn; nhưng #7 vẫn tranh GPU | Có thể chuyển sang model nhỏ hơn, hoặc chạy ngoài giờ cao điểm/GPU khác? #8 và #9 dùng chung hợp đồng "purpose" |

(Suy luận) Hai ghi chú từ repo hỗ trợ nhóm A–B tách riêng: (i) nhiều tác vụ cạnh tranh một GPU làm Decision dao động 0,23–1,9 s; (ii) thử nghiệm gộp Decision vào Reply (một request) **chậm hơn** (948 so với 834 ms) và làm đổi quyết định, trong khi chạy song song hai request chỉ tiết kiệm 25–110 ms vì chờ prefill trên cùng GPU.

---

## 5. Bằng chứng sẵn có: model đã thử, kết quả đo

### 5.1 Decision

| Cấu hình | Dữ liệu | Kết quả | Nguồn |
|---|---|---|---|
| **Gemini 2.5 Flash** (thinking động, bản cũ) | 52 ca / 213 lần | **100% "behaviourally correct"**, 0 lỗi quan trọng; **p50 1998 ms**, p95 2603; ~186 token thinking cho 3 token đầu ra; nhất quán 0,87 | `E/decision-producer-model-benchmark-2026-07-24.md` |
| Gemini 2.5 Flash, thinking=64 | 41 + 52 lần | 0 hồi quy quan sát; **p50 ~1175 ms** (−41%) | như trên |
| **Gemini 2.5 Flash-Lite** | **chỉ n=1** (52 ca) | 49/52 đúng hành vi (−3), tham chiếu 17/19 (−2); **p50 773 ms**; không nêu được lớp nào giảm | như trên (chạy dở vì hết credit) |
| **Qwen3.6-35B-A3B** (prompt `current`, nhận kind) | 216 lượt tự soạn / 207 lượt thật | Mức **kind**: 203/216 (**94,0%**) tự soạn, **150/207 (72,5%)** thật. Mức **action**: **97,2%** / **81,2%** | `E/decision-baseline-2026-10-13/report.md` |
| Qwen3.6, prompt `v2` (7 caller kind) | như trên | Action: 77,3% tự soạn, 67,6% thật (**kém hơn** `current`) | như trên |
| Qwen3.6 — độ lặp lại | 52 ca × 3–5 lần | Cùng đầu ra ở **44/52 (84,6%)**; ngưỡng 90% → **không đạt (P7)**. Hai lần chạy giống hệt khác nhau ở 11/217 yêu cầu | `decision-baseline` report; `E/decision-latency-ideas-2026-10-19/RESULT.md` |
| Qwen3.6 — độ trễ thực | cuộc gọi GB10 | Decision **286–347 ms** (setup C/D); biến động theo tải 229–1899 ms | `E/gb10-latency-context-2026-10-12.md` |
| **Bộ mã hóa `multilingual-e5-small` int8 + đầu tuyến tính** | **10 ca** (tập chọn thinking-budget), đầu huấn luyện 54 lượt | **9/10**, **p50 7,5 ms** (so với 16/16 và 1002 ms của Gemini thinking=64 trên cùng tập). Zero-shot cosine: 1–5/10 | `services/voice-ai-orchestrator/assets/decision/README.md` |

Ghi chú về đầu tuyến tính (đã có trong code, `DECISION_PRODUCER_MODE=local|hybrid`):
- Đầu chỉ phát được kind đã học; **từ chối lượt (trả `None`) khi tập hợp lệ chứa kind an toàn (`reject`, `escalate`) mà đầu không phát được**; huấn luyện từ chối nếu corpus thiếu kind an toàn.
- Corpus huấn luyện được README ghi là "**791 dòng tự soạn, 0 lượt thật**". README sau đó mô tả một bản export mined 197 dòng cho v7 (`turn_corpus/mined_turns_v7.jsonl`); **chưa thấy bằng chứng đầu được fit lại** bằng dữ liệu này.
- Đầu giao kèm **đạt 0,0 trên `reject`** ở 10 ca giữ lại dù huấn luyện trên 55 dòng. README cho rằng nguyên nhân là thiếu các *họ* reject khác nhau (16 họ, mỗi họ ~4 cách diễn đạt), không phải thiếu số dòng.
- Đầu **không chỉ đọc câu nói**: nó đọc thêm các đặc trưng ngữ cảnh, lớn nhất là `workflow_active` (trọng số 27,99). Dữ liệu D1 ghi `workflow_active` sai (đổi 26 lần trong 10 cuộc gọi nhiều lượt; 18 dòng ghi `false` sau khi cuộc gọi đã route).
- **Lệch huấn luyện/chạy cho v7 vẫn mở**: `turn_features.FEATURE_NAMES` chưa cập nhật cho v7; mọi đầu fit trước v7 bị loader từ chối (chiều đầu vào 391 → 387).
- `assets/decision/` **rỗng có chủ ý trong git**: không có `model.onnx`, `tokenizer.json`, `head.npz` thì client cục bộ báo không khả dụng và mọi chế độ quay về đường từ xa.
- Con số 9/10, 7,5 ms đo trên bộ 10 ca dùng để chọn thinking budget, **so với Gemini** (16/16, p50 1002 ms); chưa so với Qwen3.6 trên 207 lượt thật.
- Offline chỉ giữ được **một** lượt agent trước đó, trong khi Producer thấy 12 lượt.

Bộ 207 lượt thật do người duyệt (đã duyệt xong 210, trừ 10 dòng loại khỏi huấn luyện vì không có bằng chứng `workflow_active`): **0 `reject`, 0 `escalate`, 0 `courtesy`**. Hai bộ gán nhãn AI độc lập cũng ra 0 `reject` trong 420 đề xuất dù `reject` hợp lệ ở cả 210 dòng. Quét 543 lượt người gọi thật chỉ có 2 lượt giống `reject`, cả hai là lượt thử của lập trình viên. Vì vậy 72,5% kind-level của Qwen trên bộ thật **không nói gì về an toàn**.

**Thử nghiệm độ trễ Decision đã làm:** thinking budget (`E/decision-thinking-budget-experiment-2026-07-26.md`), gộp Decision vào Reply, Decision làm "guide", cache prompt (mục 5.5), DG21/DG22 để giảm token đầu ra (`E/decision-latency-ideas-2026-10-19`; kết quả: ý tưởng 1 đã build; ý tưởng 3 dừng vì mọi thứ tự prompt thử đều đổi Decision nhiều hơn nhiễu).

### 5.2 Reply, Bridge, Grounded reply (sinh văn bản)

| Cấu hình | Kết quả | Nguồn |
|---|---|---|
| Ministral-3-8B FP8 (bản cũ) vs **Qwen3.6-35B-A3B NVFP4** làm reply LLM | Thời gian tới câu đầu **1704 → 831 ms** (trung vị); ttfa server 4316 → 2810 ms; Qwen "trả lời từ knowledge và nói 'I do not have' khi không có nguồn"; **Ministral bịa chi tiết ở 2/4 câu** (mẫu rất nhỏ: một cuộc gọi mỗi model) | `E/llm-stage-latency-2026-10-11/report.md` |
| `llama3.2:3b` (4-bit, RTX 5060 Ti) | Sinh trọn câu trả lời **382 ms p50 / 478 p90** (prefix lạnh); ttft 164; câu đầu 247. **Nhưng thông báo "đã đặt lịch" sai ở 19% lượt đặt lịch sâu**; `llama3.1:8b`: 4,8% | `E/wave2-local-generation-results-2026-08-22.md` |
| 3B + "few-shot + tự sinh lại khi vi phạm" (S7) | **0% vi phạm**, p90 470 ms, lần gọi thứ hai chỉ ở 2,4% lượt | `E/wave2e-invention-prevention-2026-08-24.md` |
| Đưa quyết định vào cùng request reply | Chậm hơn (948 vs 834 ms), đổi quyết định ở 6/15 lượt thật | `E/gb10-latency-context-2026-10-12.md` mục 5 |
| Bridge — sinh bằng reply LLM | Token đầu **170–360 ms**; vấn đề chính là TTS của bản clone (0,7–1,0 s) chứ không phải model; sau "late take" người gọi chờ ~1,2 s thay vì ~2,2 s | `E/bridge-late-2026-10-15/experiment.md` |
| Ảnh hưởng prompt | Reply: ~⅔ thời gian là đọc prompt 4,3k token, ~⅓ là decode câu đầu; chuyển dòng metadata người gọi xuống cuối giảm token đầu lượt 1 từ 596 → 310 ms | `E/gb10-latency-context-2026-10-12.md` |
| TTS clone | OmniVoice clone thêm 1,0–1,8 s ttfa mỗi lượt; Kokoro gần như không | như trên |

Grounded reply (#6) và anger judge (#2): **không thấy** benchmark model riêng.

### 5.3 Trích xuất trường (#5)

Bộ vàng 246 dòng (1145 cơ hội bịa, 111 nhãn trường không rỗng), cùng hợp đồng triển khai:

| | Gemini 3.5-flash | NuExtract-2.0-2B Q4_K_M | NuExtract + A10 (hậu xử lý cố định) |
|---|---|---|---|
| Invention | 1,66% | **1,57%** | 1,57% |
| Omission | **3,60%** | 10,81% | 6,31% (gate: thất bại 0,71 điểm) |
| Semantic match | **88,6%** | 84,6% | 87,0% (đạt) |
| Vi phạm enum | 0 | 2 | 0 |
| p50 / p95 | 982 / 1355 ms | **250 / 331 ms** | **144 / 191 ms** (RTX 5060 Ti) |

- Model cục bộ **thắng ở không bịa** (lớp A, B, G) và **thua ở bắt giá trị** (lớp C, H chữ số, E, F).
- Điểm mạnh yếu thực tế: câu trả lời ngắn không dấu chấm → NuExtract trả `null`; sửa bằng prompt "đối thoại" lần hai (tìm đúng trường 65 → 78/83, invention 0,70% → 0,96%).
- **Ràng buộc đã đo**: đưa `extraction_hint` có ví dụ vào template làm model **bịa** (2/9); ép ngữ pháp toàn đối tượng gây bịa 26/26.
- Ba việc được chuyển sang mã cố định (`postprocess.py`): khôi phục số điện thoại, quy đổi ngày tương đối theo ngày gọi, ánh xạ enum.

### 5.4 Cảm xúc (#7)

- Kênh audio: SenseVoiceSmall (~50 ms trên GPU); trên CREMA-D (giọng diễn): giận ≥0,5 ở 9/10, neutral không vượt 0,30 → ngưỡng giận ~0,4; **cần kiểm lại trên cuộc gọi thật**. `frustrated` và `confused` gần như bằng 0 ở kênh audio, chỉ đến từ kênh text.
- Kênh text trên LLM: chạy trên **model trả lời** tốt hơn đặt trên model khác (Ministral cho text-emotion trong khi Qwen trả lời chậm hơn: 1414 vs 831 ms tới câu đầu); text-emotion tốn 204 ms (với Kokoro) – 640 ms ttfa trên Qwen.
- Không thấy bộ nhãn cho kênh text.

### 5.5 Bộ nhớ đệm prompt và tác động tới việc tách model

- Reply lượt 1 không trúng cache vì dòng metadata người gọi nằm giữa system prompt; chuyển xuống cuối cắt token đầu 596 → 310 ms.
- Decision 1,1–1,3k token < một khối cache (2096) → không bao giờ trúng cache trên Qwen3.6 (model lai). **Model khác có thể có khối cache nhỏ hơn**: một báo cáo ước lượng `--mamba-block-size` mịn hơn tiết kiệm ~130 ms/lượt (ước lượng, chưa thử).
- (Suy luận) Đặt Decision lên model riêng ngắn hơn khối cache của Qwen có thể có lợi hoặc không; chưa có số đo.

---

## 6. Mức độ tách rời của cấu hình hiện tại (cái gì đổi được bằng `.env`)

| Tác vụ | Biến cấu hình riêng | Dùng chung với | Cần sửa code để tách? |
|---|---|---|---|
| 1 Decision | `DECISION_LLM_BASE_URL`, `DECISION_LLM_MODEL`, `DECISION_LLM_TIMEOUT_S` (3 s); `DECISION_PRODUCER_MODE` (`gemini/shadow/hybrid/local/llm`); `DECISION_MODEL_PATH` (đầu cục bộ) | — | **Không** — đã tách. Đổi model chỉ cần đổi `DECISION_LLM_BASE_URL/MODEL` (đường A, mục 11). Có sẵn chế độ `shadow` để so model song song, và `hybrid` (đường B cần cấp artefact vì `assets/decision/` rỗng) |
| 4 Reply | `LLM_PROVIDER`, `LOCAL_LLM_BASE_URL`, `LOCAL_LLM_MODEL`, `LOCAL_LLM_TOOL_CALLING`… | **Bridge** (cùng endpoint, lấy từ cùng biến) | Không cho Reply; **có** nếu muốn model riêng cho Bridge |
| 3 Bridge | *(không có)* | Reply | **Có** — `generate_bridge` nhận `base_url`/`model` của reply LLM |
| 5 Extraction | `EXTRACTION_PROVIDER`, `LOCAL_EXTRACTION_*`, `EXTRACTION_QWEN_*` | — | Không — đã tách |
| 7 Sentiment-text | `EMOTION_TEXT_BASE_URL`, `EMOTION_TEXT_MODEL` | — | Không — đã tách |
| 2 Anger judge | `ANGER_JUDGE_TIMEOUT_S` (chỉ timeout) | `LOCAL_JSON_LLM_*` | **Có** |
| 6 Grounded reply | `GROUNDED_REPLY_TIMEOUT_S` | `LOCAL_JSON_LLM_*` | **Có** |
| 8 Call purpose | `CALLER_PURPOSE_MODEL` (chỉ khi dùng Gemini) | `LOCAL_JSON_LLM_*` | **Có** |
| 9 Post-call | `CONVERSATION_INTELLIGENCE_MODEL` (chỉ khi dùng Gemini) | `LOCAL_JSON_LLM_*` | **Có** |
| 10 KB step validation | — (hằng số `LLM_TIMEOUT_S` 4 s, `RETRIEVAL_TIMEOUT_S` 3 s trong code) | `LOCAL_JSON_LLM_*` | **Có** |
| 11 Intent fallback | — (timeout 4 s đặt cứng ở `LLMIntentClassifier`) | `LOCAL_JSON_LLM_*` | **Có** |
| 12 Publish wording | — (timeout 20 s đặt cứng ở `_question_model`) | `LOCAL_JSON_LLM_*` | **Có** |

Ghi chú về chi phí sửa code: hàm `local_json_llm.complete(...)` **đã nhận tham số `base_url`** (mặc định đọc `LOCAL_JSON_LLM_BASE_URL`), nên tách một tác vụ ra endpoint riêng thường chỉ cần truyền `base_url` và tên model ở điểm gọi. Tên model được tra theo URL (`model_name(url)`), nên mỗi endpoint cần phục vụ đúng model mong muốn. (Chưa xác minh việc này cho mọi điểm gọi.)

**Một biến chuyển cả họ.** Các điểm gọi đọc `LOCAL_JSON_LLM_BASE_URL` (qua `local_json_llm.complete` hoặc `GeminiCaller` khi biến này được đặt): `ai/grounded_reply.py` (#2, #6), `ai/gemini_json.py` (dùng bởi #8, #9, #10, #11, `knowledge/intent_gate.py`, `intake/realizer.py`, `management/api.py`), cùng `intake/extraction.py` và `appointment_booking/extraction.py`. Nên đổi biến này là chuyển đồng loạt chứ không riêng một tác vụ. Chưa xác minh từng bước định tuyến trong `gemini_json.py`.

Khi dùng Gemini (đường cũ): các tác vụ #8, #9 đi qua `GeminiCaller`; chỉ thành "cục bộ" khi `LOCAL_JSON_LLM_BASE_URL` được đặt.

---

## 7. Danh sách kiểm tra tương thích cho model ứng viên

Tính năng code đang dùng trên đường gọi; model/máy chủ ứng viên phải có hoặc bị loại tác vụ tương ứng:

| Tính năng | Dùng bởi | Ghi chú |
|---|---|---|
| Tắt chế độ thinking (`chat_template_kwargs: {enable_thinking: false}`) | Decision, Bridge, Sentiment-text, mọi gọi `local_json_llm` | Qwen3 mặc định "nghĩ" hàng trăm token; tài liệu ghi Mistral tokenizer trả HTTP 400 với khóa này (code thử lại không có khóa) |
| Streaming + `stream_options.include_usage` | Reply, Bridge | Cần để TTS bắt đầu ở câu đầu và để đếm token |
| Tool calling (`tools`, `tool_choice: auto`, parser `qwen3_xml` hoặc `mistral`) | Reply (workflow/domain/KB tools) | Máy chủ phải bật `--enable-auto-tool-choice` và parser đúng; `LLM_PROVIDER=local` + tools mà không bật → code **từ chối khởi tạo** |
| `continue_final_message=true`, `add_generation_prompt=false` | Reply khi có bridge (viết tiếp câu bridge đã phát) | Cần hỗ trợ tiếp tục message `assistant` cuối |
| `structured_outputs: {regex}` | Decision ngữ pháp v2 (hiện không bật) | Chỉ vLLM bản hỗ trợ |
| `response_format: json_object` / `json_schema` | Sentiment-text, `local_json_llm` (purpose, post-call…) | Ghi chú code: ràng buộc schema thuần có thể làm Qwen "đánh nhau với ngữ pháp" → code nói rõ schema trong prompt |
| Văn bản thô `/v1/completions` với khung `<|im_start|>` | NuExtract | Chỉ cho NuExtract, không áp template chat lần hai |
| Prefix caching | Reply, Decision (nhóm A/B) | Quan trọng với prompt 4k; hành vi phụ thuộc kiến trúc model (hybrid vs dense) |
| Nhiệt độ 0 | Decision, anger, sentiment, purpose | Không bảo đảm lặp lại hoàn toàn (xem 5.1) |
| Chuyển tên model tự động | Decision, Sentiment-text | Nếu `DECISION_LLM_MODEL`/`EMOTION_TEXT_MODEL` rỗng, code lấy model đầu tiên từ `/v1/models` |

---

## 8. Tiêu chí đánh giá và công cụ đo có thể tái dùng

### 8.1 Các cổng (gate) đã dùng, có thể mượn làm khung

- **Extraction** (so với Gemini): invention ≤ Gemini; unparseable = 0; p50 < 500 ms, p99 < 1500 ms; semantic ≥ Gemini − 3 điểm; omission ≤ Gemini + 2 điểm; enum vi phạm = 0 (`E/extraction-goldset/RESULTS.md`).
- **Decision** (`E/decision-baseline-2026-10-13/plan.md`, tiêu chí P1–P9): kind/action đúng; **nhất quán ≥ 90%** qua các lần lặp; truyền tải không lỗi; tái lập số báo cáo trước (P6). Các lớp "quan trọng" trong bench: withdrawal, resume, capability switching, knowledge interruption, protection khi capability đã xong, roster rỗng, tham chiếu bị cấm.
- **Reply wording** (`E/wave2e`): tỷ lệ "thông báo hành động chưa xảy ra" (breach), độ liên quan, tỷ lệ gọi lại lần hai, p90.
- **Độ trễ**: `technical_ttfa_ms`, `user_perceived_ttfa_ms`, `llm_first_segment_ms`, `decision_ms` trong dòng log `[turn.timing]`; phải **chạy A-B-A** vì tải GPU thay đổi giữa các lần.

### 8.2 Bộ dữ liệu và công cụ có sẵn trong repo

| Tài nguyên | Nội dung | Dùng cho tác vụ |
|---|---|---|
| `E/decision_corpus/cases.jsonl` (+`SPLIT.json`) | **207 lượt thật** được người duyệt, chia theo phiên; giữ lại 3 capability (`accident_intake`, `lead_capture`, `dispute_intake`) để thử chưa thấy | #1 (và tham khảo #2) |
| `services/.../spikes/decision_thinking_bench.py` + `E/decision-model-bench.py` | 52 ca / 18 lớp tự soạn (có lớp quan trọng, STT vỡ từ cuộc gọi thật) | #1 |
| `E/decision_corpus/counterfactual_probe.jsonl` (150), `ablation_v9/`, `staged_v8/v9_*` | Các thăm dò phản chứng và ablation | #1 |
| `E/extraction-goldset/` (246 dòng + not-applicable + ambiguous), `extraction-demoset/`, `extraction-corpus/` | Bộ vàng trích xuất, 3.531 lời gọi thật được khai thác | #5 |
| `E/decision-latency-ideas-2026-10-19/tone_cases.json` | 40 câu người gọi để thử giọng mở đầu (nhãn do tác giả, không phải người) | #3 (chỉ tham khảo) |
| 18 prompt Decision thật (2 cuộc gọi, capture 2026-10-18) + `replay_cache.py` | Phát lại để so cache/đầu ra | #1, #4 |
| `.claude/skills/gb10-latency-ab` (`run_experiment.sh`, `analyze.py`, `breakdown.py`, `llm_probe.py`, `capture_llm_requests.sh`, `replay_prompts.py`) | Chạy A/B trên staging với đảo ngược cấu hình bằng `SH_*`, ghi lại request LLM thật để phát lại | #1, #3, #4, #7 |
| `services/.../spikes/wave2_local_generation.py`, `wave2e_invention_prevention.py`, `wave5_local_wording_screen.py`, `decision_model_bench.py` | Harness đo sinh văn bản cục bộ / bịa / sàng lọc wording | #3, #4, #6 |
| `E/bridge-late-2026-10-15/`, `E/bridge-same-turn-text-2026-10-15/` | Thử nghiệm bridge thật (`gb10.md`, `regression.md`) | #3 |
| `services/.../spikes/intent_eval/` | Đánh giá phân loại intent | #1 (liên quan) |

### 8.3 Dữ liệu **chưa thấy** (cần tạo nếu muốn so sánh model)

| Tác vụ | Thiếu gì |
|---|---|
| #1 Decision | Kind an toàn trên lượt thật: bộ 207 dòng có 0 `reject`/`escalate`/`courtesy`, nên an toàn chỉ kiểm bằng dữ liệu tự soạn. Lịch sử ≥2 lượt agent không khôi phục được offline (cần thay đổi logging). Nhãn `workflow_active` của D1 bị lỗi. Còn 10 dòng `workflow_active: null` chỉ dùng được làm bộ kiểm tra bất biến |
| #2 Anger judge | Nhãn "giận/không giận" trên lượt thật; không thấy báo cáo đo độ chính xác/dương tính giả (chỉ có docstring về STT cắt câu giận) |
| #3 Bridge | Bộ đánh giá chất lượng/đa dạng/tỷ lệ bị bộ kiểm tra loại ở quy mô lớn (chỉ có thử nghiệm vài lượt) |
| #4 Reply | Bộ so sánh nhiều model trên cùng prompt thật 4k token và đúng tool calling (có so Ministral/Qwen ở mẫu rất nhỏ) |
| #6 Grounded reply | Không thấy đo riêng |
| #7 Sentiment-text | Không có nhãn cảm xúc thật cho kênh text; kênh audio chỉ kiểm trên giọng diễn |
| #8 Purpose / #9 Post-call | Không thấy bộ so sánh chất lượng giữa Gemini và Qwen |
| #10 KB step validation, #11 Intent fallback, #12 Publish wording | Không tìm thấy benchmark riêng (chỉ tìm bằng từ khóa trong code; chưa tìm trong `E/`). Cũng chưa có số liệu tần suất chạy: bao nhiêu workflow thật có bước `knowledge_base`, bao nhiêu lượt trượt từ khóa và rơi vào #11 |
| Khối lượng token thật mỗi tác vụ | Chỉ có ước lượng (mục 3). Cần số thật để biết chuyển tác vụ nào giảm tải Qwen nhiều nhất. Có cơ chế `UsageAccumulator` với bucket `per_component` (ví dụ #11 dùng `intent_router`); chưa kiểm tra có đo được token từng tác vụ không, và #12 chưa được gán bucket |

---

## 9. Hạn chế của dữ liệu và điều cần cẩn trọng

- **Tải GPU chung**: nhiều kết quả bị trộn bởi stack khác; chỉ so sánh trong cùng lần chạy hoặc theo A-B-A.
- **Phần cứng khác nhau**: số wave 2 (RTX 5060 Ti/3060, Ollama) không so sánh trực tiếp được với GB10 (vLLM, NVFP4).
- **Mẫu nhỏ**: Flash-Lite cho Decision chỉ n=1; so sánh Ministral/Qwen về bịa chi tiết dựa trên 4 câu; đầu tuyến tính chỉ 10 ca.
- **Nhãn của tác giả**: các nhãn tone (bridge), nhiều ca tự soạn là của tác giả, không phải người duyệt độc lập. Corpus Decision có "cases.jsonl" là người duyệt, nhưng nhãn khởi tạo từ AI (`ai_labels.jsonl`) rồi người duyệt.
- **ASR môi trường dev**: bộ vàng trích xuất dùng ASR dev; tác giả báo con số chỉ mang tính định hướng.
- **Nhiễu do chính model**: Decision ở nhiệt độ 0 khác giữa hai lần chạy 11/217 yêu cầu; chênh dưới ~11 dòng không phải kết quả.
- **Bản ghi hết credit**: đo Gemini Flash-Lite gián đoạn do hết credit prepaid.
- **Khối lượng đầu vào mới là ước lượng**: các số 5,9–6,6k token mỗi lượt và tỷ lệ từng tác vụ (mục 3) tính từ prompt trong code và vài báo cáo; chỉ Reply và Decision có số đo, còn Bridge, Anger, Sentiment ghi "ước lượng". Chưa biết phần nào trúng prefix cache ngoài Reply và Decision.
- **Harness baseline Decision nằm ngoài repo**: báo cáo `decision-baseline-2026-10-13` ghi harness và bằng chứng thô ở scratchpad của phiên đo. Phải xác nhận còn dùng được trước khi chạy lại với model ứng viên (mục 11).
- **Hai giả định trong nhãn Decision**: `capability_available` được *giả định* (lô mined đều chạy với roster không rỗng; ~1,4% lưu lượng thật thì không, không có trong bộ này). Và tên workflow (`reference`) không phải mục tiêu của Decision, nên chỉ so kind và hành động.
- **Tìm kiếm của tôi có thể sót**: nhận định "không thấy benchmark" dựa trên tìm từ khóa trong `E/` (hơn 2.300 file); chưa đọc từng file.
- **Tác vụ #10–#12 mới đọc từ code**: chưa trace đầy đủ ai gọi `_run_routed_turn` trong Driver production (suy từ comment `routing_engine.py:229`); `intake/realizer.py` (LLM diễn đạt lại câu hỏi) và `knowledge/intent_gate.py` (ANSWER/KNOWLEDGE/AMBIGUOUS, timeout 4 s) cũng dùng `GeminiCaller` nhưng chưa được đưa vào bảng; realizer ghi là "Pilot only".
- **Chưa xác minh trên staging hiện tại**: cấu hình hiệu lực và nhãn model lấy từ `docker compose config` (mục 18 của `STAGING_OVERVIEW_VI.md`), không phải từ container đang chạy.

---

## 10. Câu hỏi nghiên cứu gợi ý (suy luận)

Hướng hiện tại là chuyển bớt tác vụ khỏi Qwen sang vài đích dùng chung (mục 4), nên mỗi câu hỏi được trả lời theo từng tác vụ nhưng kết quả được tổng hợp theo đích. Decision làm trước (mục 11). Thứ tự của các câu sau câu 1 chưa phản ánh thứ tự làm.

1. **Decision (làm trước):** Model LLM khác (đường A) có đạt ngang Qwen3.6 trên 216 lượt tự soạn và 207 lượt thật không (kind 94,0% / 72,5%, hành động 97,2% / 81,2%)? Mức nhất quán có đạt ≥ 90% (Qwen hiện 84,6%)? Có lỗi nào ở kind an toàn mà bộ 207 dòng không bắt được? Hiệu quả khi chạy cùng GPU? Đầu cục bộ (đường B) chỉ xét sau khi có thêm họ `reject` và sửa `workflow_active`. **Anger** là tác vụ riêng, xem câu 12.
2. **Bridge:** Có cần LLM không, hay đủ model rất nhỏ (<2B) vì đầu ra ≤5 từ và có bộ kiểm tra? Độ trễ token đầu <170 ms có đạt không? So với phương án "pool lời mở đầu dựng sẵn" đã thử (tiết kiệm ~45–100 ms, chưa build).
3. **Reply:** Model nào giữ được tool calling, bám knowledge, không bịa, và cho TTFT thấp với prompt 4k token? Dense nhỏ so với MoE; ảnh hưởng đến prefix cache.
4. **Extraction:** Có model nào hơn NuExtract 2B về omission/chữ số mà vẫn giữ invention và độ trễ? Phần nào nên ở lại mã cố định (như A10)?
5. **Nhóm nền (#7–#9):** Có thể gom về một model "phân tích" rẻ hơn, hoặc chuyển ra GPU/giờ khác để không tranh chấp với đường găng?
6. **Cấu trúc triển khai:** Số model đồng thời mà GPU hiện tại chịu được (VRAM, băng thông) trước khi lợi ích tách bị triệt tiêu bởi tranh chấp?
7. **Dữ liệu:** Cần xây bộ nhãn cho #2, #3, #7, #8, #9, #10, #11 trước khi so sánh (mục 8.3).
8. **KB step validation (#10):** Tần suất thật: bao nhiêu workflow đã publish có bước `knowledge_base`, và bao nhiêu lượt gọi thực sự chạy tác vụ này? Nếu gần bằng không thì không đáng tối ưu. Nếu có, độ trễ chủ yếu nằm ở truy xuất KB (≤3 s) hay ở LLM (≤4 s)? Một model đánh giá nhỏ có cho verdict `valid/invalid/unclear` khớp với Qwen không? Độ tự tin do model tự báo (không có log-prob) có đủ tin cậy ở ngưỡng mặc định 0,5 không, hay nên bỏ confidence và chỉ giữ verdict? Có thể che độ trễ này khi lượt đang suy đoán không, vì hiện nay nó bỏ lượt suy đoán?
9. **Intent fallback (#11):** Tỷ lệ lượt rơi vào LLM sau khi từ khóa trượt là bao nhiêu, và độ trễ `phases.routing` trong các lượt đó là bao nhiêu? Có thể mở rộng từ khóa hoặc dùng bộ mã hóa + đầu phân loại (như đường `local` của Decision) thay cho LLM không? Cần đo độ chính xác trên roster workflow thật, kể cả ca ngoài roster (phải trả `UNKNOWN_INTENT`, không route nhầm). Ngưỡng tin cậy cao/thấp hiện dùng cho `decide(...)` có giữ nguyên khi đổi model không? Có nên gộp với Decision (cùng bài toán "người gọi muốn gì") hay tách riêng?
10. **Publish wording (#12):** Vì không nằm trên đường găng, có thể dùng model lớn hơn hoặc chạy ngoài giờ cao điểm để không tranh GPU với cuộc gọi không? Tiêu chí chất lượng là gì: đúng schema `list[str]`, không bịa giá trị, giữ ổn định giữa các lần publish (do `previous_wording` bảo đảm tái dùng)? Fail-closed khi dịch vụ lỗi có chấp nhận được với model tách riêng, hay cần model dự phòng? Chi phí hiện không được gán vào bucket nào (thiếu `usage`/`component`); có cần sửa trước khi so sánh chi phí giữa các model không?
11. **Phạm vi:** Nên có đưa `knowledge/intent_gate.py` (ANSWER/KNOWLEDGE/AMBIGUOUS, timeout 4 s) và `intake/realizer.py` (pilot) vào danh sách tác vụ nghiên cứu không? Cần trace chúng giống #10–#12 trước khi quyết định (mục 9).
12. **Anger judge (#2), sau Decision:** Cần bộ nhãn "giận/không giận" trên lượt thật trước (mục 8.3). Dương tính giả khóa cuộc gọi vào hold đến hết, nên tỷ lệ dương tính giả chấp nhận được là bao nhiêu, và ngưỡng nào của text-emotion (hiện 0,8) hợp với model mới? Bộ mã hóa + đầu phân loại có đủ cho bài toán nhị phân này không?
13. **Tổng khối lượng trên Qwen và đích dùng chung:** Chuyển Decision, Anger, Sentiment, Bridge khỏi Qwen giảm đầu vào mỗi lượt từ khoảng 5,9–6,6k xuống còn khoảng bao nhiêu (số thật, không phải ước lượng)? Một endpoint X1 gộp nhiều tác vụ có tái tạo đúng tranh chấp GPU mà ta muốn tránh không, nhất là khi X1 cùng GPU với Qwen? Đích X1 nên có bao nhiêu tác vụ thì lợi ích giảm tải còn lớn hơn chi phí tranh chấp?

---

## 11. Kế hoạch tách Decision khỏi Qwen (bước 1) (suy luận)

> **Trạng thái:** đề xuất, chưa thực hiện. **Chưa chọn model ứng viên** và chưa biết GPU/VRAM dành cho nó (mục 11.5). Không phép đo nào được chạy lại cho mục này; số liệu lấy từ `E/decision-baseline-2026-10-13/report.md` và hai README ở mục 12.

### 11.1 Mục tiêu và mốc so sánh

Decision là tác vụ đầu tiên trong kế hoạch chuyển bớt tác vụ khỏi Qwen (mục 4.4), vì đây là phần đầu vào lớn nhất có thể chuyển (~1,1–1,3k token mỗi lượt, không trúng cache). Ở bước này chỉ Decision đổi, các tác vụ còn lại giữ nguyên. Đích có thể là endpoint nhỏ dùng chung với các tác vụ JSON (X1) hoặc đầu cục bộ (X0). Mốc hiện tại của Qwen3.6 (prompt `current`, nhận kind):

| Chỉ số | Tự soạn (216 lượt) | Thật (207 lượt) |
|---|---|---|
| Kind đúng | 203/216 (94,0%) | 150/207 (72,5%) |
| Hành động đúng | 210/216 (97,2%) | 168/207 (81,2%) |
| Lặp lại | 44/52 ca giống nhau (84,6%); ngưỡng P7 là 90% → không đạt | |
| Độ trễ trên cuộc gọi GB10 | 286–347 ms, biến động 229–1899 ms theo tải | |

### 11.2 Hai đường

| | A. Model LLM khác cho Decision | B. Đầu cục bộ (`hybrid`) |
|---|---|---|
| Cách làm | Đổi `DECISION_LLM_BASE_URL/MODEL` | Cấp `model.onnx`, `tokenizer.json`, fit `head.npz`, bật `hybrid` |
| Sửa code | Không | Không, nhưng phải sửa lệch `FEATURE_NAMES` v7 |
| Bộ đo | Harness baseline: 216 + 207 lượt | Chưa đủ (mục 11.4) |
| Rủi ro an toàn | Thấp, đảo ngược dễ | Cao ở kind an toàn |
| Độ trễ mong đợi | Phụ thuộc model, chưa đo | ~ms cho lượt đầu tự tin (số 7,5 ms chưa so với Qwen) |

Đề xuất: **làm đường A trước**, vì bộ đo đã có và đổi ngược dễ. Đường B chỉ bắt đầu khi có thêm họ `reject` và hết lỗi `workflow_active`.

### 11.3 Quy trình đường A

1. Xác nhận harness baseline còn dùng được (mục 9). Nếu không, dựng lại từ `decision_thinking_bench.py` và `E/decision_corpus/cases.jsonl`.
2. Kiểm tra tương thích (mục 7): tắt thinking, `max_tokens` 32, đầu ra một dòng `kind` hoặc `kind|ref`.
3. Chạy `current` với model ứng viên trên cả hai bộ, mỗi ca lặp lại như baseline (3 hoặc 5 lần).
4. Áp các cổng (11.4). Báo cáo **theo từng kind × từng ô ngữ cảnh**, không dùng một trung bình.
5. Đo độ trễ theo A-B-A trên cùng GPU (mục 8.1).
6. Chạy `DECISION_PRODUCER_MODE=shadow` trên cuộc gọi thật trước khi chuyển.

### 11.4 Cổng chấp nhận (đề xuất)

Mượn từ P1–P9 của baseline và từ cổng an toàn trong README corpus:

- Kind và hành động **không thấp hơn Qwen** ở cả hai bộ; chênh dưới ~11 dòng không tính là kết quả (nhiễu của chính Qwen).
- Lặp lại ≥ 90% (P7).
- **Cổng an toàn chặn tuyệt đối:** không chọn kind ngoài tập hợp lệ; không mang `reference` không phải do runtime xác định; không phát `reject`/`escalate` sai.
- Không hồi quy ở các lớp quan trọng: withdrawal, resume, capability switching, knowledge interruption, protection khi capability đã xong, roster rỗng, tham chiếu bị cấm.
- Truyền tải không lỗi (HTTP 200 trên mọi yêu cầu).

**Khoảng trống của cổng an toàn:** bộ 207 dòng thật không có `reject`/`escalate`/`courtesy`, nên cổng này chỉ kiểm được bằng 52 ca tự soạn. Cần bổ sung ca an toàn trước khi tin vào kết quả an toàn (và README cảnh báo không tạo dữ liệu trông như lượt thật).

### 11.5 Thông tin còn thiếu

- **Model ứng viên.** Chưa chọn, chưa nghiên cứu ngoài repo.
- **Dùng chung đích với tác vụ khác không?** Nếu Decision và các tác vụ JSON cùng một endpoint X1, ứng viên phải qua danh sách tương thích của cả hai (mục 7) và chịu tải gộp. Nếu tách riêng thì lại thành mỗi tác vụ một model, trái hướng hiện tại.
- **GPU và VRAM** dành cho ứng viên: chạy chung GB10 với Qwen hay GPU khác? Ghi chú compose nói vLLM dùng chung giữ ~60% GPU.
- **Prefix cache:** Decision (1,1–1,3k token) ngắn hơn một khối cache của Qwen (2096), nên không bao giờ trúng cache; model dense có thể khác (mục 5.5, chưa đo).
- **Harness baseline** nằm ngoài repo (mục 9).

### 11.6 Việc cần làm trước nếu theo đường B

1. Thêm các **họ** `reject` mới (README: nhiều họ khác nhau quan trọng hơn nhiều dòng cho cùng một họ).
2. Sửa nguồn `workflow_active` ở D1 (đây là lỗi runtime độc lập với dữ liệu).
3. Cập nhật `turn_features.FEATURE_NAMES` cho v7 để hết lệch huấn luyện/chạy, rồi fit lại đầu.
4. So đầu với **Qwen** (không chỉ Gemini) trên 207 lượt thật và 52 ca tự soạn.

---

## 12. Nguồn đã dùng

| Chủ đề | Tệp |
|---|---|
| Benchmark Decision (Gemini) | `E/decision-producer-model-benchmark-2026-07-24.md` (+ `.json`) |
| Baseline Decision trên Qwen3.6 | `E/decision-baseline-2026-10-13/report.md`, `plan.md` |
| Corpus Decision | `E/decision_corpus/README.md`, `cases.jsonl`, `SPLIT.json` |
| Độ trễ Decision | `E/decision-latency-ideas-2026-10-19/RESULT.md`, `E/decision-thinking-budget-experiment-2026-07-26.md` |
| Đầu phân loại cục bộ (giới hạn, kind an toàn, artefact) | `services/voice-ai-orchestrator/assets/decision/README.md` |
| Corpus Decision: 0 reject trong dữ liệu thật, `workflow_active`, cổng an toàn | `E/decision_corpus/README.md` (mục 5b–5f, 6, 7, 8) |
| Độ trễ LLM stage, so Ministral/Qwen | `E/llm-stage-latency-2026-10-11/report.md`, `context-note.md` |
| Tổng hợp độ trễ GB10 | `E/gb10-latency-context-2026-10-12.md` |
| Sinh cục bộ model nhỏ | `E/wave2-local-generation-results-2026-08-22.md`, `E/wave2e-invention-prevention-2026-08-24.md` |
| GPU placement | `docs/archive/docs/latency-architecture/runbook-gpu-placement.md` |
| Extraction | `E/extraction-goldset/RESULTS.md`, `E/nuextract-asked-field-2026-10-19/RESULT.md`, `E/extraction-wait-2026-10-19/RESULT.md` |
| Bridge | `E/bridge-late-2026-10-15/experiment.md` |
| Cảm xúc | `O/ai/emotion_sensevoice.py` (docstring), `E/llm-stage-latency-2026-10-11/report.md` |
| Cấu hình/endpoint | `selfhosted-ai/docker-compose.connect.yml`, `selfhosted-ai/llm/docker-compose.vllm.yml`, `O/ai/local_json_llm.py`, `O/ai/bridge.py` |
