# 9 tác vụ LLM của luồng hội thoại: phân tích và prompt đầy đủ

> Tách từ `STAGING_OVERVIEW_VI.md` (trước đây là mục 19). Tài liệu này mô tả 9 tác vụ LLM của hệ thống `staging/` và **trích nguyên văn prompt** của từng tác vụ từ mã nguồn.
>
> **Quy ước.**
> - Prompt được giữ **nguyên bản tiếng Anh** (đó là chuỗi thật gửi cho model). Phần giải thích bằng tiếng Việt.
> - Đường dẫn tính từ `staging/services/voice-ai-orchestrator/src/voice_ai_orchestrator/` (viết tắt `O/`), trừ khi ghi khác.
> - Chỗ có biến động, như `{name}`, `{caller}`, `__CURRENT__`, là chỗ code điền dữ liệu lúc chạy.
> - Chưa chạy hệ thống. Nội dung rút ra từ code, không phải log runtime. Phần chưa trích hoặc chưa xác minh được ghi rõ ở cuối mỗi tác vụ và ở mục 11.
> - Ký hiệu đối chiếu với mô tả ban đầu: ✅ khớp · ⚠️ khớp một phần · ❓ chưa kiểm chứng.

---

## 1. Bảng đối chiếu nhanh

| # | Tác vụ | Khớp? | Model hiệu lực (staging) | Tham số gọi | Nơi cài đặt |
|---|---|:-:|---|---|---|
| 1 | Decision | ✅ | Qwen3.6-35B-A3B (`DECISION_LLM_*`) | temp 0, max_tokens 32, timeout 3 s, thinking tắt | `O/conversation_orchestrator/decision/` |
| 2 | Anger judge | ⚠️ | Qwen (qua `LOCAL_JSON_LLM_*`) | temp 0, max_tokens 2, timeout 2 s | `O/ai/grounded_reply.py::judge_anger` |
| 3 | Bridge | ⚠️ | Qwen (endpoint reply-LLM, `LOCAL_LLM_*`) | temp 0.7, max_tokens 12, stream, thinking tắt | `O/ai/bridge.py` |
| 4 | Reply | ⚠️ | Qwen (`LOCAL_LLM_*`) | temp 0.7 (cố định), max_tokens 320, stream, tool_choice auto | `O/ai/llm_local_chat.py`, `O/ai/llm.py` |
| 5 | Field extraction | ✅ | **NuExtract-2.0-2B** (vLLM riêng) | completion thô, timeout 8 s | `O/information_extraction/nuextractor.py` |
| 6 | Grounded reply | ⚠️ | Qwen (qua `LOCAL_JSON_LLM_*`) | temp 0.6, max_tokens 90 (hold: 60), timeout 4 s | `O/ai/grounded_reply.py::compose_reply` |
| 7 | Sentiment (text) | ✅ | Qwen (`EMOTION_TEXT_*`) | temp 0, max_tokens 200, JSON mode, thinking tắt | `O/ai/emotion.py`, `O/ai/emotion_sensevoice.py` |
| 8 | Call purpose | ✅ | Qwen (qua `GeminiCaller` → `LOCAL_JSON_LLM_*`) | temp 0, max_tokens 120, JSON mode, timeout 8 s | `O/ai/caller_purpose.py` |
| 9 | Post-call analysis | ✅ | Qwen (như trên) | temp 0.2, max_tokens 600, JSON mode | `O/ai/post_call_analysis.py` |

**Về câu "tất cả dùng Qwen3.6 35B A3B trừ khi nêu khác":** đúng với cấu hình hiệu lực hiện tại (`docker compose config`): `DECISION_LLM_MODEL`, `LOCAL_LLM_MODEL`, `LOCAL_JSON_LLM_MODEL`, `EMOTION_TEXT_MODEL` đều là `qwen3.6-35b-a3b`, cùng trỏ `http://vllm_chat_qwen_3_6:8000`. Ngoại lệ là #5 (NuExtract 2B, vLLM riêng `ai-nuextract`). Các tác vụ #8, #9 vẫn dùng lớp `GeminiCaller`, nhưng chạy cục bộ **vì** `LOCAL_JSON_LLM_BASE_URL` được đặt. Mất biến này thì chúng quay về Gemini.

---

## 2. Tác vụ 1 — Decision

**Mục tiêu.** Chọn đúng một "kind" cho lượt hiện tại. **Thời điểm:** mỗi lượt, kể cả bắt đầu sớm ở mỗi STT segment final khi `EARLY_LANE=on`.

**Xác minh mô tả.**
- ✅ Có đúng 11 kind trong `DecisionCategory` (`decision/model.py`): `greet, answer, route, offer, clarify, transfer, escalate, reject, fallback, courtesy, close`. Không có `wait`/`backchannel`.
- ✅ `.env` ghi `DECISION_PRODUCER_MODE=shadow` nhưng overlay đặt `${SH_DECISION_PRODUCER_MODE:-llm}` nên hiệu lực là **`llm`**.
- Prompt dưới đây là **ngữ pháp 1.3.0** (mặc định, `DECISION_GRAMMAR` không đặt). Ngữ pháp v2 (`prompt_v2.py`, 7 loại caller kind, bật bằng `DECISION_GRAMMAR=2.0.0`) là prompt khác, **hiện không bật trên staging** và được trích ở mục 2.4.

**Cách gọi** (`producer_client_openai.py`): một request `/v1/chat/completions` với hai message `system` + `user`, `temperature: 0.0`, `max_tokens: 32`, `chat_template_kwargs: {enable_thinking: false}`. Câu trả lời được parse máy móc (không phán đoán ngữ nghĩa): không nhận diện được hoặc kind không hợp lệ thì thành `fallback`. Có thể bị cắt bằng timeout 3 s (`DECISION_LLM_TIMEOUT_S`).

### 2.1 Message `system` (`_SYSTEM_NOTE`, nguyên văn)

```text
Each decision kind names what the AGENT does next, not what the caller just did. 'answer' means the agent answers a question the caller asked. When the caller is replying to something the agent asked - giving a name, date, time, number, choice, yes/no or other detail - that is NOT 'answer': if a task in progress is waiting for that information, choose 'route' to that task. If the caller wants to do something else but does not make clear which of the listed tasks they mean, choose 'clarify' - do not guess a task and do not refuse.
```

### 2.2 Message `user` (khung do `prompt.py::build_prompt` dựng)

Phần đầu "ổn định" (đặt trước để tận dụng prefix cache), phần động đặt sau.

```text
{persona}

Choose exactly ONE of these decision kinds:
{kinds}
{reference_clause}

Conversation:
{conversation}

Known facts:
{facts}

{tail}
```

- `{persona}` mặc định (`DEFAULT_PERSONA`):
  ```text
  You are the decision layer of a voice agent. Reason over the conversation and the known facts, then choose how the agent should act this turn.
  ```
- `{kinds}`: mỗi kind một dòng `- <kind>: <nghĩa>`, sắp theo tên, **chỉ gồm các kind hợp lệ ở lượt này** (xem 2.3).
- `{reference_clause}`: chỉ có khi lượt có tham chiếu (capability khả dụng hoặc bằng chứng). Không có mô tả thì là câu:
  ```text
  If your chosen kind names one of these, also choose exactly ONE of these references: {ref1, ref2, ...}.
  ```
  Có mô tả thì thành danh sách:
  ```text
  If your chosen kind names one of these, also choose exactly ONE of these references (name it by its identifier, before the colon):
  - {ref}: {description}
  ```
- `{conversation}`: các lượt trước, dòng cuối là lời người gọi hiện tại, được gắn nhãn `Caller:` (`label_current_utterance`). `(none)` nếu chưa có.
- `{facts}`: các dòng `- {subject}.{meaning} = {value!r} [{owner}]`, sắp theo thứ tự; bỏ các dòng đã lặp (nội dung lời người gọi, mô tả capability đã in ở phần reference). `- (no facts)` nếu rỗng.
- `{tail}`, nguyên văn:
  - Khi có tham chiếu: `Respond as '<kind>' or '<kind>|<reference>' in lowercase, and nothing else.`
  - Khi không có: `Respond with only the single chosen kind in lowercase, and nothing else.`

### 2.3 Ý nghĩa 11 kind (`grammar.py::DECISION_KIND_MEANINGS`, nguyên văn)

Phần `{kinds}` ở trên lấy từ bảng này. Nguyên văn các câu mô tả (đã bỏ chú thích code):

| Kind | Nội dung đưa vào prompt |
|---|---|
| `greet` | return the caller's greeting — they are making or renewing social contact with you and doing nothing else, whether the call has only just begun or they simply greet you again partway through; a leave-taking is the OPPOSITE act and belongs to a close however alike the two sound, so choose this only when the caller is arriving rather than departing; choose it only when the turn contains no question and no request, and supplies nothing you need to act on |
| `answer` | answer the caller's question — the agent answers from its own knowledge, so this needs no capability and no matching task, and it stays the right choice for a question none of the listed capabilities covers; appropriate even while a task is in progress, and the right choice when the caller asks ABOUT a task rather than for it: how it works, what it requires, what it costs or how long it takes |
| `route` | hand this turn to a capability because THIS TURN advances the task that capability handles — including when the caller returns to a suspended task without adding to it — name that capability as the reference; choose this only when the turn ASKS for that task to be started, carried on or carried out. A turn that states a fact about the caller's own situation, or that asks how the task is done, what it needs, or whether it is possible, is not such a request even though it names the very same subject |
| `courtesy` | acknowledge appreciation — the caller thanks you, or is pleased with what you just did, and asks for nothing further; choose this only when the turn contains no question and no request, only when the caller is not opening or re-opening contact with you — a greeting or an opening good wish is not appreciation however warm it sounds — and only while the caller gives no sign of finishing the call |
| `close` | say goodbye because the caller is FINISHING the call — a farewell, a parting good wish, saying that is all they need, or a closing thanks that settles the conversation rather than continuing it; choose this over a courtesy whenever the same turn both thanks you AND signals the end, and not when the caller is still asking for something or still telling you something you need to act on |
| `offer` | volunteer something the caller has not asked for but plainly needs — a capability that could handle what they are trying to do, or, when a task is already under way, what you know that they do not and that would let them answer what that task is asking of them — and name what you are offering as the reference; choose this over handing the turn onward whenever the caller cannot supply what is being asked from what they already know, whether they say so, ask what the choices are, or leave the choice to you, because none of those is progress on the task and asking again will not make it so |
| `clarify` | ask the caller to resolve an ambiguity |
| `transfer` | hand the call to another destination |
| `escalate` | raise the call to a higher authority |
| `reject` | decline the caller's request |
| `fallback` | none of the above applies — including when the caller withdraws from, abandons or calls off the task now in progress without asking for anything else |

**Lớp bảo vệ sau LLM** (không nằm trong prompt): `guards.py` / `admissibility*.py` thu hẹp tập kind hợp lệ trước khi dựng `{kinds}` (ví dụ `route` bị loại khi không có capability khả dụng). `decision_engine.py` đổi một `close` không có lời tạm biệt thành `answer`.

### 2.4 Ngữ pháp v2 (`DECISION_GRAMMAR=2.0.0`) — hiện **không bật** trên staging

Cấu hình hiệu lực của orchestrator **không có** biến `DECISION_GRAMMAR`, nên `resolve_grammar` trả `1.3.0`. Ngữ pháp 2.0.0 chỉ được chấp nhận khi `DECISION_PRODUCER_MODE=llm` (đang đúng), còn ngược lại tự lùi về 1.3.0 kèm cảnh báo. Hai công tắc `DECISION_IMPLIED_REF=on` và `DECISION_SOCIAL_LINE=on` đã được overlay đặt sẵn nhưng chỉ có tác dụng khi v2 chạy.

**Khác biệt với 1.3.0** (`prompt_v2.py`):
- Khối kind khác: 7 "caller kind" (`social, request, reply, question, stop, human, unclear`) thay vì 11 kind của agent.
- Dòng trả lời khác, và **không có message `system`** (chỉ một message `user`).
- Request thêm `structured_outputs: {"regex": <mẫu DG7>}` để vLLM ép đầu ra đúng dạng; `max_tokens` 64 thay vì 32. `temperature 0.0`, thinking tắt như cũ.
- Đầu ra là một dòng có cấu trúc, sau đó được kiểm tra bằng `re.fullmatch` và chuẩn hoá theo thứ tự ưu tiên `human > request > stop > reply > question > unclear`; `social` không có thứ hạng.

**Khung message `user`** (`build_prompt_v2`):

```text
{persona}

{kind_block}
{capability_list}

Conversation:
{conversation}

Known facts:
{facts}

{answer_lines}
```

`{persona}`, `{conversation}`, `{facts}` giống mục 2.2. `{capability_list}` chỉ có khi lượt có capability:

```text
Capabilities:
- {capability_id}: {description}
```

**`{kind_block}`** (nguyên văn; ví dụ cho biến thể đầy đủ gồm cả cổng `reply` và `request`). Các dòng câu hỏi được đánh số lại từ 1 và **bỏ dòng của kind không hợp lệ** ở lượt đó:

```text
1. Does the caller ask for a person? → human
2. Is a task waiting for input, and does this turn respond to its question? → reply
3. Does the caller ask for a task? → request
4. Does the caller drop the task? → stop
5. Does the caller ask for information? → question
6. Does the turn only greet, thank, or say goodbye? → social
7. Otherwise → unclear
Kinds:
social: The turn only greets, thanks, or says goodbye.
request: The caller asks for a task that a listed capability performs: to start it, resume it, or switch to it.
reply: A task is waiting for an answer, and this turn responds to its question: with the answer, a confirmation, or a sign that the caller cannot give it.
question: The caller asks for information.
stop: The caller drops the current task and asks for nothing else.
human: The caller asks to speak to a person, or refuses to continue with an automated agent.
unclear: You cannot tell what the caller wants, or the caller did not catch what the agent said.
```

Có 4 biến thể (cổng `reply` mở khi có task đang chờ trả lời, cổng `request` mở khi có capability điều hướng được). `social, question, stop, human, unclear` luôn hợp lệ.

**`{answer_lines}`** (nguyên văn, với `DECISION_SOCIAL_LINE=on` thì dòng thứ 3 được chèn sau dòng `reply`):

```text
For request, write request:<id> with the id of the listed capability the caller asks for.
For reply, write reply:yes when the caller gives the answer or confirms, reply:no when the caller cannot give it.
A turn that only greets, thanks or says goodbye is social, also while a task waits for an answer.
If the turn also holds a second act of another kind, add |also= and that act. also is never social.
If the turn greets, thanks or says goodbye, add |acts= and those words from hello, thanks, bye, joined by commas. social always has acts.
Answer with one line only, for example: question, request:<id>, reply:yes, social|acts=hello, stop|acts=thanks
```

Với `DECISION_IMPLIED_REF=on` và đúng một capability được liệt kê, model chỉ cần viết `request` (không kèm `:<id>`), id được điền lại khi parse.

**Đầu ra hợp lệ** (mẫu DG7): `social|acts=<hello|thanks|bye,…>`, hoặc `<act>[|also=<act>][|acts=<…>]` với `act` là `request:<id>`, `reply:yes`, `reply:no` hoặc một kind trần. Không khớp mẫu thì lượt thất bại với lý do `not_dg7_line`.

---

## 3. Tác vụ 2 — Anger judge

**Mục tiêu.** Trả lời "yes"/"no": người gọi có đang giận hoặc rất bực với dịch vụ không. **Thời điểm:** mỗi lượt người gọi, song song với Decision. Hàm `judge_anger` ở `O/ai/grounded_reply.py`, gọi từ `ai/driver_lifecycle.py::take_escalation_reason`.

**Cách gọi** (`local_json_llm.complete`): một message `user`, `temperature 0.0`, `max_tokens 2`, timeout 2 s (`ANGER_JUDGE_TIMEOUT_S`), thinking tắt. Kết quả dương tính khi câu trả lời, sau khi trim và hạ chữ thường, **bắt đầu bằng `yes`**. Lỗi, timeout hoặc chưa có URL thì trả `None` và **không** leo thang.

### Prompt (nguyên văn, ghép từ code)

`{context}` chỉ có khi có lời người gọi trước đó (tối đa 2 lượt gần nhất, cũ nhất trước). Lý do trong code: STT streaming hay cắt một câu giận dữ thành nhiều lượt.

```text
You monitor a customer service phone call for a home loan lender.
Just before, the customer said:
- "{earlier_line_1}"
- "{earlier_line_2}"
The customer now says: "{utterance}"

Is the customer angry or strongly frustrated with the service right now? Harsh judgements of the service or the delay, or saying plainly that they are unhappy, count as yes. Neutral questions, giving details, thanks, mild surprise and simple disappointment are no.
Answer with exactly one word: yes or no.
```

**Khác với mô tả ban đầu (⚠️).**
- Không chỉ LLM. Chính sách có **hai nguồn kích hoạt**: text-emotion của lượt *trước* là `angry`/`frustrated` với điểm ≥ 0,8 (kiểm tra trước), nếu không mới gọi LLM ở trên.
- Kích hoạt **đúng một lần mỗi cuộc gọi** và được chốt; từ đó mọi lượt trả lý do `hold`.
- Câu bàn giao là **văn bản cố định** (không do LLM sinh), xem mục 8. Chỉ câu "hold" ở các lượt sau do LLM sinh.

---

## 4. Tác vụ 3 — Bridge (câu mở đầu ngắn)

**Mục tiêu.** Sinh vài từ đầu của lượt trả lời trong khi câu trả lời thật còn đang chuẩn bị (ví dụ "Hmm, okay -", "Oh dear -"). **Thời điểm:** mỗi STT segment final / early final khi `BRIDGE_MODE=live` (hiệu lực staging: `live`). `O/ai/bridge.py`, spec `artifacts/spec/orchestrator/bridge-late.yaml` (LB4).

**Cách gọi:** endpoint chat của **reply-LLM** (`LOCAL_LLM_BASE_URL`/`LOCAL_LLM_MODEL`), stream, `temperature 0.7`, `max_tokens 12`, thinking tắt, không có knowledge, không có tools. Chờ tối đa 700 ms ở cuối lượt.

**Khác biệt nhỏ (⚠️).** Prompt yêu cầu **1–4 từ**, bộ kiểm tra (`BRIDGE_MAX_WORDS = 5`) chấp nhận **tới 5 từ**.

### 4.1 Message `system` = câu persona + `\n` + prompt (nguyên văn)

Câu persona (`persona_sentence`): `You are {name}, {role}.` ({role} là cụm mô tả vai trò trong prompt chính, bỏ dấu chấm cuối; không có role thì chỉ `You are {name}.`).

Tiếp theo là khối sau, trong đó `{name}` được thay bằng tên agent:

```text
On a phone call, you say the first beat of your turn while your real answer is still being prepared. Another system speaks the answer straight after your words.

You do not know the answer. It may be a yes, a no, "I don't have that information", or a question back. Your words must fit before ANY of these, so never sound like you are about to look something up or explain the subject.

React to what the caller just did, the way a warm, relaxed person on the phone does:
- asked something -> a light thinking beat: "Hmm, okay -", "Good question -", "Mm, so -", "Okay, so -", "Right then -"
- worried, upset or has a problem -> empathy: "Oh, I'm sorry -", "Ah, that's no fun -", "I hear you -", "Oh dear -"
- said something nice or agreed -> "Oh good -", "Lovely -", "Great to hear -"
- hesitated, unsure, or changed their mind -> "That's okay -", "No worries at all -", "All good -"
- asked who you are -> you may say your name: "I'm {name} -"
These are examples of the style; make your own words that fit THIS caller.

Rules:
- 1 to 4 words. End with a dash or a comma.
- Never name the subject, a product or any detail of the request.
- Never say check, look, find, help, sort, or anything you will do.
- No facts, no questions.
- Your first word must differ from the first word of each of your recent openings.
- If the caller's line stops mid-sentence or is only filler (uh, um), write exactly: NONE
```

### 4.2 Message `user`

```text
Agent's last line: {agent_last}
Your recent openings: {recent}
Caller: {caller}

Your opening words:
```

`{recent}` là tối đa 3 bridge đã phát trong cuộc gọi, cũ nhất trước, nối bằng `"; "`; `(none)` nếu chưa có.

### 4.3 Kiểm tra kết quả (`generated_line_checks.py`)

Một bridge bị loại nếu trả `NONE` (sau khi bỏ khoảng trắng, `.`, `-`, `,`, không phân biệt hoa thường), vượt quá 5 từ, chứa chữ số, có dấu `?`, chứa lời hứa, "can know", "recorded", hoặc tự nhận là người. Danh sách lý do theo thứ tự: `too_long, digit, question, promise, can_know, recorded, claims_human` (spec GC1).

---

## 5. Tác vụ 4 — Reply

**Mục tiêu.** Sinh câu trả lời nói ra cho người gọi. **Thời điểm:** theo mô tả ban đầu là khi Decision chọn `answer`. Qwen qua `O/ai/llm_local_chat.py` (`LLM_PROVIDER=local`), có tool calling (`LOCAL_LLM_TOOL_CALLING=1`) và knowledge được chèn vào (`KB_CONTEXT_INJECTION=1`).

**Khác biệt (⚠️).** Các chú thích trong `conversation_composition/bindings/capability.py` cho thấy các kind khác (ví dụ một `transfer` do Producer chọn) cũng đi qua "conversational default". Chưa kiểm chứng đầy đủ nhánh nào gọi reply LLM.

**Phần "section 2" của mô tả ban đầu không có nội dung** nên tài liệu này không đối chiếu với nó. Dưới đây là prompt **đúng như code lắp ráp**.

### 5.1 System prompt = ghép nhiều khối

Thứ tự lắp (`orchestrator.py::_build_system_prompt` + các khối phụ):

1. **`profile.system_prompt`**: persona/hướng dẫn của agent, lấy từ tenant-config (cấu hình theo từng agent trên dashboard; **không nằm trong repo**, chỉ có mẫu mặc định).
2. **Khối ngôn ngữ/miền** (`_language_context_block`): chỉ khi ngôn ngữ agent là `vi`; agent tiếng Anh/"multi" không có khối này.
3. **Khối vai trò** (`_role_context_block`): `\n\nRole: {Role Title}.` nếu agent có `role_id`.
4. **Khối knowledge** (`KNOWLEDGE_USE_SYSTEM_BLOCK`).
5. **`prompt_addendum` của domain pack** (`ai/domains/*`; dành riêng cho từng miền, **chưa trích**).
6. **Dòng metadata người gọi** (cuối cùng, theo spec RP1).

> Spec RP1–RP3: dòng metadata là khối cuối; chỉ dẫn dùng knowledge chỉ nêu **một lần** trong phần tĩnh.

**(a) Mẫu mặc định của dashboard** (`frontend/src/product/runtime-experience/workspace/surfaces/agents/constants/defaultPrompts.ts`, bản EN; `You are {Name}, a professional…` khi có tên agent):

```text
You are a professional, polite, and attentive customer service voice assistant. Your role is to assist callers with their inquiries, collect necessary information accurately, and resolve issues effectively.

Key Spoken Telephony Guidelines:
- Conversational & Concise: Keep responses natural, direct, and concise (1-3 sentences per turn) suitable for spoken phone calls. Avoid long monologues or markdown formatting.
- Active Listening: Acknowledge caller statements empathetically and confirm understanding before taking action.
- Step-by-Step Collection: When gathering caller information, ask for only one detail at a time.
- Verification: Clearly confirm critical details (such as dates, phone numbers, reference IDs) with the caller.
- Requests for a Person: If the caller asks for a person, acknowledge the request in one sentence. Never offer, promise or suggest a transfer, a connection to a person or a callback. You may give a phone number or website found in the knowledge.
```

Có bản tiếng Việt (`DEFAULT_SYSTEM_PROMPT_VI`) trong cùng file.

**(b) Quy tắc điện thoại dùng chung** (`ai/profiles.py::_TELEPHONY_RULES`, nằm cuối các prompt mặc định trong code như `_DEFAULT_PROMPT`, `_INSURANCE_PROMPT`):

```text
You are speaking with a caller over the phone. Your replies are spoken aloud by a text-to-speech engine, so you MUST follow these rules:
- Keep every reply concise — two or three short sentences.
- When the answer includes an amount or limit, a form name, where to send or submit something, or a time period, say every one of them, even if it takes a third sentence. Never drop one of them to stay short.
- Plain prose only — no markdown, bullet points, asterisks, code blocks, emojis, or any special formatting.
- Do not list options with numbers or letters — ask the caller which they prefer in plain language.
- Ask for one piece of information at a time.
- If the caller asks you to repeat, do so without complaint.
- If you don't know something, say so plainly and offer to take a message.

Capabilities (absolute):
- When asked what you can do, describe ONLY what the instructions below actually give you. Never guess, never generalise from examples, and never offer a task just because it sounds like something your kind of business does. If it is not named below, you cannot do it.
- Never invent a form, record, log, case or booking type that is not named below. If a caller asks for one, say plainly that you cannot do that on this call and offer to take a message.

Tone context (silent):
- Caller messages may be prefixed with a marker like `[tone: frustrated]` or `[tone: angry]`. This is acoustic metadata from a separate model — it is NOT something the caller said. NEVER read the marker aloud, NEVER acknowledge it explicitly, and NEVER ask the caller about their emotional state because of it. Just let it inform your warmth and pacing. If you see `[tone: frustrated]` or `[tone: angry]`, lead with empathy and offer a concrete next step. If you see `[tone: distressed]`, slow down and soften. If you see `[tone: confused]`, simplify your language and confirm understanding. If the caller asks for a person, acknowledge the request in one sentence. Never offer, promise or suggest a transfer, a connection to a person or a callback. You may give a phone number or website found in the knowledge.
```

Prompt mặc định "persona" ngắn đứng trước khối này, ví dụ `_DEFAULT_PROMPT`: `You are a helpful AI voice assistant for a BPO company. You are speaking with a caller over the phone.` (ngoài ra có `_FINTECH_PROMPT`, `_REALESTATE_PROMPT`, `_INSURANCE_PROMPT` cho các miền).

**(c) Khối ngôn ngữ** (chỉ `vi`, ghép `Language:` + hướng dẫn):

```text
Language: Vietnamese.
You must communicate with callers in Vietnamese. Respond naturally in Vietnamese unless the caller explicitly asks you to switch to another language. Never claim that you cannot speak Vietnamese, and never say you can only speak another language — that contradicts your configuration and what the caller can hear.
```
(Nếu agent có `domain`, thêm một dòng `Domain: {Domain}. Stay in character.`)

**(d) Khối knowledge** (`retrieval.py::KNOWLEDGE_USE_SYSTEM_BLOCK` = `_INSTRUCTION_SUCCESS` + `_INSTRUCTION_NO_QUESTION_CLAUSE`):

```text
When the caller's message is followed by a [knowledge] block (JSON with `status`, `results` and `count`):
The passages in `results` are your ONLY source of company-specific fact. Being on a related topic is not enough: a passage supports an answer only if it actually states the fact the caller asked about.
Do NOT apply a rule from one case to a different case, product, item, service, person or situation unless a passage says it applies there. Do NOT combine separate passages into a conclusion none of them states. Do NOT offer a related fact as though it answered the question. Do NOT fill a gap from your own knowledge.
If the passages state the answer, give it. If they state only part of it, give that part and say plainly what you cannot confirm. If they do not state it, say plainly that you do not have enough information to confirm that, and offer to take a message.
Speak naturally to the caller. Never mention passages, results, documents, searching, records or a knowledge base.
If the caller did not ask a question, do NOT abstain and do NOT say you lack information: simply continue the conversation, using a passage only if it contradicts or corrects what the caller just said.
```

Hai chỉ dẫn trạng thái khác (dùng khi đường knowledge theo kiểu "forced round", không phải kiểu chèn):

```text
(zero hits) The knowledge base contains nothing relevant. You do NOT know the answer. Say plainly that you do not have that information on file and offer to take a message. Do NOT answer from your own knowledge.

(error) The knowledge base could not be reached, so nothing was retrieved. Say plainly that you cannot access your records right now and offer to take a message. Do NOT answer from your own knowledge.
```

**(e) Dòng metadata người gọi** (`_caller_context_line`, thêm cuối; chỉ khi có ít nhất một thông tin):

```text
For this call you have the following metadata about the caller (do not read these values aloud unless the caller asks you to confirm them): caller_id_num='…', caller_id_name='…', dialed_extension='…', known_contact_name='…', known_contact_organization='…'.
```
(Chỉ in các trường có giá trị; contact "placeholder" bị bỏ qua.)

### 5.2 Message `user` mỗi lượt

- Lời người gọi, có thể có tiền tố `[tone: X] ` (xem tác vụ 7).
- Kèm khối `\n\n[knowledge]\n{JSON}` (với `status`, `results`, `count`) khi `KB_CONTEXT_INJECTION=1` và có truy xuất.

### 5.3 Tham số gọi và cấu trúc request (`llm_local_chat.py::_body`)

Request `/v1/chat/completions` (stream khi phát tiếng, unary ở nơi khác):

| Trường | Giá trị |
|---|---|
| `messages` | `[system]` + lịch sử tối đa 12 cặp (user, assistant) + các message đang chờ của lượt này |
| `temperature` | **0.7**, hằng số trong code (`_DEFAULT_TEMPERATURE`, không đọc từ env) |
| `max_tokens` | **320** (`llm._MAX_OUTPUT_TOKENS`, dùng chung với đường Gemini để hai backend cùng ngân sách độ dài) |
| `stream` / `stream_options` | `true` / `{include_usage: true}` khi stream |
| `tools`, `tool_choice` | các tool của workflow + domain, `"auto"`, chỉ khi `LOCAL_LLM_TOOL_CALLING=1` và có tool |
| `options.num_ctx` | 4096 (khóa kiểu Ollama; vLLM không dùng, chưa kiểm tra có bị bỏ qua) |
| `continue_final_message`, `add_generation_prompt` | `true` / `false` **chỉ khi có bridge** (xem tác vụ 3): request kết thúc bằng một message `assistant` chứa câu bridge đang phát để model viết tiếp |

Timeout kết nối 2 s, đọc 15 s. Thinking **không** được tắt từ phía client (không có `chat_template_kwargs` trong body này); overlay ghi rằng server vLLM tự tắt bằng `--default-chat-template-kwargs` và phân tích tool call bằng `--tool-call-parser qwen3_xml`.

Message `user` mỗi lượt: lời người gọi (có thể có `[tone: X] ` ở đầu) + khối `\n\n[knowledge]\n{JSON}` (cắt tại `_INJECTED_LIMIT`; bỏ trường `instruction` vì chỉ dẫn đã nằm trong system prompt). Khối `[knowledge]` được cắt khỏi lịch sử ở các lượt sau.

### 5.4 Công cụ (tools) gửi kèm

Nếu agent bật RAG và có knowledge base (cổng `rag_enabled` + `knowledge_base_id`), hai tool được gắn (`ai/retrieval.py::build_kb_retrieval_pack`). Mô tả nguyên văn:

```text
search_knowledge_base — Search the agent's configured knowledge base for passages relevant to the caller's question. Call this whenever the caller asks about policies, products, procedures, or any company-specific information you are not certain about. Returns the source documents that ground your answer.
```

```text
(no-lookup declaration) — Declare that answering the caller's latest message needs NO company-specific information — for example greetings, thanks, goodbyes, yes/no confirmations, small talk, asking the caller to repeat, or collecting details for something you are already helping them with. Call this INSTEAD of search_knowledge_base, and ONLY when no fact about the company, its policies, products, pricing, hours, locations or procedures is needed. If in any doubt, call search_knowledge_base instead.
```

Khi `KB_CONTEXT_INJECTION=1` (đang bật), việc truy xuất được thực hiện **trước** lượt và kết quả chèn vào message user, nên không có vòng "bắt buộc chọn tool" (forced round). Các tool của workflow và domain (nếu có) được dịch từ khai báo hàm sang `tools` của OpenAI (`to_openai_tools`).

### 5.5 `prompt_addendum` của domain pack

Với agent có `domain` khớp một domain pack, `pack.prompt_addendum` được **nối vào cuối** system prompt (`orchestrator.py` khoảng dòng 3566). Có 5 pack trong `O/ai/domains/`: `fintech`, `healthcare`, `insurance`, `port_reception`, `real_estate`. `healthcare` **không có** addendum (file ghi rõ "no free-form tools or prompt addendum today"). Ví dụ `insurance` (nguyên văn):

```text
You answer questions about insurance policies and coverage. Listen for what the caller actually wants to know, then call the retrieve_policy_info tool with their question — include the policy type (auto, home, life, health) and the topic (coverage, deductible, exclusions, claims, renewal) when you can tell. Explain the retrieved information in one or two plain sentences. Only state what the retrieved passages support — never invent coverage terms, limits, or numbers. If nothing relevant is retrieved, say you are not certain.
```

Addendum của `fintech`, `real_estate`, `port_reception` nằm ở `_PROMPT_ADDENDUM` trong từng file tương ứng (chưa trích; cùng cấu trúc: mô tả việc cần làm + tool tương ứng + quy tắc không bịa số liệu).

**Về `ENABLE_PROMPT_VARIANTS=true` / `PROMPT_VARIANT_MODE=fixed_b`:** đây **không phải** prompt gửi cho LLM. `O/prompt_variants.py` chứa các câu hỏi thu thập dữ liệu **tĩnh** (ví dụ cho workflow `womens_health_intake`: "May I take your full name?" / bản B "To better understand your symptoms and situation, I'll ask you a few questions — could I start with your full name?" / bản C…), chọn theo chỉ số 0/1/2 (`fixed_a/b/c`). Chỉ là chọn câu nói ra, không đi qua model. `ENABLE_ACK_LAYER=true` cùng thuộc cơ chế này (lời xác nhận lấy từ Response Library).

**Chưa trích (❓):** prompt riêng của từng agent trong tenant-config (cơ sở dữ liệu, không nằm trong repo), và `_PROMPT_ADDENDUM` của `fintech`, `real_estate`, `port_reception`.

---

## 6. Tác vụ 5 — Field extraction (NuExtract-2.0-2B)

**Mục tiêu.** Trích các trường (tên, mã hồ sơ, số tiền…) từ câu người gọi, theo template JSON. **Thời điểm:** mỗi lượt trong workflow, phát ngay sau khi Decision trả về (`EXTRACTION_AFTER_DECISION=on`). `O/information_extraction/nuextractor.py`.

**Cách gọi:** `EXTRACTION_PROVIDER=nuextract`, `LOCAL_EXTRACTION_API=vllm` → **`/v1/completions`** (completion thô, **không** dùng chat endpoint, để tránh áp template chat lần hai). Model `numind/NuExtract-2.0-2B` (FP8), tên phục vụ `nuextract`, timeout 8 s. Khi lỗi, quy tắc nội bộ (in-process) trả lời thay và có circuit breaker (3 lỗi, mở 30 s), **không** quay về Gemini.

**Đặc điểm có chủ ý.** Template chỉ chứa **tên trường**, không kèm `extraction_hint`. Code ghi rõ đây là kết quả đo (2026-08-25): thêm hint kèm ví dụ làm model *bịa* giá trị lấy từ hint (2/9 dòng).

### 6.1 Prompt chính (`_prompt`, nguyên văn)

```text
<|im_start|>system
You are NuExtract, an information extraction tool created by NuMind.
Today's date is {today}.                                   ← chỉ khi workflow có trường ngày tương đối
The agent has just asked the caller about '{last_asked}'. If the caller's message is a bare answer, it refers to '{last_asked}'.   ← chỉ khi biết trường vừa hỏi
<|im_end|>
<|im_start|>user
# Template:
{
    "{field_id_1}": "verbatim-string",
    "{field_id_2}": "verbatim-string"
}
# Context:
{utterance}<|im_end|>
<|im_start|>assistant
```

Giá trị `"verbatim-string"` là chỉ dẫn "chép nguyên văn từ ngữ cảnh, không tự soạn".

### 6.2 Prompt thứ hai (`build_asked_field_prompt`)

Dùng khi lần đầu để trống trường vừa hỏi. Khung giống 6.1 nhưng `# Context:` trở thành hai dòng:

```text
Agent: What is your {last_asked với dấu _ đổi thành khoảng trắng}?
Caller: {utterance}
```

Chỉ lấy trường vừa hỏi từ kết quả này. Theo số đo trong code (gold set 246 dòng): tìm được trường vừa hỏi 78/83 so với 65/83 của prompt thường, nhưng bịa nhiều hơn nếu lấy cả các trường khác.

---

## 7. Tác vụ 6 — Grounded reply

**Mục tiêu.** Viết 1–2 câu đáp chỉ dựa trên các **sự kiện được cấp**. `compose_reply` ở `O/ai/grounded_reply.py`. **Dùng ở:** workflow `loan_status` (`_say`, tra trạng thái hồ sơ) và `hold_line` (câu giữ máy sau khi leo thang).

**Cách gọi:** `local_json_llm.complete`, một message `user`, `temperature 0.6`, `max_tokens 90` (mặc định) / `60` (hold), timeout 4 s (`GROUNDED_REPLY_TIMEOUT_S`). Lỗi trả `None` và nơi gọi dùng câu dự phòng.

### Template (nguyên văn)

`{persona}` mặc định: `You are a customer care AI assistant for a home loan lender, speaking with a customer on the phone.`

```text
{persona}

What you must do in your next reply: {task}

Facts you may use (use ONLY these; never invent names, dates, numbers or reasons that are not listed):
- {fact_key_1}: {fact_value_1}
- {fact_key_2}: {fact_value_2}

The caller just said: "{caller_said}"

Write only your reply, as it will be spoken aloud: one or two short, natural sentences, plain words, no lists, no markdown, no quotation marks.
```

(Không có sự kiện nào thì dòng facts là `- (none)`. Dấu `"` trong câu trả lời bị bỏ.)

### Các chuỗi `{task}` đã thấy trong code

| Ngữ cảnh | `{task}` | Câu dự phòng khi model lỗi |
|---|---|---|
| Hệ thống loan không truy cập được | `The loan system cannot be reached right now, so you cannot check the application. Apologise briefly and say so.` | `Sorry, I can't reach our loan system right now.` |
| Tìm thấy hồ sơ | `You have just looked up the caller's home loan application. Thank them by first name and tell them where the application is up to, in plain words. Stick to its current status; do not speculate about timing, reasons or next steps.` (facts gồm `application status in the loan system`, `in this status since`, …) | (xem `loan_status/conversation.py`) |
| Không tìm thấy lần 1 | `You found no application with that number under the caller's name. Apologise briefly, read back the number you heard, and ask them to give the application number again.` | `Sorry, I couldn't find that application. Could you give me the number again?` |
| Không tìm thấy lần 2 | `After a second try you still found no application with that number under the caller's name. Apologise and say a member of the team can look into it for them.` | `Sorry, I still can't find that application.` |
| Hold (`hold_line`, `max_tokens=60`, facts rỗng) | `You have already told the caller you are transferring them to a member of the team. Respond briefly and kindly to what they just said, and reassure them someone will be with them shortly. Do not start the conversation over.` | `Someone from our team will be with you shortly.` |

---

## 8. Câu bàn giao cố định khi leo thang (không phải LLM)

Để rõ ranh giới với tác vụ 2 và 6: khi chính sách leo thang kích hoạt, câu bàn giao là **hằng số** (`ai/escalation_policy.py::HANDOFF_LINE`), không qua model:

```text
I'm sorry about this. I'm putting you through to a member of our team now who can look into it for you. Please stay on the line.
```

Ghi chú trong code nói câu cố định được chọn vì "nothing rings a person yet" và bản do model sinh trước đây đã hứa chuyển máy. Tuy vậy nội dung câu hiện tại vẫn nói "putting you through … now". Điều này trái với quy tắc RP4 của spec (`turn-pipeline.yaml`): không hứa chuyển máy, không hứa kết nối với người, không hứa gọi lại. Cần xác nhận hành vi mong muốn.

---

## 9. Tác vụ 7 — Sentiment (kênh text)

**Mục tiêu.** Phân loại cảm xúc từ lời người gọi, ra JSON sáu nhãn `neutral, happy, frustrated, angry, distressed, confused`. **Thời điểm:** chạy nền, bắt đầu sau khi reply sinh xong (`TEXT_EMOTION_AFTER_AUDIO=reply_done`) để không làm chậm câu trả lời. `O/ai/emotion.py` (prompt) và `O/ai/emotion_sensevoice.py` (gọi Qwen).

**Cách gọi:** `/v1/chat/completions`, hai message `system` + `user`, `temperature 0.0`, `max_tokens 200`, `response_format: {type: json_object}`, `chat_template_kwargs: {enable_thinking: false}` (nếu server trả 400 thì thử lại không có tham số này).

**Cách dùng kết quả.** Kết quả text được hợp nhất với kênh audio (SenseVoice, **không phải LLM**; trọng số audio 0,3) rồi làm mượt EMA 0,4. Marker `[tone: X] ` được **chèn ở đầu** transcript của một lượt sau, chỉ khi nhãn trội khác `neutral` và điểm ≥ 0,55. Kết quả này còn dùng cho chính sách leo thang (ngưỡng 0,8).

### Message `system` (nguyên văn)

```text
You are a sentiment classifier for a customer-support phone transcript. Read the words the caller said (NOT prosody — you only see text) and return a JSON object mapping each label to a confidence in [0, 1]. Values must sum to approximately 1. Labels: neutral, happy, frustrated, angry, distressed, confused. If the text is empty, very short, or you genuinely can't tell, return neutral=1.0 and all others 0.
```

### Message `user`

`{transcript!r}` là biểu diễn `repr` của chuỗi Python (có dấu nháy).

```text
Caller said: {transcript!r}

Classify the sentiment. Respond ONLY with the JSON object — no preamble, no explanation, no markdown fences.
```

> Cùng file còn có prompt cho phân loại cảm xúc **qua audio bằng Gemini** (`_SYSTEM_PROMPT`, `_USER_PROMPT_AUDIO`). Với `EMOTION_PROVIDER=sensevoice`, kênh audio dùng SenseVoice nên prompt này không được dùng.

---

## 10. Tác vụ 8 và 9 — Call purpose và Post-call analysis

### 10.1 Tác vụ 8 — Call purpose

**Mục tiêu.** Xác định mục đích cuộc gọi, không ảnh hưởng câu trả lời. **Thời điểm:** mỗi 4 lượt người gọi (`_REFRESH_EVERY_CALLER_TURNS = 4`), lần đầu từ lượt thứ 2, nên rơi vào các lượt **2, 6, 10, 14…** ✅. `O/ai/caller_purpose.py`.

**Cách gọi:** `GeminiCaller` → (do `LOCAL_JSON_LLM_BASE_URL` được đặt) Qwen, `json_mode`, `temperature 0.0`, `max_output_tokens 120`, timeout 8 s. Prompt đi như **một message duy nhất**. Mặc định trong code là `gemini-3.1-flash-lite` (`CALLER_PURPOSE_MODEL`).

**Lắp ráp prompt** (`build_prompt`): `{_TASK}` + dòng trống + `{contract}` + dòng trống + `Transcript so far:` + transcript. `contract` là `_FIRST_CONTRACT` nếu chưa có mục đích hợp lệ, ngược lại là `_UPDATE_CONTRACT` (với `__CURRENT__` thay bằng mục đích đã ghi). Transcript gồm các dòng `Caller: …` / `Agent: …`, hoặc `(no turns yet)`.

**`_TASK` (nguyên văn):**

```text
You read a PARTIAL call transcript between a contact-center voice agent and a
caller. The call is still in progress.

Your subject is the caller's OVERALL REASON FOR CALLING — the answer to "why did
this person pick up the phone?" It is not a summary of what has happened so far,
and it is not a routing category.

Name the subject matter concretely: what kind of claim, which product, what
problem, whose account.

Base it ONLY on the transcript. Never invent names, numbers, or outcomes. Write it
in the transcript's dominant language.
```

**`_FIRST_CONTRACT` (nguyên văn, lần trích đầu):**

```text
No purpose has been recorded for this call yet.

Return ONE JSON object: {"purpose": "..."}

At the start of a call the caller's FIRST clear request is the reason for the call
— record it as soon as they have made it. Only if they have not yet said what they
want, return {"purpose": ""}: an empty string is the correct answer for a greeting
or a false start, but NEVER output vague filler such as "General inquiry",
"Customer support", "Question", or "Unknown".

Return ONLY the JSON object.
```

**`_UPDATE_CONTRACT` (nguyên văn, các lần cập nhật):**

```text
THE DISTINCTION THAT MATTERS NOW. The reason for a call is NOT the topic of its
latest exchange. Once a caller has said why they called, the questions they go on
to ask ABOUT that same matter belong to it and do not replace it: someone who
called to file a claim and later asks how long it will take still called to file a
claim.

This call has ALREADY been recorded as being about:

    "__CURRENT__"

Write the best statement of the caller's overall reason for calling that the
transcript now supports, then say how it relates to the recorded one:

{"purpose": "...", "relation": "same" | "refines" | "changed"}

  "same"    — the recorded purpose already says why this person called, as well as
              the transcript supports. Use it when the recent transcript is a
              follow-up question, a detail of a step already under way, a
              confirmation, or a digression WITHIN the recorded purpose. On a long
              call this is the usual answer. Repeat the recorded purpose.
  "refines" — the reason is the same, but it can now be stated MORE COMPLETELY or
              more correctly: the recorded wording is vague, is missing the subject
              matter, or was a first guess the transcript has since corrected.
  "changed" — the reason for the call has genuinely changed, or has grown to include
              a further goal the recorded purpose does not cover. Then "purpose"
              must state the call's overall reason, covering what the caller
              actually rang for.

Worked examples:

  recorded: "Filing a new insurance claim"
  the transcript now also shows the accident was a rear-end collision
  -> {"purpose": "Filing a new insurance claim for a rear-end car accident",
      "relation": "refines"}    — the same reason, stated more completely

  recorded: "Filing a new insurance claim for a rear-end car accident"
  the caller now asks how long a claim decision takes
  -> {"purpose": "Filing a new insurance claim for a rear-end car accident",
      "relation": "same"}       — a question INSIDE the reason already recorded

  recorded: "Asking for the company's mailing address"
  the caller now also wants to open a claim
  -> {"purpose": "Getting the company's mailing address and filing a new claim",
      "relation": "changed"}    — a second goal the record did not cover

The question is never "is more detail available?" — it is "what is that detail
ABOUT?" Detail about the REASON belongs in the purpose ("a claim" becomes "a
claim for a rear-end collision"). Detail about a MOMENT in the call does not
("filing a claim" must NOT become "asking how long the claim will take") — that is
a question inside the reason, and its answer is "same". Narrowing the purpose onto
whatever was discussed most recently is the specific mistake to avoid.

Return ONLY the JSON object.
```

**Cách áp dụng kết quả** (`adopt`, không nằm trong prompt): kết quả không dùng được (rỗng, quá chung chung, lỗi) không bao giờ thay một mục đích đã có; chưa có gì thì lấy luôn; chỉ `refines` hoặc `changed` mới được thay; mặc định là giữ nguyên.

### 10.2 Tác vụ 9 — Post-call analysis

**Mục tiêu.** Sinh JSON tổng kết khi cuộc gọi kết thúc. `O/ai/post_call_analysis.py`.

**Cách gọi:** `GeminiCaller` → Qwen (khi có `LOCAL_JSON_LLM_BASE_URL`), `json_mode`, `temperature 0.2`, `max_output_tokens 600`, `thinking_budget 0`, chạy trong thread riêng. Model: `CONVERSATION_INTELLIGENCE_MODEL` hoặc `GEMINI_MODEL` (mặc định `gemini-3.5-flash`) nếu quay về Gemini. Prompt là **một chuỗi**: `_SYSTEM_PROMPT` + dòng trống + khối transcript/signals.

**`_SYSTEM_PROMPT` (nguyên văn):**

```text
You are a post-call Conversation Intelligence analyst for a contact-center. You
receive a COMPLETED, PII-masked call transcript plus structured call signals.
Produce ONE JSON object that reads like an executive call review written by an
experienced operations analyst — something a supervisor can skim in five seconds
and that analytics/search can aggregate. It must NOT read like a transcript dump.

Base EVERY statement ONLY on the provided transcript and signals — never invent
facts, names, numbers, or resolutions. Always prefer the MOST SPECIFIC wording
the transcript supports; use a vaguer phrase only when nothing specific exists.
Write in the transcript's dominant language. Return ONLY the JSON object, no
prose outside it.

Fields:
- "purpose": ONE concise business statement of the REASON for the call — it
  answers "What was this call about?" (the customer's goal / subject), NOT the
  play-by-play. e.g. "Customer asked whether the clinic is open on weekends." NOT
  a routing category. NEVER output vague fillers such as "General inquiry",
  "Customer called for assistance", "Customer support", "Question", "Unknown", or
  "Intent unavailable". If nothing is discernible, return "" (empty).
- "summary": MAXIMUM three short operational sentences, written like an
  experienced operations supervisor logging the call. It answers "What HAPPENED
  during the call?" — a DIFFERENT question from purpose, so do NOT repeat the
  wording used in purpose. Cover: (1) what the assistant did, (2) how the
  conversation progressed, (3) how it ended. Do NOT replay the dialogue
  turn-by-turn; drop minor back-and-forth; no bullets, metrics, jargon, or quotes.
  GOOD (does not restate the purpose): "The assistant asked which clinic the
  caller meant, since multiple clinics were available. The caller did not specify
  one and ended shortly after. No clinic hours were provided."
  BAD (storytelling — forbidden): sentences that open with "The customer
  initiated...", "The AI assistant then...", or "The customer then...". Never
  narrate the conversation as a story; state what happened.
- "customerExperience": ONE short supervisor phrase on the caller's emotional arc,
  drawn from the signals (emotion timeline, frustration peak, sentiment trend,
  escalation) — e.g. "Calm throughout the conversation.", "Mild frustration during
  verification.", "Positive interaction.", or "Customer became frustrated during
  identity verification." A single tight sentence only; do not narrate.
- "actionTaken": a short phrase naming the assistant's PRIMARY, SPECIFIC action —
  bias toward what it actually DID, not a generic category. Prefer precise phrasing
  such as "Requested clinic clarification", "Performed knowledge lookup", "Verified
  customer identity", "Collected insurance information", "Scheduled appointment",
  or "Escalated to supervisor". AVOID generic phrases like "Collected intake
  information", "Handled customer request", or "Processed inquiry". If the
  assistant effectively did nothing, use "None". This is the ACTION, distinct from
  the outcome.
- "keyOutcome": ONE short sentence naming the actual BUSINESS result — what
  HAPPENED, not merely whether it succeeded. Prefer specifics like "Customer ended
  before identifying the clinic", "Knowledge request answered", "Appointment
  successfully scheduled", "Transferred to supervisor", or "Insurance claim
  created". AVOID generic fillers such as "Call completed", "Conversation ended",
  "No resolution reached", or "Resolved" unless absolutely unavoidable; never
  output "Unknown".
- "tags": 3-6 short Title-Case business-topic tags describing the domain and
  subject (e.g. ["Healthcare", "Knowledge", "Clinic Hours"]). Tags describe
  TOPICS, never routing methods or sentiment. No duplicates, no PII.

Output JSON shape:
{"purpose": "...", "summary": "...", "customerExperience": "...",
 "actionTaken": "...", "keyOutcome": "...", "tags": ["...", "..."]}
```

**Khối dữ liệu nối sau** (`build_prompt`):

```text
TRANSCRIPT (masked):
  [{speaker}] {text}  (tools: {tool1, tool2})
  ...

SIGNALS:
  workflow_executed: {workflow | none}
  tools_invoked: [...]
  knowledge_lookups: [...]
  extracted_fields: {...}
  duration_seconds: {n}
  turn_count: {n}
  sentiment_trend: [...]
  frustration_peak: {x}
```

---

## 11. Phần chưa trích hoặc chưa xác minh

| Nội dung | Ghi chú |
|---|---|
| Prompt riêng của từng **agent** trên staging | Lưu trong tenant-config (cơ sở dữ liệu), không nằm trong repo; tài liệu chỉ có mẫu mặc định (mục 5.1) |
| `_PROMPT_ADDENDUM` của `fintech`, `real_estate`, `port_reception` | Chưa trích; chỉ trích `insurance` làm ví dụ, `healthcare` không có addendum (mục 5.5) |
| `options.num_ctx` trong body của Reply | Khóa kiểu Ollama được gửi cho vLLM; chưa kiểm tra vLLM bỏ qua hay báo lỗi |
| Nhánh nào gọi reply LLM ngoài `answer` | Chú thích trong `capability.py` cho thấy các kind khác cũng dùng "conversational default"; chưa theo dấu đầy đủ |
| Prompt của kênh audio Gemini (`_SYSTEM_PROMPT`, `_USER_PROMPT_AUDIO` trong `ai/emotion.py`) | Không dùng khi `EMOTION_PROVIDER=sensevoice`; không trích |
| Đường gọi của `GeminiCaller` khi dùng Qwen | Đã xác nhận `local_json_llm.complete` gửi prompt như một message `user`. Chưa xác minh từng bước định tuyến trong `gemini_json.py` |
| Các tác vụ LLM khác ngoài 9 tác vụ | KB step validation, intent fallback, publish wording (cùng `LOCAL_JSON_LLM_*`); truy vấn embedding RAG (Qwen3-Embedding, không phải LLM sinh văn bản); speculation (`ai/speculation*.py`) |

---

## 12. Nhận xét tổng hợp

- **Một máy chủ vLLM dùng chung (suy luận từ cấu hình, chưa đo).** Decision, anger judge, bridge, reply, sentiment, grounded reply và các tác vụ nền cùng trỏ `vllm_chat_qwen_3_6`. Trong một lượt có thể có 3–4 yêu cầu gần đồng thời; nên đo độ trễ khi tranh chấp GPU trước khi tăng số cuộc gọi đồng thời (đã có thí nghiệm `wave2b/2c` trong `artifacts/eval/`).
- **Hầu hết tác vụ có đường lùi an toàn** (trả `None` hoặc câu dự phòng). Ngoại lệ là latch leo thang: không có cách reset nên một kết quả dương tính giả giữ cuộc gọi ở trạng thái hold đến hết.
- **Điểm cần quyết định:** (1) nội dung `HANDOFF_LINE` so với RP4; (2) bridge có prompt "1–4 từ" nhưng validator cho 5 từ; (3) latch leo thang không đảo ngược được.
- **Thứ tự thời gian trong một lượt (suy ra từ code):**
  ```text
  STT segment final (EARLY_LANE=on)
     ├─ Bridge        (reply-LLM, chờ ≤700 ms)
     ├─ Decision      (Qwen, 11 kind)
     ├─ Anger judge   (Qwen, 2 s, song song Decision)
     │      └─ "yes" hoặc emotion ≥0,8: HANDOFF_LINE cố định + vào handoff queue + chốt "hold"
     └─ Field extraction (NuExtract 2B, ngay sau Decision, chỉ khi có workflow)
  → phát bridge → Reply (nếu cần LLM)
  Sau khi reply sinh xong: Sentiment text (nền) → [tone: X] cho lượt sau
  Mỗi 4 lượt người gọi:     Call purpose (nền)
  Khi cuộc gọi kết thúc:    Post-call analysis
  Trong workflow loan_status / hold: Grounded reply (Qwen, ≤4 s, có câu dự phòng)
  ```
