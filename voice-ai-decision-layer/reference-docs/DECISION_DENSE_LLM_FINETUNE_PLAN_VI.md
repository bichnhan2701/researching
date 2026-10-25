# Decision layer: phân tích và kế hoạch cho hướng 2 (Dense LLM + fine-tuning)

Ngày: 2026-10-23. Phạm vi: chỉ Decision plane (chọn kind và reference cho mỗi lượt hội thoại).

Cách đọc tài liệu này:
- Các số đo lấy từ báo cáo trong repo (nguồn ở mục 9). Tôi không tự đo lại số nào.
- Thông tin về các model chuyên biệt lấy từ `docs/decision_specialist_models_verification_vi.md`. Tôi chưa kiểm tra lại các nguồn bên ngoài mà tài liệu đó trích dẫn.
- Những chỗ ghi "ước lượng" hoặc "chưa xác minh" là phán đoán, chưa có số đo.
- Đường dẫn tính từ `staging/` nếu không ghi khác.

Trạng thái: chưa chọn model dense cụ thể nào. Đã chọn hướng (dense LLM + fine-tuning), còn model nào do bước 2 và 3 quyết định bằng số đo.

---

## 1. Hiện trạng

### 1.1 Decision đang được thực hiện thế nào

Hệ thống chạy ở chế độ self-hosted, `DECISION_PRODUCER_MODE=llm` (mặc định của `selfhosted-ai/docker-compose.connect.yml`). Decision do Qwen3.6 làm qua `OpenAICompatibleProducerClient`
(`services/voice-ai-orchestrator/src/voice_ai_orchestrator/conversation_orchestrator/decision/producer_client_openai.py`).

| Mục | Giá trị |
|---|---|
| Endpoint | `POST {DECISION_LLM_BASE_URL}/v1/chat/completions` (mặc định `http://vllm_chat_qwen_3_6:8000`) |
| Model | `qwen3.6-35b-a3b` (NVFP4, 35B tổng, khoảng 3B active), vLLM 0.30.0 trên GB10 |
| Tham số | `temperature=0`, `enable_thinking=false`, timeout 3 giây, không retry |
| Khi lỗi | trả `None`, lượt đó rơi vào FALLBACK |
| Input | một message user: persona, tối đa 12 lượt hội thoại, known facts, danh sách kind được phép, danh sách capability. Prompt khoảng 1.1–1.3k token. Dòng hiện tại gắn nhãn `Caller:` |
| Đường grammar 1.3.0 | thêm `_SYSTEM_NOTE`, `max_tokens=32`, output text tự do dạng `kind\|ref` |
| Đường grammar 2.0.0 | 7 kind (`social, request, reply, question, stop, human, unclear`), `max_tokens=64`, ràng buộc bằng regex (`structured_outputs`, `dg7_pattern`) nên format không sai. Spec 1.4.0 (DG21): nếu chỉ có một capability thì model chỉ cần viết `request` |
| Output thực tế | khoảng 2 token |
| Độ trễ Decision | median 286–347 ms (câu trả lời của agent chờ Decision xong) |

Chưa kiểm tra `DECISION_GRAMMAR` trong deployment đang chạy là 1.3.0 hay 2.0.0.

### 1.2 Chất lượng hiện tại (baseline 2026-10-13, grammar 1.3.0)

| Prompt | Bộ dữ liệu | Mức kind | Mức action |
|---|---|---|---|
| `current` | authored 216 lượt | 203/216 (94.0%) | 210/216 (97.2%) |
| `current` | corpus 207 lượt thật | 150/207 (72.5%) | 168/207 (81.2%) |
| `v2` (bản nháp lúc đó) | authored | không định nghĩa | 167/216 (77.3%) |
| `v2` (bản nháp lúc đó) | corpus | không định nghĩa | 140/207 (67.6%) |

- Gemini 2.5 Flash đạt 215/216 trên bộ authored (bench 2026-07-24).
- `_SYSTEM_NOTE` nâng bench từ 163 lên 204/216: model đọc `answer` là "caller trả lời" thay vì "agent trả lời câu hỏi".
- **Độ nhiễu:** ở temperature 0, hai lần chạy giống hệt nhau vẫn lệch ở 11/217 dòng. Chênh dưới khoảng 11 dòng corpus không phải kết quả.
- Số `v2` ở trên là của bản nháp trước các thay đổi DG21/DG22 (2026-10-19). Chưa thấy số đo mới cho grammar 2.0.0/1.4.0.

### 1.3 Cấu hình `multilingual-e5-small` + Linear Head

Đây là một đường khác, chỉ dùng ở mode `local`, `hybrid`, `shadow`. `assets/decision/` hiện chỉ có README, không có `model.onnx`, `tokenizer.json`, `head.npz`, nên các mode đó tự quay về Gemini. Đường này không phải đường Decision đang chạy.

### 1.4 Yêu cầu hiện tại

- **Ngôn ngữ:** chưa có yêu cầu tiếng Việt hay đa ngôn ngữ. Dữ liệu đánh giá hiện đều là tiếng Anh (0/207 dòng trong `cases.jsonl` và 0/791 dòng trong `authored_turns_v6` có dấu tiếng Việt).
- **Độ trễ:** độ trễ Decision phải **thấp hơn hoặc ít nhất bằng** hiện tại (median 286–347 ms). Đây là mục tiêu không tệ hơn, không phải mục tiêu tiết kiệm.
- **Cấu trúc:** giữ nguyên cấu trúc in/output và serving OpenAI-compatible, tránh thay đổi lớn.
- **Server:** hạn chế động vào server, nên benchmark chất lượng chạy ngoài.

---

## 2. Phát hiện quan trọng cho quyết định

1. **Nút thắt là dữ liệu, không phải model.** Corpus thật có 207 dòng, bộ authored 646 dòng. Báo cáo dataset (2026-08-29) ghi rằng chưa có lượt thật nào từng được dùng để train. Nhãn D1 là output của Gemini, chưa ai adjudicate. Có 5 nhóm nhãn xung đột (cùng text và cùng feature nhưng nhãn khác nhau) và 8/15 call của D1 bị rò sang tập extraction.
2. **Tác vụ gần với phân loại.** Output rất ngắn và đã bị ràng buộc bằng regex. Chi phí nằm ở prefill khoảng 1.2k token. Prompt cache không giúp vì prompt ngắn hơn một block (544 token).
3. **Đạt độ trễ "bằng hoặc thấp hơn" không tự động.** Qwen3.6 là MoE với khoảng 3B tham số active, chạy NVFP4. Prefill 1.2k token tốn compute theo số tham số active, nên một model dense 4B không nhanh hơn và có thể chậm hơn, nhất là nếu chạy bf16. Model ≤1.7B hoặc bản lượng tử hóa (FP8/AWQ/NVFP4) mới có cơ sở nhanh hơn (suy luận, cần đo). Một phần độ trễ hiện tại còn do Decision xếp hàng sau việc đọc prompt của reply trên cùng GPU, nên endpoint riêng có thể giúp (cũng cần đo).
4. **Model nhỏ nhạy với prompt.** Qwen3.6-35B còn cần system note. Model ≤1.7B zero-shot nhiều khả năng kém hơn, nhưng chưa có số đo.
5. **Độ trễ không phụ thuộc trọng số đã fine-tune.** Nó phụ thuộc kích thước, kiến trúc và lượng tử hóa. Vì vậy có thể đo khả thi độ trễ bằng bản base của ứng viên trước khi tốn công fine-tune.

---

## 3. Ba hướng và đánh giá

| | Hướng 1: dense thuần (zero-shot) | Hướng 2: dense + fine-tune | Hướng 3: model chuyên biệt |
|---|---|---|---|
| Ví dụ | Qwen3 0.6B/1.7B/4B-2507, Gemma3 1B/4B-it, Phi-4-mini | Qwen3-1.7B/4B + LoRA, Qwen3.5-0.8B + fine-tune | Tev1-4B, Kev-0.8B, Laya Typed-Decisions, Von 1.3 |
| Cần dữ liệu | Không | Có, vừa phải | Có, nhiều hơn và cần thiết kế |
| Tích hợp | đổi `DECISION_LLM_URL`/`MODEL` | như hướng 1, giữ client và regex | Kev, Laya, Von cần runtime/SDK riêng; Tev1 có hướng dẫn vLLM nhưng bị chặn license |
| Khi grammar đổi | rule nằm trong prompt | nên retrain khi kind đổi | thêm option thì được, đổi ngữ nghĩa kind thì phải có dữ liệu và đánh giá lại |
| Rủi ro chất lượng | cao ở model ≤1.7B | thấp–trung bình nếu dữ liệu tốt | chưa có số đo nội bộ cho cả bốn |
| Rủi ro khác | thấp | thấp | xem bảng dưới |

### Kết quả xác minh bốn model chuyên biệt

Nguồn: `docs/decision_specialist_models_verification_vi.md` (23/10/2026). Tôi chưa kiểm tra lại các nguồn bên ngoài.

| Model | License | Kiến trúc và runtime | Context | Điểm chặn chính |
|---|---|---|---|---|
| Tev1-4B-experimental | license của weights chưa chốt | Qwen3.5-4B fine-tune, chọn một chữ cái; có hướng dẫn vLLM | 2.048 token (recipe) | License weights |
| Kev-0.8B | Apache-2.0 | Qwen3.5-0.8B-Base đóng băng + LoRA + pointer head; SDK/server riêng (`/v1/systemone`) | 8.192 token (đã xác thực) | Runtime riêng, English-only |
| Laya Typed-Decisions | Apache-2.0 | ModernBERT-large 421M; thư viện/server riêng | **1.024 token** | Input Decision khoảng 1.1–1.3k token, phải cắt hoặc thiết kế lại input |
| Von 1.3 | Apache-2.0 | ModernBERT khoảng 395M; `von-sdk`/`von serve` | không ghi | Repo đã archive (19/10/2026), runtime riêng |

Điểm chung của cả bốn:
- Chưa có kết quả chạy trên bộ authored 216 lượt hay corpus 207 lượt. Kết luận đúng là "chưa chứng minh được", không phải "kém hơn Qwen3.6".
- Chưa chứng minh xử lý đúng `reject`/`escalate`, và chưa chứng minh hiểu `workflow_active`, `awaiting_input`.
- Benchmark công bố của tác giả không cùng bài toán (voice-agent/hội thoại ngắn) nên không so trực tiếp với baseline.
- Mask theo `K_allowed` làm được ở tầng ứng dụng bằng cách chỉ truyền các option được phép. Điều này tương đương với mask bằng regex của hướng 2, nên không phải lợi thế riêng của hướng nào.
- English-only không còn là điểm trừ vì chưa có yêu cầu đa ngôn ngữ. Đây là rủi ro tương lai nếu sau này cần tiếng Việt.
- Lợi thế thật của hướng 3 là độ trễ tiềm năng thấp (Von ghi p50 khoảng 23 ms trên A10G và 96 ms trên CPU, là phép đo của tác giả, không tương đương GB10). Nhưng mục tiêu hiện tại chỉ là không tệ hơn, nên độ trễ không phải yếu tố quyết định.

---

## 4. Khuyến nghị

Chọn **hướng 2** làm nhánh chính, đi theo từng bước, giữ nguyên giao diện OpenAI-compatible và regex decoding.

Lý do chính là chi phí tích hợp và rủi ro vận hành: ba trong bốn model chuyên biệt cần runtime riêng, Laya bị giới hạn context 1.024 token, Tev1 chưa có license cho weights. Chất lượng của cả hai hướng đều chưa được chứng minh, nên không khẳng định hướng 2 tốt hơn.

- Hướng 1 chỉ là baseline (bước 2), không phải đích.
- Hướng 3 **không bị loại**: ở bước 2 đưa tối đa một model chuyên biệt vào benchmark làm ứng viên phụ, nếu adapter tối thiểu chạy được trên cùng prompt đã dump. Kev hợp lý nhất (context 8.192 token, Apache-2.0). Laya chỉ nên thử nếu rút gọn được input xuống dưới 1.024 token.
- Chuyển hướng nếu một model chuyên biệt vượt qua các điều kiện cứng và cho kết quả tốt hơn trên cùng dữ liệu và điều kiện.

### Tiêu chí chấp nhận

| Tiêu chí | Ngưỡng |
|---|---|
| Độ trễ Decision p50 | không cao hơn baseline đo cùng điều kiện (hiện 286–347 ms) |
| Độ trễ Decision p95 | không cao hơn baseline đo cùng điều kiện (cần đo; quan sát rộng 0,23–1,9 s) |
| Chất lượng (mức action) | không thấp hơn baseline vượt ngưỡng nhiễu (khoảng 11 dòng trên 217) |
| `reject`, `escalate` | không tụt so với baseline |
| Vi phạm `K_allowed` | 0 |

Đo độ trễ dưới tải thật, không đo trên GPU rảnh, vì độ trễ hiện tại chịu ảnh hưởng của tranh chấp GPU.

---

## 5. Kế hoạch 6 bước và tài nguyên

### Bước 1. Chuẩn hóa dữ liệu, nhãn, grammar

Có trong repo (`artifacts/eval/decision_corpus/`):
- `cases.jsonl`: 207 dòng nhãn người (`labeller: tson`), có `agent_prior` và `context` kèm provenance.
- `SPLIT.json`: chia theo session, không rò rỉ (train 135, eval-unseen-workflow 54, test 33, validation 22 theo số session trong file).
- `candidates.jsonl` (490), `ai_labels.jsonl` (420), `turn_corpus/authored_turns_v6.jsonl` (646), `mined_turns_v7`, `counterfactual_probe.jsonl` (150).
- Công cụ gán nhãn: `label.py`, `LABELLING.md`, `RUNBOOK.md`.
- Grammar: `grammar.py` (v1, 11 kind), `grammar_v2.py` (7 kind), `V1_TO_ACTION` trong `llm-decision-guide-2026-10-12/guide_prompts.py`.

Còn thiếu:
- Quyết định grammar đích (v1 hay v2) và nhãn v2 (cần map và người duyệt lại).
- Dữ liệu cho kind hiếm, đặc biệt `reject` và `escalate`.
- Đóng băng tập đánh giá, metric (action-level, riêng kind an toàn) và các tiêu chí chấp nhận ở mục 4.
- Gắn phiên bản `grammar`, `dataset`, `prompt`, `model`, `parser` vào mỗi lần chạy.

### Bước 2. Benchmark zero-shot (ngoài server, chỉ đo chất lượng)

Ứng viên: Qwen3 0.6B/1.7B/4B-Instruct-2507, Gemma3 1B/4B-it, Phi-4-mini-instruct, Qwen3.5-0.8B, cùng tối đa một ứng viên chuyên biệt (ưu tiên Kev).

Có trong repo:
- Bộ 52 case / 216 lượt: `spikes/decision_thinking_bench.py`, `spikes/decision_model_bench.py`.
- Baseline Qwen3.6 để so (mục 1.2).
- `build_prompt_v2` và `dg7_pattern` là hàm thuần, dựng được prompt mà không cần server.

Còn thiếu:
- Script ghi lại prompt (một `ProducerClient` giả cắm vào `build_serving_pipeline(client=...)`) và script replay ngoài.
- GPU ngoài. Máy làm việc hiện tại có 7 GB RAM, không có GPU, không có `torch/peft/vllm`.
- Harness baseline ngày 2026-10-13 nằm ngoài repo (scratchpad), nên chỉ dùng `ablation_v9/harness.py` và các `spikes/*bench.py` làm điểm xuất phát.

Điều kiện để kết quả ngoài server có giá trị:
1. Dùng đúng prompt do code của hệ thống dựng, dump ra JSONL cùng `kinds` và `references`.
2. Dùng cùng cách decode: temperature 0 và ràng buộc regex (vLLM `structured_outputs`).
3. Chỉ so độ chính xác, không so độ trễ.

Lưu ý kỹ thuật:
- `Qwen3-*-Instruct-2507` không có thinking mode. `Qwen3-0.6B/1.7B` bản base có, phải truyền `enable_thinking: false`.
- Gemma không có system role. Đường v2 chỉ dùng message user nên không ảnh hưởng, đường v1.3.0 (có `_SYSTEM_NOTE`) thì có.
- Với model chuyên biệt, giữ nguyên thông tin ngữ nghĩa đầu vào, ghi rõ adapter, truyền đúng danh sách option hợp lệ.

### Bước 3. Kiểm tra khả thi độ trễ sớm trên GB10

Chạy bản base của 1–2 ứng viên còn lại tốt nhất trong container riêng, cổng riêng, rồi đo p50 và p95 trên prompt thật (khoảng 1.2k token). Loại model không đạt p50 ≤ baseline. Làm bước này trước khi fine-tune vì độ trễ không phụ thuộc trọng số đã fine-tune.

Có:
- `decision-latency-ideas-2026-10-19/` (`dl_gate.py`, `replay_cache.py`, `run_on_gb10.sh`).
- `.claude/skills/gb10-latency-ab/`.
- Override `SH_DECISION_LLM_URL` (chỉ cho Decision, dùng cho A/B).

Còn thiếu:
- Container thử nghiệm trên cổng riêng, để không động đến Qwen3.6 đang chạy.
- Dung lượng bộ nhớ trống trên GB10 (chưa kiểm tra).
- Thử cả bản lượng tử hóa (FP8/AWQ/NVFP4) vì model dense bf16 có thể không nhanh hơn Qwen3.6 NVFP4.

### Bước 4. Fine-tune duy nhất model triển vọng nhất

Có: dữ liệu ở bước 1, Qwen3.6 làm teacher qua endpoint hiện có.

Còn thiếu:
- GPU ngoài (LoRA 1.7B/4B; GPU 16–24 GB là ước lượng).
- Thư viện `peft/trl/transformers` và script train.
- Prompt dài khoảng 1.2k token nên cần tính bộ nhớ theo độ dài này.

Nguyên tắc:
- Distill từ Qwen3.6 cộng nhãn người. Nhãn người sửa chỗ teacher sai.
- Tập vàng do người gán nhãn chỉ dùng để đánh giá, không dùng để train.
- Tách train/eval theo session và nhóm paraphrase.

### Bước 5. Đo chất lượng và hiệu năng đầy đủ trên GB10

Đo theo các tiêu chí chấp nhận ở mục 4, dưới tải thật, với p50, p95, p99, VRAM, tỷ lệ output hợp lệ, vi phạm `K_allowed`, `reject`/`escalate` và timeout. Tách độ trễ inference riêng, độ trễ cả Decision stage, và độ trễ end-to-end khi dùng chung GPU.

### Bước 6. Shadow deployment

Có: `producer_mode.py` (mode `shadow`), `ShadowProducerClient`.

Còn thiếu:
- Trong `decision_serving.py`, shadow hiện có `primary=GeminiProducerClient` và `secondary` là client local ONNX. Hệ thống đang ở mode `llm`, nên cần mở rộng code để Qwen3.6 là primary và model fine-tune là secondary qua `OpenAICompatibleProducerClient`.
- Shadow tạo thêm tải trên GPU dùng chung. Phải chạy bất đồng bộ và không nằm trên đường chờ của caller.
- Nơi lưu bản ghi, cách join với transcript, và ngưỡng rollback theo các tiêu chí ở mục 4.

---

## 6. Các vấn đề cần xử lý trước khi chạy

1. **Nhãn là grammar v1, không phải v2.** `cases.jsonl` có 8 kind: route 103, answer 41, clarify 33, fallback 20, offer 5, transfer 2, greet 2, close 1. Chưa có `escalate`, `reject`, `courtesy`. Trong luồng e5+head, `train` từ chối fit nếu thiếu kind an toàn. Với hướng 2 cũng cần có kind an toàn trong tập đánh giá.
2. **Tập đánh giá quá nhỏ.** Chia theo session còn `test` 26, `validation` 22, `eval-unseen-workflow` 47 dòng. Với độ nhiễu khoảng 11/217, 26 dòng không phân biệt được hai model gần nhau. Nên dùng cross-validation theo session và gom các tập held-out làm tiêu chí chính.
3. **Nhãn corpus real chưa chắc đã adjudicate thống nhất.** Cần xác minh nguồn nhãn trước khi coi 72.5% là baseline chính thức.
4. **Shadow chưa dùng được cho kịch bản này** (mục bước 6).
5. **`decision_corpus/README.md` đã cũ:** ghi "0 rows labelled" nhưng `cases.jsonl` có 207 dòng.

---

## 7. Cách trình bày lý do chọn hướng 2

Mục tiêu độ trễ chỉ là bằng hoặc thấp hơn hiện tại, nên **độ trễ không phải lý do để làm dự án**. Không dùng con số "tiết kiệm 100–200 ms" làm căn cứ trừ khi bước 3 hoặc 5 xác nhận.

Bản tóm tắt (khoảng 30 giây):

> Decision là bài phân loại ngắn nhưng phụ thuộc ngữ cảnh. Qwen3.6-35B làm được việc này nhưng là model lớn dùng chung GPU với các thành phần khác. Chúng tôi muốn một model nhỏ chuyên dụng cho Decision mà không phải viết lại hệ thống. Fine-tune một LLM nhỏ giữ nguyên giao diện OpenAI-compatible, regex decoding và client, nên đổi model chỉ là đổi biến môi trường. Ba trong bốn model chuyên biệt cần runtime riêng, một cái bị giới hạn context, một cái chưa có license weights. Chất lượng của cả hai hướng đều chưa được chứng minh trên dữ liệu của chúng tôi, nên kết luận cuối dựa trên số đo, với tiêu chí chấp nhận và ngưỡng rollback định trước.

Lợi ích có thể nêu (đều là giả thuyết cần đo):
- Tách Decision khỏi Qwen3.6 dùng chung GPU, giảm tranh chấp tài nguyên.
- Giảm độ nhiễu (hiện lệch 11/217 dòng giữa hai lần chạy giống hệt nhau).
- Cơ hội cải thiện chất lượng bằng dữ liệu của chính hệ thống (corpus thật chỉ đạt 72.5% mức kind).
- Không phụ thuộc vào một model 35B chỉ để làm một bài phân loại.

Các phản biện có thể gặp:

| Phản biện | Trả lời |
|---|---|
| Sao không giữ Qwen3.6? | Chất lượng ổn (203/216 authored). Lý do thay là tách khỏi GPU dùng chung, giảm nhiễu và cải thiện bằng dữ liệu riêng, không phải vì độ trễ. Nếu bước đo cho thấy không có lợi ích thật thì có thể giữ nguyên. |
| Sao không zero-shot? | Qwen3.6-35B còn cần system note để từ 163 lên 204/216. Model ≤4B zero-shot nhiều khả năng kém hơn, nên bước 2 chỉ để đo. |
| Sao không dùng model chuyên biệt? | Không loại. Ba trong bốn model cần runtime riêng, Laya giới hạn context 1.024 token, Tev1 chưa có license weights, Von đã archive. Cả bốn chưa có số đo nội bộ. Kev được đưa vào bước 2 làm ứng viên phụ. |
| Sao không dùng e5 + linear head (7 ms)? | Mới fit trên 791 dòng authored và 0 lượt thật. Nên giữ cho chế độ `hybrid` ở giai đoạn sau. |
| Model dense nhỏ có chắc nhanh hơn không? | Không chắc. Qwen3.6 là MoE khoảng 3B active, NVFP4. Model 4B dense có thể không nhanh hơn, nên ưu tiên ≤1.7B hoặc bản lượng tử hóa, và đo sớm ở bước 3. |
| Dữ liệu có đủ không? | Chưa chắc, 207 dòng thật là ít. Vì vậy bước 1 đặt lên đầu, và cần distill từ teacher cộng nhãn người. |
| Tiếng Việt thì sao? | Chưa có yêu cầu, và dữ liệu hiện đều là tiếng Anh. Nếu sau này cần thì phải bổ sung dữ liệu và đánh giá lại. |

Điểm yếu cần nói thẳng:
- Distill kế thừa lỗi của teacher. Qwen3.6 chỉ đạt 72.5% mức kind trên corpus, một phần do nhãn mơ hồ.
- Rủi ro lớn nhất là tụt chất lượng ở `reject` và `escalate`.
- Không đảm bảo đạt mục tiêu độ trễ nếu chọn model quá lớn hoặc không lượng tử hóa.
- Giảm thiểu bằng: tập vàng do người gán nhãn dùng riêng để đánh giá, kiểm tra độ trễ sớm ở bước 3, ngưỡng dừng sau bước 2 và 3, và shadow trước khi chuyển production.

---

## 8. Việc tiếp theo đề xuất

1. Viết script ghi lại prompt (client giả, không cần server) để dựng bộ prompt cho bước 2.
2. Chốt grammar đích và cách map nhãn v1 sang v2.
3. Chuẩn bị GPU ngoài cho bước 2 và 4.
4. Kiểm tra bộ nhớ trống trên GB10 và `DECISION_GRAMMAR` đang chạy.
5. Chốt ngưỡng cụ thể cho các tiêu chí chấp nhận (mục 4) với người phụ trách.

## 9. Nguồn trong repo

- `docs/decision_specialist_models_verification_vi.md` (xác minh bốn model chuyên biệt, 2026-10-23)
- `services/voice-ai-orchestrator/src/voice_ai_orchestrator/conversation_orchestrator/decision/` (`producer_client_openai.py`, `producer_mode.py`, `grammar_v2.py`, `prompt_v2.py`, `local_model.py`)
- `services/voice-ai-orchestrator/src/voice_ai_orchestrator/conversation_integration/decision_serving.py`
- `services/voice-ai-orchestrator/assets/decision/README.md`
- `selfhosted-ai/docker-compose.connect.yml`, `selfhosted-ai/.env.example`
- `artifacts/eval/decision-baseline-2026-10-13/report.md`
- `artifacts/eval/decision-producer-model-benchmark-2026-07-24.md`
- `artifacts/eval/decision-dataset-strategy-2026-08-29.md`
- `artifacts/eval/decision-latency-ideas-2026-10-19/RESULT.md`
- `artifacts/eval/gb10-latency-context-2026-10-12.md`
- `artifacts/eval/decision_corpus/` (`cases.jsonl`, `SPLIT.json`, `LABELLING.md`, `README.md`)
- `artifacts/eval/llm-decision-guide-2026-10-12/guide_prompts.py`
