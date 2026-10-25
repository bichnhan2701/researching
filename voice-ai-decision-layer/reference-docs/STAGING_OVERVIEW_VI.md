# Tài liệu mô tả hệ thống 

> Tài liệu này mô tả nội dung thư mục `staging/` (nền tảng **Enterprise AI Voice Agentic BPO**).
> Được tổng hợp từ việc đọc cấu trúc thư mục và các tài liệu chính (README, CLAUDE.md, DEPLOYMENT.md,
> các file compose, `.ai-first/project-context.yaml`, README/CHANGELOG của từng service, spec, infra,
> selfhosted-ai…). Thư mục `staging/` **không bị sửa đổi** trong quá trình viết tài liệu.
>
> **Lưu ý về phạm vi đọc:** `staging/` nặng ~4,6 GB với hàng chục nghìn file (riêng `artifacts/` ≈ 4.000 file,
> `services/` ≈ 4.700 file, `frontend/` ≈ 2.000 file). Tài liệu này dựa trên việc đọc kỹ các file tổng quan
> và quét cấu trúc/đầu file của phần còn lại; không đọc từng dòng của mọi file (ví dụ log, dữ liệu eval, mã nguồn chi tiết).
> Các file `.env*` chứa bí mật **không được mở** — chỉ ghi nhận sự tồn tại.

---

## 1. Tổng quan

**Mục đích:** hệ thống microservice voice-AI trả lời/gọi ra điện thoại qua tổng đài **Asterisk PBX**, chạy vòng lặp
**STT → LLM → TTS** cho mỗi cuộc gọi, kèm dashboard quản trị/vận hành (quản lý agent, campaign, knowledge base,
workflow, live-ops, override/take-over của người thật, analytics…).

**Đặc điểm đáng chú ý:**
- Dự án được **xây dựng bằng quy trình AI agent** (một orchestrator + bốn agent, làm việc theo artifact), nên repo chứa
  cả **mã chạy thật** lẫn **các artifact sinh ra/điều khiển quá trình sinh mã** (spec, plan, eval, log…).
- **Spec là nguồn sự thật duy nhất**: `artifacts/spec/` (system/api/data/constraints/usecases…). Khi code và spec mâu thuẫn, spec thắng
  (theo `CLAUDE.md`; thứ tự ưu tiên đầy đủ: code đang chạy + test pass → spec → operating-guide → CLAUDE.md → README/design → tài liệu lịch sử).
- Trạng thái triển khai: `.deployed_commit` ghi `9cbb5019… 20261019T091247Z (from d5ffd43a; scenario2 + 7 dev merges)`.
  Repo `staging/` có `.git` riêng; remote GitLab nội bộ `ai_voice_agentic_bpo` (TMA – AI Center R&D).
- `README.md` gốc phần lớn vẫn là **template GitLab mặc định**; chỉ có thêm thông tin đăng nhập demo
  (`admin@demo.local`, mật khẩu = `BOOTSTRAP_ADMIN_PASSWORD` trong `.env`) và link mockup dashboard.

---

## 2. Kiến trúc runtime

### 2.1 Voice plane (đường dữ liệu cuộc gọi)

```
PSTN/SIP ──> Asterisk PBX ──AudioSocket TCP :9092──> (Orchestrator / Session_Manager)
                                   │
        per-call:  Audio ─> STT (Deepgram Nova-3 | Nemotron self-host)
                          ─> LLM (Gemini Flash | vLLM Qwen self-host)
                          ─> TTS (Google WaveNet/Chirp | Kokoro | OmniVoice | …)
                          ─> audio trả về Asterisk
                   song song: phân tích cảm xúc, PII masking, knowledge/RAG, CRM/MCP tools
```

- Spec (v1.8.0) mô tả mô hình mục tiêu: `Session_Manager` (admission + routing, kết thúc TCP 9092) → gRPC
  `voicebpo.internal.voice.v1` (cổng 9093) → `Voice_Worker` (task graph cho mỗi session, K=25–50 phiên đồng thời).
- Bản **đang chạy trong `services/`** là `voice-ai-orchestrator` — **một tiến trình Python asyncio** đảm nhiệm cả
  AudioSocket server lẫn pipeline AI cho mỗi cuộc gọi (README của nó nêu rõ: thay thế `audiosocket-server` cũ).
- Cổng quan trọng: AudioSocket `9092`/`9093`, management API của orchestrator `8081` (host `8089`).

### 2.2 Control/Data plane (microservice)

`API_Gateway` (NestJS, JWT + RBAC, điểm vào duy nhất có xác thực) định tuyến tới:
`Tenant_Config_Service`, `Campaign_Service`, `Audit_Service`, `HandoffQueue_Service`, `Document_Ingestion_Service`,
và chuyển các đọc `/api/v{MAJOR}/{plural}/view` sang `View_API_Service`.

**Bất biến "sole-writer" (không được vi phạm):**
- `Session_Service` là nơi duy nhất ghi Redis.
- `Conversation_Service` là nơi duy nhất ghi Mongo *System* (call session, transcript).
- `Tenant_Config_Service` là nơi duy nhất ghi Mongo *User* (org, agent, `DidMap`).

### 2.3 Frontend

Vite + React 18 + TypeScript. Luồng dữ liệu: `transportClient` → `unwrapEnvelope` (`{data, meta, error}`) →
`dtoFromServer` (snake→camel) → `validate*ViewDto` (Zod) → `map*ViewDtoToVm` → React Query.
Quy ước mọi endpoint đọc: `/api/v{MAJOR}/{plural_resource}/view`, body snake_case.

---

## 3. Hai cây mã nguồn — cần phân biệt

| Cây | Tính chất | Nội dung |
|---|---|---|
| `services/` + `frontend/` | Do con người duy trì (runtime thực) | `voice-ai-orchestrator` (Python), `view-api-service` (Node), `mock-crm-server`, `mcp-tools`, `response-manager`, `response-store` (Python), `Django-CRM` (BottleCRM), React `frontend` |
| `artifacts/implementation/` | Do AI agent sinh ra | 20 service backend (NestJS/Python) + 7 adapter tích hợp + 4 mục infra + 3 thư viện dùng chung. `docker-compose.yml` build nhiều service (api-gateway, tenant-config, session, conversation, audit, handoff-queue, campaign, document-ingestion, document-processor, stt/tts/llm-engine…) từ cây này |

Quy tắc: sửa ở đúng cây chứa service; không sao chép thay đổi sang cả hai trừ khi được yêu cầu.

---

## 4. Cấu trúc thư mục `staging/`

| Đường dẫn | Dung lượng / số file | Nội dung |
|---|---|---|
| `services/` | 88 MB / ~4.700 | Các service runtime viết tay (mục 5) |
| `frontend/` | 25 MB / ~2.000 | Dashboard React (mục 6) |
| `artifacts/` | 165 MB / ~3.970 | Spec, design, requirement, planning, eval, governance, implementation do agent sinh (mục 7) |
| `infra/` | 1,6 MB / ~190 | Asterisk PBX test, WireGuard, monitoring, HTTPS, self-hosted voice, patch PBX (mục 8) |
| `selfhosted-ai/` | **4,2 GB** / 18 | Compose cho từng model AI chạy tại chỗ (mục 9); phần lớn dung lượng là `extractor/` (model + llama-server) |
| `docs/` | 3,8 MB / ~197 | `archive/` (tài liệu lịch sử), `superpowers/plans/` (kế hoạch live-transcript PII captions) |
| `scripts/` | 0,6 MB / 50 | Script demo, deploy, dev, did-map, ops, mip4/mip5, smoke test (mục 10) |
| `config/` | 3 file | `asterisk_instances{,.dev,.test}.yaml` — cấu hình các instance Asterisk |
| `deploy/` | 4 file | `apache`, `docker`, `observability` (gen-env cho Langfuse) |
| `experiments/` | 1,3 MB / 69 | `conversational_brain/` và `gemini-live-spike/` — thí nghiệm, kết quả (mục 11) |
| `backlog/` | 5 file | Kế hoạch chưa làm: multi-accent English, domain-capability convergence, voice catalog (refactor + implementation), code-map tooling pilot |
| `logs/orchestrator/` | 13 file, 632 KB | Log JSON của orchestrator (08/10 → 21/10/2026) |
| `runtime-forensics-2026-06-23/` | 7 file | Bằng chứng điều tra runtime ngày 23/06/2026 (session doc, turns, audit events, tool invocations, callbacks…) |
| `.ai-first/` | 1 file | `project-context.yaml` — metadata máy đọc được cho `af` |
| `.claude/`, `.claude_history/` | — | Skill `gb10-latency-ab`; 4 tài liệu phân tích lịch sử (permission-interrupt, latency-observability, integrations-runtime-verification, platform-maturity) — **lịch sử, không có thẩm quyền** |
| `.git/` | — | Repo git của staging |

**File ở gốc:**

| File | Vai trò |
|---|---|
| `CLAUDE.md` | Hướng dẫn cho Claude Code + quy tắc quy trình 4-agent (xem mục 12) |
| `DEPLOYMENT.md` | Hướng dẫn triển khai nhanh, migrate subnet, bật auth cho data store, auth gate dashboard |
| `README.md` | Template GitLab + thông tin đăng nhập demo |
| `af` | CLI "AI-First" (bash): `verify`, `review`, `logs`, `deploy`, `status`, `health` |
| `quick-start.sh` | Chạy cụm "host-run" ở dải cổng 34xxx (api-gateway 34000, view-api 34001, tenant-config 34003, handoff-queue 34006, orchestrator 38081, frontend 34173) |
| `caller-purpose-summarization-investigation.md` | Điều tra (2026-09-28, DRAFT) về tác vụ tóm tắt mục đích người gọi → `conversationIntelligence.purpose` |
| `docker-compose*.yml` | Các stack Docker (mục 13) |
| `.env`, `.env.bak.*`, `.env.*.example` | Cấu hình/biến môi trường (xem mục 14 về bí mật) |
| `llama-server-cuda.tar.gz` | File 20 byte ở gốc (có vẻ là placeholder/con trỏ; bản thật nằm ở `selfhosted-ai/extractor/`) |
| `.backpassrc.json`, `.dockerignore`, `.gitignore` | Cấu hình công cụ (backpass: memory file `CLAUDE.md`, skills dir `.claude/skills`…) |

---

## 5. Các service runtime (`services/`)

| Service | Ngôn ngữ | Mô tả |
|---|---|---|
| **voice-ai-orchestrator** | Python (asyncio, FastAPI mgmt) | Lõi voice: AudioSocket TCP, connection manager (registry Asterisk + allow-list IP + health check), pipeline AI mỗi cuộc gọi. Rất lớn, nhiều module (bên dưới) |
| **view-api-service** | Node 20, `.mjs`, không phụ thuộc | Backend đọc-tổng-hợp sở hữu mọi endpoint `/api/v1/.../view` (dashboard, agents, campaigns, queues, integrations, analytics, transcripts, override, knowledge, org-settings, pii-rules, voice…). Đổi tên từ `dashboard-view-service` (2026-05-24). Có dữ liệu seed thực tế, trả envelope `{data,meta,error}` |
| **mock-crm-server** | Python (FastAPI) | CRM giả (`:8090`) với router `contacts`, `calendar`, có seed |
| **mcp-tools** | Python | Các adapter MCP: `appointment_scheduler`, `case_ticket_creator`, `notification_dispatcher`, `workflow_router` |
| **response-manager** | Python (uv) | "Response Library" (W6) — CMS cho nội dung giao tiếp tái sử dụng: Response → Variant → Voice Asset; vòng đời Draft→Review→Approved→Published→Deprecated→Archived; không resolve/phát/sinh audio |
| **response-store** | Python (uv) | Kho lưu cho response library (có Dockerfile, được compose build) |
| **Django-CRM** | Django 6 + SvelteKit + Flutter | **BottleCRM** — CRM mã nguồn mở multi-tenant (PostgreSQL RLS), tích hợp làm CRM thật (client `bottlecrm_client.py` trong orchestrator). Có tài liệu riêng (README, DESIGN_SYSTEM, RLS_SETUP, CONTRIBUTING, mkdocs) |

### 5.1 Các module chính của `voice-ai-orchestrator/src/voice_ai_orchestrator/`

- `audiosocket/`, `connection_manager/`, `ami/`, `admission/`, `browser_voice/` — vào/ra audio, quản lý PBX, gọi thử từ trình duyệt.
- `ai/` — trái tim pipeline: `orchestrator.py`, `stt*.py` (Deepgram / self-host), `tts*.py` (Google, Kokoro, OmniVoice, Chatterbox, Qwen3, Deepgram, local HTTP), `llm*.py`, `emotion*.py` (Gemini / SenseVoice), `bridge*.py`, `speculation*.py` (suy đoán giảm độ trễ), `echo_guard`, `sentence_chunker`, `escalation_policy`, `risk_tracker`, `post_call_analysis`, `caller_purpose`, `usage/pricing`, `observability/`, `domains/`, v.v.
- `conversation_runtime/` (kèm `gemini_live/`), `conversation_orchestrator/`, `conversation_composition/`, `conversation_integration/` — runtime hội thoại.
- `response/` — pipeline phản hồi (resolver, playback, wording adapter, acknowledgement, telemetry).
- `knowledge/` — RAG: answerer, intent gate, rerank, local embedder, providers, catalog, governance.
- `workflows/`, `intake/`, `appointment_booking/`, `information_extraction/`, `demo_faq/` — workflow, thu thập thông tin, đặt lịch, trích xuất trường.
- `pii/` — che PII (log, hiển thị, theo luật của agent); `captions/` — phụ đề trực tiếp.
- `handoff/`, `management/` (API `/health`, `/instances`, `/sessions`, `/config/reload`, call control, knowledge search), `session_reconciler/` (đóng cuộc gọi "treo" với lý do `ABANDONED_RECONCILED`).
- `clients/` — CRM (BottleCRM, mock), tenant-config, response-store, loan application…
- `persistence/`, `audio_audit/`, `runtime/runtime_config.py`, `voice_assets/`, `capabilities/`, `prompts/`.
- Thư mục khác: `assets/` (decision, fillers), `voice_library/`, `config/pricing`, `docs/adr`, `spikes/` (rất nhiều script thí nghiệm độ trễ/hành vi), `tests/`, `scripts/`, `recordings/`, `store/`.
- `CHANGELOG.md` viết theo góc nhìn nghiệp vụ; mục đáng chú ý: session reconciler (mặc định bật, tắt bằng `SESSION_RECONCILER_ENABLED=false`), trích xuất trường bằng model self-host NuExtract/Qwen (mặc định tắt, `SH_EXTRACTION_PROVIDER`).

**Cảm xúc (emotion):** phân loại song song theo từng câu — audio + text (6 nhãn: neutral, happy, frustrated, angry, distressed, confused) → hợp nhất theo trọng số (`EMOTION_AUDIO_WEIGHT`=0.3) → làm mượt EMA (`EMOTION_SMOOTHING_ALPHA`=0.4) → nếu vượt ngưỡng 0.55 thì chèn marker `[tone: X]` vào transcript cho LLM (agent không được nói ra marker).

---

## 6. Frontend (`frontend/`)

- Stack: React 18, react-router 6, React Query 5, Zustand, react-hook-form + Zod, Radix Select, Vite 5, Vitest (+ Playwright).
- Script: `dev`, `build`, `typecheck`, `test` (offline, loại `*.live.test.ts`), `test:live` (cần `view-api-service` ở `:3001`).
- `src/app/` (shell, router, providers, policies), `src/shared/` (transport/api/auth), `src/features/` gồm:
  `access-role`, `agent`, `agent-extension`, `analytics`, `auth`, `browser-takeover`, `callSession`, `campaign`, `capability-package`, `credentials`, `dashboard`, `developer-tools`, `developer-voice`, `entitlements`, `human-queues`, `integrations`, `kb-documents`, `kb-search`, `knowledge`, `knowledge-base`, `live-ops`, `organization`, `org-settings`, `override`, `pii-rules`, `role`, `runtime-control`, `transcript`, `user-mgmt`, `voice-assets`, `workflow`.
  Ngoài ra các thư mục theo milestone/chủ đề: `enablement`, `explainability`, `focus`, `governance`, `integration`, `operability`, `product`, `readiness`, `rehearsal`, `review-session`, `staging`, `workspace`…
- `server/authGate.ts`: cổng xác thực đặt trước proxy của Vite (dashboard `:5173`). Chỉ cho 4 endpoint runtime (`GET/PUT /runtime-config`, `POST /dev-tts`, WS `/dev-call`, `/take-over-call`); mọi thứ khác dưới `/runtime-api` bị từ chối kể cả với admin; **container dashboard không giữ `JWT_SECRET`** — hỏi gateway (`/api/v1/auth/permissions`), lỗi gateway → 503 (fail-closed).
- `CHANGELOG.md` (hướng người dùng): knowledge base dạng thẻ + popup (Configuration/Files/Retrieval testing/Logs), xem chunk đã embed, quản lý Role (RBAC chi tiết, gợi ý "Recommended"), toggle 2-way communication cho campaign, ẩn nút theo quyền, xác nhận đăng xuất, cảnh báo xoá KB đang dùng…

---

## 7. `artifacts/` — sản phẩm của quy trình AI-first

Luồng: `Requirement → Planning → Design → Implementation → Eval → Review → Log`.

| Thư mục | Nội dung |
|---|---|
| `spec/` | **Nguồn sự thật có cấu trúc**: `system.yaml` (v1.8.0, 2026-10-16), `api.yaml`, `data.yaml`, `constraints.yaml`, `usecases.yaml`; thư mục con `api/` (API theo tính năng + `view-api-convention.yaml`), `constraints/` (ai, api, ci_cd, data, frontend, integration, realtime, rpc, security), `integrations/` (CRM), `orchestrator/` (turn-pipeline, decision-grammar-v2, fast-reply, bridge-late, handoff-listen, reply-audio, stt-early-final, turn-latency, llm-usage…), `ui/` |
| `schema/` | JSON schema cho `ai_plan`, `api`, `data`, `system`, `usecases` |
| `requirement/` | PRD (`PRD_Claude_Ver.md` là bản chuẩn; bản `_Gemini_Ver` không còn hiệu lực), use case M1–M3, các tài liệu UX "generated outputs", yêu cầu ticket (Asterisk 1.1, mock CRM 3.1.0, MCP tools 3.2.0), fast-reply & voice-latency demo 2026-10-13 |
| `planning/` | `ai_plan.json`, `task_state.json`, epics/tickets M1, kế hoạch ticket, deferred tasks, báo cáo toàn vẹn kế hoạch, phạm vi conversational-brain MIP4/MIP5 |
| `design/` | HLD (`HLD_Claude_Ver.md` chuẩn), AI models architecture, Asterisk integration HLD, API contracts/versioning, data models, internal RPC, realtime protocols, OpenAPI, ADR-010, frontend flows… |
| `architecture/` | Tài liệu ràng buộc nhiều service: ADR, thiết kế response-manager/response-library, conversation runtime, `voice-catalog-final.md`, `deployment-readiness-report.md`, `deployment-acceptance-report.md` (bằng chứng nghiệm thu, verdict GO), latency, operations… |
| `governance/` | doc-lifecycle (CLEANUP_AUTHORIZATION), frontend, orchestration, schema |
| `eval/` | **Lớn nhất (~2.370 file, 125 MB)**: báo cáo audit/điều tra/đo đạc (độ trễ GB10, TTS, wave2–wave5 sinh nội dung cục bộ, intent classifier, understanding layer, demo truthfulness, concurrent calls readiness, auth audit…), benchmark JSON/JSONL, ảnh chụp màn hình live, mẫu accent… |
| `implementation/` | Mã do agent sinh: `backend/` (20 service: api-gateway, audit, campaign, conversation, document-ingestion, document-processor, emotion-classifier, handoff-queue, llm-engine, pii-masking-engine, rag-search-engine, session-manager, session-service, stt-engine, tenant-config, tts-engine, voice-ai-orchestrator-core, voice-biometric-auth, voice-cloning, voice-worker), `integration/` (ccaas-bridge, crm-erp-gateway, mcp-appointment/case-ticket/notification/workflow, outbound-dialer), `infra/` (asterisk, mongodb, redis, vectordb), `shared/` (auth-utils, contracts, error-model) |
| `observation/` | Thiết kế "observation as runtime hook" (ADR-001, WM-06, schema, architecture review) |
| `runtime/` | adapters, capability-mining, execution-events/lifecycle/unit, permission-intelligence, validation… |
| `state/` | `orchestration-lineage.yaml` |
| `log/` | Schema log thống nhất (`unified_log`, `execution_log`, `attempt_log`, `cost_tracking`, …) + `VALIDATION.md` |
| `infra/`, `framework-extraction/`, `research/`, `review/` | Phụ trợ (research/review hiện rỗng) |

---

## 8. Hạ tầng (`infra/`)

- `asterisk_pbx/asterisk_docker_test` — PBX Asterisk dạng Docker để thử nghiệm (≈125 file).
- `pbx/` — các bản patch cấu hình cho PBX thật (`extensions.conf`, `manager.conf`, `pjsip.conf`).
- `wireguard/` — cấu hình tunnel orchestrator ↔ PBX (`10.99.0.x`). PBX live: `11.11.7.18` (nội bộ) / `103.199.7.200` (public).
- `monitoring/` — Prometheus, Grafana, Loki, Alloy, blackbox (+ redis/mongo exporter, cAdvisor, node-exporter); một dashboard Grafana chung cho metric và log.
- `https/` — Caddy (CA nội bộ) phục vụ dashboard qua HTTPS theo IP (micro trình duyệt cần origin an toàn).
- `selfhosted-voice/` — README chi tiết và mã cho: `stt-gateway` (VAD Silero + endpointing, phát lại event kiểu Deepgram), `nemotron`, `sensevoice`, `kokoro`, `embedding`, `kb-indexer`, `https`, `preflight.sh`, test.
- `local-generation/` — phụ trợ sinh nội dung cục bộ.

---

## 9. AI tự lưu trữ (`selfhosted-ai/`)

Chạy các model ngay trên server thay vì API cloud; bật bằng **một dòng** `COMPOSE_FILE=docker-compose.yml:selfhosted-ai/docker-compose.connect.yml` trong `.env`.

| Thành phần | Cloud (mặc định) | Self-host |
|---|---|---|
| STT | Deepgram Nova-3 | Nemotron 3.5 ASR Streaming qua `ai-stt-gateway` |
| Emotion (audio) | Gemini Flash-Lite | SenseVoiceSmall |
| LLM | Gemini | vLLM Qwen3.6 (`qwen3.6-35b-a3b`) |
| Embedding KB | gemini-embedding-001 | Qwen3-Embedding-0.6B (collection Qdrant riêng) |
| TTS | Google | Kokoro (voice Google), OmniVoice (voice thiết kế + clone), Qwen3-TTS / vLLM-Omni (benchmark) |
| Trích xuất trường | Gemini | NuExtract / Qwen (`extractor/`, kèm model & llama-server CUDA — phần chiếm ~4,2 GB) |

- `docker-compose.core.yml` (bật trước: network `ai-net`, volume `ai-model-cache`, `ai-stt-gateway`), `docker-compose.connect.yml` (overlay chỉ ghi đè env/network của `voiceai-orchestrator`, `view-api-service`, `document-processor`, `document-ingestion-service`, `frontend`), `_template/` để thêm model mới.
- Quy ước: `AI_STACK_PREFIX` (mặc định `ai`) cho mọi tên container/network/volume; cổng publish chỉ ở `127.0.0.1`; chạy từ gốc repo với `--project-directory .`.
- Nemotron/SenseVoice/vLLM chia sẻ với stack ngoài (`voice_stt_simulation`, `vllm_chat_qwen_3_6`) qua các network ngoài.
- Thứ tự bật: AI trước → stack chính sau. Quay lại cloud: xoá dòng `COMPOSE_FILE`.

---

## 10. Scripts (`scripts/`)

- `smoke-test.sh` — kiểm tra triển khai (thoát khác 0 nếu có lỗi).
- `demo/` — seed transcript/knowledge/loan status, sinh cuộc gọi hoàn tất, kịch bản cuộc gọi demo (Resimac, Mia home-loan refinance), `knowledge-pack`, `workflows`, `validate_demo_fleet.py`.
- `deploy/verify_courtesy_deployment.py`; `dev/python`.
- `did-map/` — seed/đối chiếu `DidMap` với dialplan.
- `ops/` — backfill audio workflow, migrate org cho call session, reassign workflow mồ côi, reclaim audio mồ côi.
- `mip4/`, `mip5/` — thu thập bằng chứng shadow, kiểm tra ownership matrix, export workflow tenant đã publish, đo thời gian compile.

---

## 11. Thí nghiệm (`experiments/`)

- `conversational_brain/` — script, fixtures và kết quả (`results_*.jsonl`, `scores_round1.json`, log) cho các thí nghiệm về "bộ não hội thoại": baseline control, knowledge, tín hiệu, strength, follow-up, confirm-gen.
- `gemini-live-spike/` — chuỗi spike `exp0…exp9` khảo sát Gemini Live (model, session memory, streaming, tools, prompt lifecycle, modalities, hybrid, interruption, limits) + `experiment-results.md`.

---

## 12. Quy trình AI-first (theo `CLAUDE.md`)

- **Một orchestrator, bốn agent**, mỗi agent một trách nhiệm; gọi trực tiếp, không có tầng điều phối trung gian.
- **Artifact-first nghiêm ngặt:** agent chỉ đọc đầu vào từ `artifacts/`; `context/*` chỉ design-agent được dùng khi sinh spec.
- Thực thi theo **task_id** trong `artifacts/planning/ai_plan.json` (không tự lập kế hoạch, không làm việc ngầm); phải ghi log đúng schema tại `logPath` được cấp.
- **Chuỗi nguồn duy nhất:** spec → ai_plan.json → thực thi.
- 3 lần sửa thất bại thì leo thang cho người; **test không bao giờ tự sửa**; commit tại các checkpoint đã được xác minh; **không dùng `Co-Authored-By`** trong commit.
- Tài liệu: ràng buộc nhiều service → `artifacts/architecture/`; một service → `docs/` của service; audit/report → `artifacts/eval/`. Không coi là hiện hành: `archive/`, `.superseded/`, `backup/`, `legacy-*`, `.claude_history/`, file `*_Gemini_Ver*`, `final*`.
- "Done" = đã được import/gọi từ entry point thật và được kiểm thử với hành vi thật.
- Công cụ: CLI `af` (`verify | review | logs | deploy | status | health`); dùng cấu hình trong `.ai-first/project-context.yaml` (toolchain Python verify bằng `PYTHONPATH=src pytest -q`; TypeScript `npx tsc --noEmit && npx vitest run`; Node `npm test`).
  Ghi chú: file này còn liệt kê compose cũ (`docker-compose.demo.yml` + `.asterisk.yml`) nhưng `af` hiện dùng `docker-compose.yml` hợp nhất (ghi chú "facts as of 2026-07-08").

---

## 13. Docker Compose

| File | Vai trò |
|---|---|
| `docker-compose.yml` (108 KB) | **Stack hợp nhất hiện hành**, 22 service chia hai profile `core` và `telephony`: `redis`, `mongo`, `api-gateway`, `tenant-config-service`, `session-service`, `conversation-service`, `audit-service`, `handoff-queue-service`, `campaign-service`, `document-ingestion-service`, `qdrant`, `qdrant-init`, `document-processor`, `stt-engine`, `tts-engine`, `llm-engine`, `response-store`, `view-api-service`, `frontend`, `https`, `voiceai-orchestrator`, `mock-crm-server` |
| `docker-compose.demo.yml`, `docker-compose.asterisk.yml` | **Đã bị thay thế** (giữ lại một bản phát hành, không dùng) |
| `docker-compose.selfhosted.yml` | Bản self-host độc lập cũ (`sh-nemotron`, `sh-sensevoice`, `sh-kokoro`, `sh-stt-gateway`, `sh-embedding`, `sh-kb-indexer`, `sh-orchestrator`, `sh-frontend`, `sh-https`) |
| `docker-compose.observability.yml` | Langfuse v3 (RFC-012) tách project riêng: web, worker, clickhouse, minio, redis, postgres; image ghim digest, chỉ `langfuse-web` mở (loopback) |
| `docker-compose.omnivoice.yml` (+ `.override.yml`), `docker-compose.vllm-omni-tts.yml` | TTS OmniVoice / vLLM-Omni |

**Mạng cố định:** `app-network` 172.24.0.0/24, `voiceai-network` 172.28.0.0/24. Dữ liệu bền nằm ở named volume (`mongo-data`, `redis-data`, `qdrant-data`, `document-store`) — **không** dùng `down -v`.

**Cổng host:** dashboard `:5173`, API gateway `:3010`, mock CRM `:8090` (mọi interface); Qdrant `:6333`, Mongo `:27018`, Redis `:6379`, mgmt orchestrator `:8089` (chỉ `127.0.0.1`); AudioSocket `:9092/:9093` (PBX ở máy khác gọi tới → nên giới hạn bằng firewall/`DOCKER-USER`, `AUDIOSOCKET_BIND`).

---

## 14. Triển khai & vận hành (tóm tắt `DEPLOYMENT.md`)

1. `git pull`; `cp .env.example .env`; điền khoá nhà cung cấp (`GEMINI_API_KEY`, `DEEPGRAM_API_KEY`, `GOOGLE_TTS_API_KEY` hoặc `GCP_CREDS_PATH`) và **tự sinh mọi secret** (`RPC_SERVICE_TOKEN`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `QDRANT_API_KEY`, `REDIS_PASSWORD`, `MONGO_ROOT_PASSWORD`, `BOOTSTRAP_ADMIN_PASSWORD`) — compose từ chối khởi động nếu thiếu, không có giá trị mặc định.
2. `docker compose up -d --build` (mặc định `COMPOSE_PROFILES=core,telephony`; `--profile core` = không có voice path; `--profile telephony` = chỉ voice path). Telephony cần `config/asterisk_instances.yaml` + `AMI_SECRET`.
3. `./scripts/smoke-test.sh`, rồi nghiệm thu bằng cách gọi agent "Andy" (insurance) hỏi về tri thức đã upload.
4. Tự động: Qdrant + `qdrant-init` tạo collection `document_chunks` (768-d, Cosine); migration Mongo; seed admin và agent demo (Andy → insurance, Sophia → healthcare; seed theo `agent_code`).
5. Có hướng dẫn riêng cho: HTTPS dashboard (Caddy), migrate subnet cho bản cài cũ, bật auth cho Redis/Mongo/Qdrant trên bản cài cũ (`ENCRYPTION_KEY` của tenant-config **không** được xoay vòng vì mã hoá credential tích hợp), auth gate dashboard, self-hosted AI.

**Về bí mật:** `staging/` chứa `.env` và 6 bản sao lưu `.env.bak*` (quyền `600`) cùng `.env.observability`; các file này có khả năng chứa khoá/mật khẩu thật và **không nên** bị commit hay chia sẻ. Tài liệu này không đọc nội dung chúng. Ngoài ra `services/Django-CRM/` có `.env.docker(.local)`, `security_audit.log`, `server.log`.

---

## 15. Điểm cần lưu ý / rủi ro

- **Tài liệu và cấu hình lệch nhau:** `CLAUDE.md` mô tả `artifacts/implementation/` là cây sinh bởi agent và nhắc `docker-compose.demo.yml`, trong khi `DEPLOYMENT.md` nói compose hợp nhất đã thay thế; `.ai-first/project-context.yaml` còn trỏ tới compose cũ.
- `README.md` gốc chưa được viết lại (còn template GitLab) và chứa địa chỉ IP nội bộ của mockup.
- Dung lượng lớn do `selfhosted-ai/extractor` (4,2 GB) và `artifacts/eval` (125 MB, >2.000 file).
- `frontend/` chứa `node_modules`, `services/*` chứa `__pycache__`, `.ruff_cache` — nên bỏ qua khi sao lưu/chuyển.
- Một số bí mật/nhật ký vận hành tồn tại trong cây thư mục (mục 14).
- Hệ thống self-host phụ thuộc container ngoài repo (`voice_stt_simulation`, `vllm_chat_qwen_3_6`) phải chạy sẵn.

---

## 16. Các luồng chính của hệ thống

> Nguồn: code (`main.py`, `audiosocket/server.py`, `ai/driver_host.py`, `conversation_integration/driver/driver.py`,
> `ami/README.md`, `management/api.py`) và spec (`artifacts/spec/orchestrator/*`). Phần nào chỉ dựa vào spec, chưa đối chiếu code,
> được ghi rõ. Chưa chạy hệ thống nên đây là kết quả đọc code/spec, không phải quan sát runtime.

### 16.1 Cuộc gọi vào (luồng cốt lõi)

```
Asterisk (dialplan ai-dispatch) ── AudioSocket TCP :9092 ──> audiosocket/server.py
   │ 1. kiểm tra IP nguồn có nằm trong allowed_cidrs không (sai → đóng kết nối)
   │ 2. đọc handshake 0x01 + UUID (= CALL_UUID), đăng ký CallSession
   │ 3. chạy song song: read_loop (khung 0x10 → audio_ingress), egress_writer, session_started_handler
   ▼
main.py::_on_session_started ──> ai/driver_host.py::run_call_via_driver
   │ a. tạo CallOrchestrator, gọi _prepare_pipeline(persist=False, greet=False):
   │      resolve profile agent (tenant-config) → từ chối nếu không rõ org / agent bị tắt /
   │      hết slot đồng thời (nói câu "bận") → mở call context → tra CRM theo số gọi
   │      (cá nhân hoá lời chào, hoặc tạo contact tạm) → dựng STT/LLM/TTS/emotion
   │ b. ghi âm audit (nếu bật), đăng ký live-listen
   │ c. tee audio: 1 luồng vào → hàng đợi STT + hàng đợi transport; chạy task STT và task emotion
   │ d. dựng ConversationDriver qua build_production_composition_root
   ▼
ConversationDriver (vòng lặp từng lượt, tuần tự, thứ tự cố định)
   A2 Admission → A3 Identity → A8 create_session (ghi conversation-service)
   mỗi lượt: A1 transport → A12 kiểm tra kết thúc → A4 tạo request → STT
             → A5 Decision/Capability → Conversation Runtime run_turn
             → A6 Response Manager (chọn wording) → A7 Playback (TTS → khung 0x10 về Asterisk)
             → A8 append_turn → A9 telemetry → A10 hoàn tất workflow → A11 release
   teardown: A8 close_session → post-call (usage, conversation intelligence, CRM, mục đích cuộc gọi)
```

- **Chỉ có một đường thực thi.** `CallOrchestrator.run()` được đánh dấu *retired* (không còn caller trong `src/`); mọi cuộc gọi đi qua Driver.
- **Driver chỉ xếp thứ tự.** Quyết định nằm ở `conversation_orchestrator/` (runtime thuần, đồng bộ); tác động phụ nằm ở adapter/gateway. Các thành phần không gọi nhau mà chỉ qua Driver (topology hình sao).
- **Quan sát vòng đời ghi.** `LifecycleObservingWriter` bọc writer để cấp lại cảm xúc từng lượt và transcript cho pipeline hậu cuộc gọi.

### 16.2 Một lượt hội thoại và giảm độ trễ

Theo spec `orchestrator/turn-pipeline.yaml` (v1.5.1):
- **Early lane** (`EARLY_LANE`): Decision, retrieval, extraction bắt đầu ngay ở mỗi STT *segment final / early final*, không đợi hết lượt. Khi lượt kết thúc, nếu văn bản vẫn là cùng một lượt (sau chuẩn hoá) thì dùng kết quả, không thì chạy lại một lần.
- **Fast lane / bridge** (`BRIDGE_MODE=off|shadow|live`): sinh trước câu mở đầu tối đa 5 từ, kiểm tra (không số, không câu hỏi, không hứa hẹn, không tự nhận là người), TTS sẵn; nếu câu trả lời chính chưa kịp thì phát bridge trước.
- **Late take:** chờ bridge tối đa 700 ms; LLM trả lời tiếp nối đúng câu bridge đã phát.
- **Filler và speculation** (`ai/speculation*.py`, `_play_filler_after_delay`) che thêm thời gian chờ.
- **Barge-in:** người gọi nói chen thì huỷ việc đang chạy.
- **Decision grammar v2** (7 loại caller kind) nằm sau cờ `DECISION_GRAMMAR`, mặc định `1.3.0`.

### 16.3 Chạy song song trong cuộc gọi

- **Cảm xúc:** audio + text → hợp nhất → làm mượt → chèn `[tone: X]` nếu đủ ngưỡng; không làm chậm câu trả lời.
- **Risk → hàng đợi handoff:** `RiskEscalationTracker` (dựa trên sentiment từng lượt) đưa cuộc gọi vào handoff queue khi vào band Risk/Concern, gỡ sau thời gian ân hạn khi về Calm.
- **Hậu cuộc gọi:** tóm tắt mục đích người gọi, phân tích cuộc gọi, usage/token/latency, conversation intelligence.
- **Dọn cuộc gọi treo:** `session_reconciler` chạy mỗi phút, đóng cuộc gọi treo với lý do `ABANDONED_RECONCILED`.

### 16.4 Knowledge (RAG) và workflow

- `knowledge/layer.py` xếp provider theo rank: TenantStructured 0, Product 10, TenantDocument 20; lấy kết quả khớp đầu tiên.
- Tài liệu upload → `document-ingestion-service` → `document-processor` → embed vào Qdrant (`document_chunks`).
- `workflows/` (compiler, field extraction, confirmation, business records) chạy intake nhiều bước; khi xong, sự kiện `WorkflowCompleted` ghi bản ghi nghiệp vụ và đẩy sang CRM.

### 16.5 Chuyển người thật (handoff, take-over, listen)

1. `POST /sessions/{uuid}/handoff-to-human` → `ThreeWayAmiClient.merge_call()` chuyển kênh người gọi vào phòng ConfBridge và originate agent người thật vào cùng phòng.
2. Mở hai ARI snoop leg (người gọi, agent) vào cổng AudioSocket thứ hai; `ListenOnlyOrchestrator` chạy STT và ghi transcript cùng `call_uuid` với nhãn `caller` / `human_agent`.
3. **Take Over từ trình duyệt:** WebSocket `/take-over-call`, dùng chung cổng thứ hai; `secondary_leg_dispatcher` phân biệt vai trò theo registry.
4. **Live-listen** của giám sát viên: WebSocket `/live-listen`.

### 16.6 Cuộc gọi ra (campaign) — theo spec, chưa theo dấu hết trong code

`Campaign_Service` tạo `CallSession.id` → `Outbound_Dialer_Adapter` gửi AMI Originate kèm `CALL_UUID` → khi bắt máy hội tụ về `ai-dispatch` như cuộc gọi vào. Campaign có tuỳ chọn *2-way communication*: bật thì thành hội thoại thật, tắt thì chỉ thông báo rồi cúp (`announce_only_orchestrator`).

### 16.7 Luồng dashboard (điều khiển và đọc)

```
Browser ─ https (Caddy) ─> frontend :5173
   ├─ /api/*  ──> api-gateway :3010 (JWT+RBAC) ──> tenant-config / campaign / audit / handoff-queue / document-ingestion
   │                                           └──> /view  ──> view-api-service (tổng hợp chỉ-đọc, envelope {data,meta,error})
   └─ /runtime-api/* ─ authGate.ts (hỏi gateway xác thực; chỉ 4 endpoint được phép) ──> orchestrator :8081
       (runtime-config, dev-tts, dev-call, take-over-call)
```

Cấu hình agent đi qua tenant-config rồi được orchestrator nạp khi bắt đầu cuộc gọi (overlay system prompt). Spec UC-S10 nói *hot-reload*; cơ chế này chưa được kiểm tra trong code.

### 16.8 Luồng triển khai

`docker compose up` (profile `core` + `telephony`) → seed (admin, Andy, Sophia) + `qdrant-init` → `smoke-test.sh`. Self-host bật bằng overlay `selfhosted-ai/docker-compose.connect.yml`.

### 16.9 Chưa xác minh

- Spec mô tả `Session_Manager` + `Voice_Worker` qua gRPC `:9093`, nhưng code chạy thực tế là **một tiến trình orchestrator**; `:9093` trong code là cổng AudioSocket thứ hai (listen/take-over/announce).

---

## 17. Luồng nào dùng model AI (STT / LLM / TTS)

> Nguồn: các file cấu hình/khởi tạo client trong `services/voice-ai-orchestrator/src/voice_ai_orchestrator/` và `services/view-api-service/src/`.
> Tên model là **giá trị mặc định trong code**; môi trường staging có thể ghi đè bằng `.env` (không được đọc ở đây), nên model thực tế đang chạy cần kiểm tra lại.

### 17.1 Bảng các provider và công tắc chọn

| Loại | Mặc định (cloud) | Self-host / thay thế | Công tắc |
|---|---|---|---|
| **STT** | Deepgram Nova-3 (`ai/stt.py`, `DEEPGRAM_MODEL`) | Nemotron 3.5 ASR Streaming qua stt-gateway (`ai/stt_selfhosted.py`) | `STT_PROVIDER=selfhosted` + `SELFHOSTED_STT_URL` |
| **LLM trả lời** | Gemini, mặc định `gemini-3.5-flash` (`ai/llm.py`, `GEMINI_MODEL`) | LLM tương thích OpenAI/vLLM, ví dụ Qwen3.6 (`ai/llm_local_chat.py`) | `LLM_PROVIDER`, `LOCAL_LLM_BASE_URL`, `LOCAL_LLM_MODEL` |
| **LLM Decision** (chọn "kind" của lượt) | Gemini | `shadow` / `hybrid` / `local` (model cục bộ) / `llm` (vLLM) | `DECISION_PRODUCER_MODE` (mặc định `gemini`); `DECISION_LLM_BASE_URL`, `DECISION_LLM_MODEL` |
| **TTS** | Google Cloud TTS (WaveNet/Chirp…) (`ai/tts.py`) | Deepgram Aura, OmniVoice, Chatterbox, Kokoro-82M, Qwen3-TTS | `TTS_PROVIDER`, nhưng **giọng của agent quyết định trước**: `clone:<id>`→OmniVoice, `qwen3:<id>`→Qwen3-TTS, `omnivoice:<preset>`→OmniVoice, id Google→Google (hoặc Kokoro nếu `TTS_GOOGLE_VOICES_LOCAL=1`) (`ai/tts_factory.py`) |
| **Cảm xúc** | Gemini audio + text, mặc định `gemini-3.1-flash-lite` (`ai/emotion.py`) | SenseVoiceSmall (audio) + LLM text cục bộ | `EMOTION_PROVIDER=sensevoice`, `SENSEVOICE_URL`, `EMOTION_TEXT_BASE_URL/MODEL` |
| **Embedding (RAG)** | `gemini-embedding-001` | Qwen3-Embedding-0.6B | `RAG_EMBEDDER=local`, `LOCAL_EMBEDDING_URL` |
| **Trích xuất trường** | Gemini (`gemini-3.5-flash`) | `LOCAL_JSON_LLM_*`; NuExtract / Qwen (`information_extraction/`) | `LOCAL_JSON_LLM_BASE_URL`, `EXTRACTION_PROVIDER` |

### 17.2 Ma trận luồng × model

Ký hiệu: ● = dùng chắc chắn (thấy trong code), ◐ = dùng có điều kiện (cờ/cấu hình), ○ = không dùng.

| Luồng | STT | LLM | TTS | Ghi chú |
|---|:-:|:-:|:-:|---|
| **Cuộc gọi vào – mỗi lượt hội thoại** (16.1) | ● | ● | ● | STT: audio người gọi → text. LLM: Decision + câu trả lời (+ tool/knowledge). TTS: phát câu trả lời (`synthesize_stream`, `ai/driver_host.py`) |
| **Lời chào đầu cuộc gọi** | ○ | ○/◐ | ● | Qua Response Manager (`_emit_greeting_via_response_manager`) hoặc lời chào cũ; được TTS (`orchestrator.py:6864`). LLM chỉ dùng nếu wording được sinh động |
| **Câu từ chối "hết slot / agent tắt / không rõ org"** | ○ | ○ | ● | `_speak_admission_prompt` tạo TTS bằng giọng của agent (`orchestrator.py:3999`) |
| **Bridge / fast lane** (16.2) | ◐ | ● | ● | Cần `BRIDGE_MODE=live` (mặc định `off`). Gọi LLM (endpoint reply-LLM) sinh ≤5 từ rồi TTS trước (`orchestrator.py:1268`) |
| **Early lane** (Decision/retrieval/extraction sớm) | ◐ | ● | ○ | Cần `EARLY_LANE=on`; chạy trên STT early final |
| **Filler / speculation** | ○ | ◐ | ◐ | Filler dựng từ clip có sẵn (`assets/fillers`, `_filler_clip`); speculation thêm lượt LLM (`ai/speculation*.py`) |
| **Phân tích cảm xúc** (16.3) | ○ (dùng audio của STT) | ● | ○ | Gemini audio/text hoặc SenseVoice; chạy song song |
| **RAG / knowledge** (16.4) | ○ | ● | ○ | Embedding câu hỏi (Gemini hoặc Qwen3-Embedding) + tìm Qdrant; sau đó LLM trả lời có căn cứ |
| **Workflow / intake – trích xuất trường** | ○ | ● | ○ | Gemini hoặc LOCAL_JSON_LLM / NuExtract / Qwen |
| **Tóm tắt mục đích người gọi** (`caller_purpose`) | ○ | ● | ○ | Mặc định `gemini-3.1-flash-lite`, làm mới mỗi 4 lượt người gọi |
| **Phân tích hậu cuộc gọi** (`post_call_analysis`) | ○ | ● | ○ | Mặc định `gemini-3.5-flash` |
| **Handoff sang người + listen-only** (16.5) | ● | ○ | ○ | STT chạy cho từng snoop leg để ghi transcript (`ai/listen_only_orchestrator.py`); không dùng LLM/TTS ở nhánh này |
| **Take-over từ trình duyệt** | ◐ | ○ | ○ | Cầu nối audio người ↔ người gọi (`takeover_bridge_orchestrator`); STT nếu bật ghi transcript — chưa kiểm tra chi tiết |
| **Cuộc gọi ra "announce-only"** (16.6) | ● | ○ | ● | `announce_only_orchestrator.py` khởi tạo `STTClient` và phát lời thông báo bằng TTS; không dùng LLM trả lời |
| **Cuộc gọi ra 2-way** | ● | ● | ● | Như cuộc gọi vào |
| **Dev call / Developer Voice trên dashboard** (`/dev-call`, `browser_voice`) | ● | ● | ● | Cùng pipeline như cuộc gọi thật, audio từ trình duyệt |
| **Nghe thử giọng (`/dev-tts`)** | ○ | ○ | ● | `browser_voice/endpoint.py` tạo TTS cho từng request |
| **Tải clip giọng lên để nhận dạng/clone** (`view-api-service/voiceTranscribe.mjs`) | ● | ○ | ◐ | STT bằng Deepgram hoặc `SELFHOSTED_TRANSCRIBE_URL` (Nemotron). `voiceCalibrate.mjs` liên quan hiệu chỉnh giọng — chưa xác định rõ model dùng |
| **Gemini Live (`conversation_runtime/gemini_live`)** | ◐ | ◐ | ◐ | Model audio-native `gemini-2.5-flash-native-audio-preview-12-2025`; có vẻ là runtime thay thế/thí nghiệm — chưa xác nhận nó nằm trên đường chạy sản xuất (xem `experiments/gemini-live-spike/`) |
| **Dashboard, API gateway, view-api, CRM, campaign, audit, analytics** | ○ | ○ | ○ | Không gọi model AI trong lúc phục vụ (view-api chỉ tổng hợp dữ liệu) |
| **Nạp tài liệu knowledge (upload → chunk → embed)** | ○ | ◐ | ○ | `document-processor` gọi embedder (Gemini hoặc Qwen3-Embedding qua `selfhosted-ai/embedding`) — là model embedding, không phải LLM trả lời |

### 17.3 Tóm tắt

- **Cả ba loại (STT + LLM + TTS) cùng xuất hiện** ở: mỗi lượt của cuộc gọi vào/ra 2-way, dev call từ trình duyệt, và bridge (LLM+TTS).
- **Chỉ STT:** ghi transcript ở handoff listen-only; tải clip giọng lên để nhận dạng.
- **Chỉ LLM (không STT/TTS trực tiếp):** cảm xúc, RAG, trích xuất trường, tóm tắt mục đích, phân tích hậu cuộc gọi.
- **Chỉ TTS:** câu từ chối khi admission, nghe thử giọng.
- **STT + TTS, không LLM:** cuộc gọi ra announce-only.
- **Không dùng model AI:** dashboard, gateway, view-api, CRM, campaign, audit, analytics.

### 17.4 Điểm chưa xác minh

- Model thực tế đang chạy trên staging phụ thuộc `.env` (không đọc). Bảng trên là mặc định trong code.
- Các mục đánh dấu "chưa kiểm tra / chưa xác nhận" (take-over, `voiceCalibrate`, Gemini Live) cần đọc sâu hơn trước khi kết luận.
- Các service trong `artifacts/implementation/backend/` (`stt-engine`, `tts-engine`, `llm-engine`, `emotion-classifier`, `rag-search-engine`, `voice-ai-orchestrator-core`) cũng có client gọi Deepgram/Gemini và có trong `docker-compose.yml`; tôi chưa xác định luồng cuộc gọi thật có đi qua chúng hay chỉ `voice-ai-orchestrator` gọi trực tiếp các nhà cung cấp.

---

## 18. Luồng self-hosted: `.env` được đọc và xử lý như thế nào

> **Phương pháp & phạm vi.** Đọc `selfhosted-ai/docker-compose.{connect,core}.yml`, `docker-compose.yml` (khối `voiceai-orchestrator`),
> `selfhosted-ai/.env.example`, `DEPLOYMENT.md`, `infra/selfhosted-voice/README.md`, `start.sh`, và code đọc biến môi trường
> (`common/config.py`, `runtime/runtime_config.py`, `ai/tts_factory.py`, `ai/stt_selfhosted.py`, `conversation_orchestrator/decision/producer_mode.py`…).
> Với `.env` thật, tôi **chỉ lấy tên biến và vài công tắc không nhạy cảm**; mọi khoá/mật khẩu/token được che.
> Giá trị "hiệu lực" lấy từ `docker compose config` (chỉ đọc, không khởi động gì). Quyền truy cập Docker API bị từ chối,
> nên **không kiểm tra được container nào đang chạy thật**. Bảng giá trị bên dưới là cấu hình *sẽ được áp dụng*, không phải trạng thái runtime.

### 18.1 Tóm tắt luồng

```
(1) Chuẩn bị          (2) Compose đọc .env              (3) Hợp nhất file           (4) Container nhận env       (5) Code đọc env
 .env ở gốc repo  ──>  COMPOSE_FILE=a.yml:b.yml   ──>   docker-compose.yml     ──>   chỉ các khoá liệt kê   ──>  os.getenv(...)
 (copy từ              COMPOSE_PROFILES                 + connect overlay            trong `environment:`        / ${VAR} trong YAML
 selfhosted-ai/        thay thế ${VAR:-default}         (overlay thắng theo khoá)    (KHÔNG có env_file)         / JSON runtime_config
 .env.example)         trong cả hai file
```

Bật self-host chỉ cần **một dòng** trong `.env`: `COMPOSE_FILE=docker-compose.yml:selfhosted-ai/docker-compose.connect.yml`.
Từ đó mọi lệnh `docker compose` (up/build/config…) tự gộp thêm overlay. Gỡ dòng này để quay về cloud.

### 18.2 Các file env và vai trò

| File | Ai đọc | Ghi chú |
|---|---|---|
| `.env` (gốc repo, quyền 600) | **Compose** (nội suy `${VAR}`; đọc `COMPOSE_FILE`, `COMPOSE_PROFILES`). Không phải container | Nguồn duy nhất cho stack chính **và** mọi file trong `selfhosted-ai/` (chạy với `--project-directory .`). 86 khoá hiện tại |
| `.env.example` | Con người | Mẫu để `cp` sang `.env` |
| `selfhosted-ai/.env.example` | Con người | **Không phải file nạp tự động**: README ghi "không có env file riêng"; phải chép các biến cần thiết vào `.env` gốc |
| `.env.selfhosted.example` | Con người | Dành cho `docker-compose.selfhosted.yml` (stack độc lập kiểu cũ), không phải đường chính thức |
| `.env.bak.*`, `.env.bak-*` (6 bản) | **Không ai đọc** | Chỉ là bản sao lưu; Compose chỉ đọc đúng tên `.env` |
| `.env.observability` (+ `.example`) | Compose của Langfuse, qua `--env-file deploy/observability/.env.observability` | Project riêng `voiceai-observability` |
| `services/voice-ai-orchestrator/.env.test.local` | `start.sh` (chạy trên host, không Docker) | Mặc định `ENV_FILE=.env.test.local`, `source` với `set -a`; có thể đổi bằng `ENV_FILE=…` |
| `services/view-api-service/.env.example`, `frontend/.env.example` | Con người / chạy cục bộ | Các service này trong Docker nhận env qua compose |
| `config/asterisk_instances.yaml` | Orchestrator (YAML, không phải env) | Có placeholder `${AMI_SECRET}`, `${ARI_PASSWORD}`, `${ORCH_TUNNEL_IP:-10.99.0.1}` được **mở rộng khi nạp** (xem 18.3-D) |

### 18.3 Cơ chế xử lý theo từng giai đoạn

**A. Compose đọc `.env` (nội suy, chưa phải biến của container)**
- Compose nạp `.env` ở *project directory*. Các file `selfhosted-ai/*.yml` phải chạy với `--project-directory .`, nếu không `.env` gốc không được đọc và đường dẫn build sai.
- `COMPOSE_FILE` dùng dấu `:` để nối file; `COMPOSE_PROFILES=core,telephony` quyết định service nào được bật. Cờ `--profile` trên dòng lệnh **thay thế** (không cộng thêm) giá trị này. `af` tự export cả hai profile nên không phụ thuộc `.env`.
- Cú pháp mặc định trong YAML: `${X:-d}` dùng `d` khi X *chưa đặt hoặc rỗng*; `${X-d}` chỉ dùng `d` khi X *chưa đặt*. Overlay dùng dạng `-` cho `EXTRACTION_PROVIDER: ${SH_EXTRACTION_PROVIDER-nuextract}`: đặt `SH_EXTRACTION_PROVIDER=` (rỗng, có trong `.env`) nghĩa là **quay về Gemini**, còn không đặt nghĩa là NuExtract.
- Cú pháp bắt buộc `${X:?msg}` làm Compose dừng nếu thiếu (dùng cho `REDIS_PASSWORD`, `MONGO_ROOT_PASSWORD`…). Biến môi trường của shell thắng `.env`.
- Lồng nhau: `${SH_EMOTION_TEXT_URL:-${SH_QWEN_URL:-http://vllm_chat_qwen_3_6:8000}}` — một khoá `SH_QWEN_*` có thể chi phối nhiều tính năng cùng lúc.

**B. Hợp nhất `docker-compose.yml` + overlay `connect`**
- Overlay **không định nghĩa service mới**; chỉ ghi đè `environment` và `networks` của `voiceai-orchestrator`, `view-api-service`, `document-processor`, `document-ingestion-service`, `frontend` (build arg).
- Với cùng một khoá, **giá trị trong overlay thắng** giá trị của file gốc (kết quả `docker compose config` xác nhận). Nhiều giá trị trong overlay là *hằng* (không đọc `.env`), ví dụ `STT_PROVIDER: selfhosted`, `LLM_PROVIDER: local`, `EMOTION_PROVIDER: sensevoice`, `RAG_EMBEDDER: local`, `TTS_GOOGLE_VOICES_LOCAL: "1"`.
- Overlay nối orchestrator vào 4 network **bên ngoài**: `ai-net`, `asr-net` (mặc định `vsim_default`), `llm-net` (`vllm_llm_net`), `decision-llm-net` (`chat_qwen_36_default`). Nếu network nào chưa tồn tại thì `up` thất bại; vì vậy thứ tự là AI stack trước, stack chính sau.

**C. Container chỉ thấy các khoá được liệt kê (không có `env_file`)**
- Khối `voiceai-orchestrator` trong `docker-compose.yml` dùng `environment:` thuần, **không `env_file:`**. Vì vậy một khoá trong `.env` mà không được nhắc tới trong `environment:` (hoặc overlay) thì **không bao giờ tới container**. Hiện orchestrator nhận 177 biến.
- Thêm một biến mới cho code Python cần 2 bước: khai báo trong compose (thường dạng `${VAR:-default}`), rồi đặt giá trị trong `.env`.

**D. Code đọc biến môi trường**
- Đọc trực tiếp bằng `os.getenv` / `os.environ`; **không dùng `python-dotenv`** (grep không thấy). `start.sh` chạy ngoài Docker thì `source` file env để đạt cùng hiệu quả.
- Hai thời điểm đọc khác nhau, có hệ quả khi đổi cấu hình:
  - **Lúc import module** (hằng cấp module), ví dụ `PLATFORM_DEFAULT_VOICE = os.getenv("TTS_DEFAULT_VOICE", …)` trong `ai/tts.py`.
  - **Lúc gọi / lúc khởi tạo client của từng cuộc gọi**, ví dụ `DECISION_PRODUCER_MODE` (`producer_mode.py` ghi rõ "đọc mỗi lần gọi, không cache"), `ai/tts_factory.py`, `STT_PROVIDER` (`stt_selfhosted.py`), `GEMINI_MODEL` (trong `__init__` của `ai/llm.py`).
  - Dù cách nào, biến chỉ thay đổi khi **container được tạo lại**: `docker compose up -d --no-deps voiceai-orchestrator` (hoặc `af deploy orchestrator`). Sửa `.env` mà không tạo lại container thì không có tác dụng.
- `common/config.py` mở rộng `${VAR}` / `${VAR:-default}` trong mọi chuỗi của `asterisk_instances.yaml` bằng `os.environ`; `${VAR}` không có mặc định mà thiếu → `KeyError` lúc khởi động (cố ý "fail loud").
- Khác với env: **`runtime/runtime_config.py`** là kho JSON (`RUNTIME_CONFIG_PATH=/app/state/runtime_config.json`, volume `orchestrator-state`) cho 3 cờ chỉnh từ dashboard (`shadow_mode`, `verbose_runtime_logging`, `developer_voice`) qua `GET/PUT /runtime-config`. Không liên quan `.env`, giữ qua lần tạo lại container, và áp dụng cho cuộc gọi *mới*.
- Frontend: `VITE_SELFHOSTED_TTS`, `VITE_QWEN3_TTS` là **build arg** nhúng vào bundle JS lúc build, nên cần `docker compose up -d --build` mới có hiệu lực.

**E. Các service khác chịu ảnh hưởng của overlay**
- `view-api-service`: `OMNIVOICE_BASE_URL`, `SELFHOSTED_TRANSCRIBE_URL` → transcript đăng ký giọng dùng Nemotron thay Deepgram (`voiceTranscribe.mjs`).
- `document-processor`: `LOCAL_EMBEDDING_URL`, `LOCAL_EMBEDDING_MODEL_VERSION` → embed tài liệu upload bằng Qwen3-Embedding.
- `document-ingestion-service`: `QDRANT_COLLECTION` → cùng collection mà orchestrator tìm kiếm.

**F. Các stack trong `selfhosted-ai/` chạy riêng**
- Mỗi file là một compose project (`name: ${AI_STACK_PREFIX:-ai}-core`…), đọc cùng `.env` gốc. `AI_STACK_PREFIX` đặt tiền tố cho container/image/network/volume để nhiều người dùng chung máy không đụng nhau; mọi cổng host là biến `SH_*_PORT` và chỉ bind `127.0.0.1`.
- `core` là file đầu tiên: nó **tạo** `ai-net` và volume `ai-model-cache`, các file model sau tham gia dạng `external`. Nó chỉ mount volume (read-only) để Compose buộc tạo volume đó.

### 18.4 Thứ tự ưu tiên (cao → thấp)

1. Biến môi trường của shell lúc chạy `docker compose`.
2. Giá trị hằng trong overlay `connect` (ví dụ `STT_PROVIDER: selfhosted`).
3. `environment:` trong `docker-compose.yml` sau khi nội suy từ `.env` (`${VAR:-default}`).
4. Giá trị mặc định nằm trong `${VAR:-default}` hoặc trong code (`os.getenv("X", default)`).
5. Cờ lưu trong `runtime_config.json` (chỉ 3 cờ dashboard, tách biệt với env).

### 18.5 Ánh xạ: công tắc `SH_*` → biến container → nơi đọc

| Chức năng | Biến trong `.env` (SH_*) | Biến container (overlay) | Nơi đọc | Cloud (khi tắt overlay) |
|---|---|---|---|---|
| STT | `SH_STT_URL`, `SH_ENDPOINTING_MS`, `SH_UTTERANCE_END_MS`, `SH_STT_EARLY_FINAL_MS` | `STT_PROVIDER=selfhosted`, `SELFHOSTED_STT_URL`, `DEEPGRAM_ENDPOINTING_MS`, `DEEPGRAM_UTTERANCE_END_MS`, `STT_EARLY_FINAL_MS` | `ai/stt_selfhosted.py`, `ai/stt.py` | Deepgram Nova-3 |
| Cảm xúc audio | `SH_SENSEVOICE_URL` | `EMOTION_PROVIDER=sensevoice`, `SENSEVOICE_URL` | `ai/emotion_sensevoice.py` | Gemini Flash-Lite |
| Cảm xúc text | `SH_EMOTION_TEXT_URL/MODEL` → `SH_QWEN_URL/MODEL` | `EMOTION_TEXT_BASE_URL`, `EMOTION_TEXT_MODEL` | `ai/emotion_sensevoice.py` | Gemini |
| LLM trả lời | `SH_LLM_BASE_URL`, `SH_LLM_MODEL`, `SH_LLM_TOOL_CALLING`, `SH_LLM_DROP_TOOLS`, `SH_LLM_ALLOW_KNOWLEDGE` | `LLM_PROVIDER=local`, `LOCAL_LLM_BASE_URL`, `LOCAL_LLM_MODEL`, `LOCAL_LLM_*` | `ai/llm_local_chat.py`, `ai/llm.py` | Gemini |
| Decision | `SH_DECISION_PRODUCER_MODE`, `SH_DECISION_LLM_URL/MODEL` | `DECISION_PRODUCER_MODE` (mặc định `llm`), `DECISION_LLM_BASE_URL/MODEL` | `decision/producer_mode.py`, `producer_client_openai.py` | `gemini` |
| Tác vụ phụ (mục đích người gọi, hậu cuộc gọi, trích xuất trường…) | `SH_QWEN_URL`, `SH_QWEN_MODEL` | `LOCAL_JSON_LLM_BASE_URL`, `LOCAL_JSON_LLM_MODEL` | `ai/local_json_llm.py`, `ai/gemini_json.py` | Gemini |
| Trích xuất trường | `SH_EXTRACTION_PROVIDER`, `SH_NUEXTRACT_*`, `SH_EXTRACTION_*` | `EXTRACTION_PROVIDER` (mặc định `nuextract`), `LOCAL_EXTRACTION_*`, `EXTRACTION_QWEN_*`, `EXTRACTION_BREAKER_*` | `information_extraction/factory.py` | Gemini (khi để rỗng) |
| Embedding / RAG | `SH_EMBEDDING_URL`, `SH_EMBEDDING_MODEL`, `SH_QDRANT_COLLECTION`, `SH_EMBEDDING_MODEL_VERSION`, `SH_RAG_QUERY_HISTORY_TURNS` | `RAG_EMBEDDER=local`, `LOCAL_EMBEDDING_URL/MODEL`, `QDRANT_COLLECTION`, `RAG_EMBEDDING_MODEL_VERSION` | `knowledge/local_embedder.py`, `ai/retrieval.py` | `gemini-embedding-001`, collection `document_chunks` |
| TTS | `SH_TTS_PROVIDER`, `SH_KOKORO_URL/VOICE`, `SH_OMNIVOICE_*`, `SH_QWEN3_TTS_URL` | `TTS_PROVIDER` (mặc định `kokoro`), `TTS_GOOGLE_VOICES_LOCAL=1`, `KOKORO_*`, `OMNIVOICE_*`, `QWEN3_TTS_BASE_URL` | `ai/tts_factory.py` | Google TTS |
| Tối ưu độ trễ | `SH_ENDPOINTING_MS`, `SH_FIRST_SEGMENT_RULE`, `SH_TTS_TRIM_LEADING_SILENCE`, `SH_WORKFLOW_FAST_PATH`… cùng `BRIDGE_MODE`, `EARLY_LANE` (đọc trực tiếp, không qua SH_*) | tương ứng | `ai/orchestrator.py`, `driver.py` | — |

### 18.6 Giá trị hiệu lực quan sát được (từ `docker compose config`)

Cấu hình staging hiện tại bật overlay (`COMPOSE_FILE` có overlay, `COMPOSE_PROFILES=core,telephony`, `AI_STACK_PREFIX=ai`). Orchestrator sẽ nhận:

| Hạng mục | Giá trị hiệu lực |
|---|---|
| STT | `STT_PROVIDER=selfhosted`; URL `ws://demo-stt-gateway:8000/v1/listen`; endpointing 700 ms, utterance-end 1500 ms, early-final 300 ms |
| Reply LLM | `LLM_PROVIDER=local` → `http://vllm_chat_qwen_3_6:8000`, model `qwen3.6-35b-a3b`, tool calling bật, KB inject bật |
| Decision | `DECISION_PRODUCER_MODE=llm` → cùng vLLM Qwen |
| Cảm xúc | `EMOTION_PROVIDER=sensevoice`; text qua Qwen (`EMOTION_TEXT_THINKING_OFF=1`) |
| Tác vụ phụ | `LOCAL_JSON_LLM_*` → Qwen; `EXTRACTION_PROVIDER=nuextract` (`http://ai-nuextract:8000`, breaker 3 lỗi / mở 30 s) |
| Embedding | `RAG_EMBEDDER=local`, `http://ai-embedding:8000`, `Qwen/Qwen3-Embedding-0.6B`, collection `document_chunks_qwen3_0_6b` |
| TTS | `TTS_PROVIDER=kokoro`, `TTS_GOOGLE_VOICES_LOCAL=1`, Kokoro `ai-kokoro`, OmniVoice `ai-omnivoice:8880`, Qwen3-TTS `ai-qwen3-tts:8091` |
| Độ trễ | `BRIDGE_MODE=live`, `EARLY_LANE=on`, `EXTRACTION_AFTER_DECISION=on`, `BRIDGE_AFTER_DECISION=workflow`, `FILLER_ENABLED=false`, `FIRST_SEGMENT_RULE=sentence40` |
| Networks | `ai-net`, `asr-net`, `llm-net`, `decision-llm-net`, `app-network`, `voice-net` |

### 18.7 Phát hiện và rủi ro

1. **Khoá "chết" trong `.env` gây hiểu nhầm.** `.env` đang có `TTS_PROVIDER=omnivoice`, `DECISION_PRODUCER_MODE=shadow`, `LLM_PROVIDER=local`, `LOCAL_LLM_BASE_URL=http://vllm:8000`, `LOCAL_LLM_MODEL=ministral`, nhưng giá trị hiệu lực lần lượt là `kokoro`, `llm`, `local`, `http://vllm_chat_qwen_3_6:8000`, `qwen3.6-35b-a3b`.
   - Nguyên nhân: overlay ghi đè `TTS_PROVIDER`/`DECISION_PRODUCER_MODE` theo `SH_*`, còn `LLM_PROVIDER` và `LOCAL_LLM_*` **không được nội suy ở compose nào** (kiểm tra bằng grep: 20 khoá trong `.env` không được `docker-compose.yml` hay overlay tham chiếu; trong đó `COMPOSE_FILE`/`COMPOSE_PROFILES` do chính Compose dùng, còn lại gồm `LLM_PROVIDER`, `LOCAL_LLM_*`, `TTS_STREAMING`, `AGENT_CODE`, `AGENTS_VIEW_SOURCE`, `ORCH_TUNNEL_IP`, `SH_LIVE_CAPTIONS`, nhóm `STATIC_DEMO_*`; phạm vi kiểm tra chỉ gồm hai file này).
   - Hệ quả: người đọc `.env` sẽ tưởng staging dùng OmniVoice/`shadow`/`ministral`, trong khi thực tế không. Nên dọn hoặc ghi chú. Muốn đổi TTS phải dùng `SH_TTS_PROVIDER`, không phải `TTS_PROVIDER`.
2. **Hai tên gateway STT.** Mặc định của overlay là `ai-stt-gateway`, còn `.env` đặt `SH_STT_URL=ws://demo-stt-gateway:8000/v1/listen`. Các báo cáo trong `artifacts/eval/` cho thấy đây là chủ ý (container `demo-stt-gateway` của project `ai-core`, tách khỏi `ai-stt-gateway`), nhưng `DEPLOYMENT.md` và README không nhắc. Tôi không xác minh được container tồn tại lúc này.
3. **Giá trị mặc định cứng trong file đã commit.** Overlay đặt mặc định cho `LOCAL_EXTRACTION_API_KEY` (token của NuExtract) ngay trong YAML, kèm ghi chú "same default as the extractor compose files". `DEPLOYMENT.md` lại nêu nguyên tắc "không có giá trị mặc định cho secret". Cần coi là mật khẩu công khai hoặc đưa vào `.env` (giá trị không được lặp lại ở đây).
4. **Chú thích mâu thuẫn về "mục đích người gọi".** Đầu overlay ghi "Still on Gemini while GEMINI_API_KEY is set: caller purpose", nhưng một chú thích khác trong cùng file nói caller purpose chạy trên `LOCAL_JSON_LLM_*`. Code `ai/caller_purpose.py` dùng `GeminiCaller`, mà `GeminiCaller` chuyển sang LLM cục bộ khi `LOCAL_JSON_LLM_BASE_URL` được đặt (`gemini_json.py`); cấu hình hiện tại đặt biến này nên nhiều khả năng đã chạy cục bộ, nhưng chưa kiểm thử runtime.
5. **Còn phụ thuộc cloud.** `GEMINI_API_KEY`, `DEEPGRAM_API_KEY`, `GOOGLE_TTS_API_KEY` vẫn được đẩy vào container ("Google credentials stay"). `EMOTION_MODEL=gemini-3.1-flash-lite` vẫn nằm trong env nhưng không dùng khi `EMOTION_PROVIDER=sensevoice`. Nghĩa là chưa phải chế độ air-gapped; nếu endpoint cục bộ lỗi, một số đường vẫn có thể rơi về Gemini tùy module (chưa kiểm tra từng chỗ).
6. **Phụ thuộc network/container ngoài repo.** `vsim_default`, `chat_llm_net`, `chat_qwen_36_default`, `vllm_chat_qwen_3_6`, `nemotron`, `sensevoice` phải tồn tại trước. Tên network trong `.env` (`SH_LLM_NETWORK=chat_llm_net`) khác mặc định của overlay (`vllm_llm_net`), nên mỗi máy phải đặt đúng.
7. **Số chiều embedding.** Qwen3-Embedding gốc 1024 chiều, được cắt Matryoshka xuống 768 (`EMBED_DIM: "768"`) để khớp hợp đồng vector của nền tảng; collection riêng `document_chunks_qwen3_0_6b` do `kb-indexer` (sao chép từ `document_chunks`) tạo. Bản thân `RAG_EMBEDDING_MODEL_VERSION` là bộ lọc tìm kiếm nên vector Gemini không thể khớp nhầm vector Qwen. Đổi model embedding cần chạy lại `kb-indexer`.
8. **`.env.bak.*` chứa bí mật** và nằm cạnh `.env`; không được nạp tự động nhưng không nên giữ lại lâu hoặc commit.


---

## 19. Phân tích 9 tác vụ LLM của luồng hội thoại

Nội dung này đã được tách sang tài liệu riêng: **[`LLM_TASKS_PROMPTS_VI.md`](LLM_TASKS_PROMPTS_VI.md)**. Tài liệu đó gồm bảng đối chiếu 9 tác vụ với code, tham số gọi và **prompt nguyên văn** của từng tác vụ.
