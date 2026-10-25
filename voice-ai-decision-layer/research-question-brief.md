# Research Question Brief — Voice AI Decision Layer

**Trạng thái:** Phạm vi và câu hỏi được người dùng xác nhận ngày 2026-10-26.  
**Giai đoạn:** Scoping; chưa bắt đầu literature review, chưa chốt model shortlist hay benchmark protocol.  
**Tài liệu hệ thống:** Nội dung trong `reference-docs/` là bản tổng hợp từ repo staging, không xác nhận runtime live. Các plan cũ chỉ là tư liệu lịch sử.

## Câu hỏi nghiên cứu đã xác nhận

> **Một LLM nhỏ, dense và có thể fine-tune có đạt chất lượng Decision cần thiết trên cả grammar 2.0.0 lẫn grammar 1.3.0, để cùng một model đảm nhận tác vụ Decision thay cho Qwen3.6?**

“Cùng một model” nghĩa là cùng bộ trọng số; mỗi grammar dùng prompt và định dạng đầu ra tương ứng.

## Mục tiêu

Tìm một LLM dense cỡ nhỏ có thể đảm nhận riêng tác vụ Decision, giảm phần tác vụ này khỏi Qwen3.6, trong khi Qwen3.6 tiếp tục là model chính cho các tác vụ còn lại. So sánh chất lượng ban đầu và sau fine-tuning; báo cáo riêng chất lượng cuối và mức cải thiện. Mức tăng lớn nhất không đồng nghĩa với chất lượng cuối cao nhất.

## Phạm vi đã chốt

- **Tác vụ:** Decision Layer của Voice AI; không nghiên cứu thay thế toàn bộ Qwen3.6 hay các tác vụ Reply, STT, TTS, emotion, extraction.
- **Hướng model:** LLM nhỏ, dense, có thể fine-tune. Model-specific shortlist chưa được chốt.
- **Grammar 2.0.0 — đích tương lai chính:** đánh giá toàn bộ 7 caller kinds: `social`, `request`, `reply`, `question`, `stop`, `human`, `unclear`.
- **Grammar 1.3.0 — điều kiện chuyển tiếp:** cùng model phải chạy được cả 11 agent-action kinds: `greet`, `answer`, `route`, `offer`, `clarify`, `transfer`, `escalate`, `reject`, `fallback`, `courtesy`, `close`.
- **Cùng model:** cùng bộ trọng số, nhưng mỗi grammar dùng prompt/output contract riêng; không yêu cầu model tự đoán phiên bản grammar.
- **Ngôn ngữ:** tiếng Anh, theo phạm vi dữ liệu hiện có.
- **Phủ nhãn:** so sánh tổng thể các kind trong từng grammar; chưa xác định kind ưu tiên riêng.
- **Dữ liệu:** có thể đề xuất bổ sung và gán nhãn dữ liệu thật, đặc biệt cho các kind an toàn còn thiếu.

## Đơn vị quyết định, input và output theo tài liệu tham khảo

Tài liệu prompt mô tả Decision như một lần chọn phản hồi cho lượt người gọi hiện tại; hệ thống có thể chạy sớm khi STT tạo segment final. Prompt nhận hội thoại dạng văn bản (các lượt trước và lượt hiện tại), facts đã biết, danh sách kind hợp lệ cho lượt, và reference/capability nếu có. Tài liệu không mô tả audio thô là input trực tiếp của Decision prompt.

Output grammar 1.3.0 là một agent-action kind, có thể kèm reference. Grammar 2.0.0 dùng bộ caller-kind và contract output riêng. Hai grammar diễn tả những khía cạnh khác nhau; benchmark và báo cáo cần tách theo grammar, không gộp thành một accuracy.

Nguồn: [LLM_TASKS_PROMPTS_VI.md](reference-docs/LLM_TASKS_PROMPTS_VI.md), mục 2; [STAGING_OVERVIEW_VI.md](reference-docs/STAGING_OVERVIEW_VI.md), mục 16.2. Theo tài liệu tham khảo, grammar 2.0.0 chưa bật trên staging; cần xác minh trạng thái code/runtime trước khi triển khai.

## Dữ kiện, giả thuyết và giới hạn

### Dữ kiện được mô tả trong reference-docs

- Decision có cấu hình/model path riêng trong hệ thống, trong khi Qwen3.6 là LLM chính cho nhiều tác vụ khác.
- Tài liệu dữ liệu hiện có nêu 207 lượt thật đã được người duyệt và 52 ca tự soạn.
- Tài liệu cảnh báo corpus thật thiếu một số kind an toàn trong grammar hiện tại và có vấn đề về nhãn/ngữ cảnh. Đây là thông tin được báo cáo trong docs; chưa audit lại trên dữ liệu gốc trong project research.

### Giả thuyết cần kiểm chứng

- Fine-tuning trên dữ liệu Decision có thể cải thiện chất lượng so với cùng model trước fine-tuning.
- Model có accuracy ban đầu cao nhất có thể không phải model có mức tăng lớn nhất sau fine-tuning.
- Accuracy tổng thể có thể che lấp hiệu năng yếu ở từng kind hoặc output không hợp lệ.

Các giả thuyết này chưa phải kết luận. Benchmark tổng quát do nhà phát triển model công bố không thay thế đánh giá task-specific.

## Các điểm còn mở trước protocol

- Định nghĩa “chất lượng Decision cần thiết”: metric chính, kết quả theo từng kind, xử lý output nhiều act/định dạng cấu trúc, ngưỡng tối thiểu và lỗi an toàn không chấp nhận.
- Audit nguồn dữ liệu, consistency/coverage nhãn cho cả hai grammar; xác định cần gán nhãn mới bao nhiêu và cách duyệt.
- Chia train/validation/test để tránh leakage theo speaker, call/session và paraphrase.
- Candidate models, cách fine-tune, phần cứng/endpoint và tiêu chí tích hợp.
- Xác nhận grammar, prompt và output contract hiện hành từ code/runtime khi bắt đầu benchmark.

## Ranh giới với ghi chú cũ

[research-brief.md](research-brief.md) là bản khởi tạo trước khi scope được chốt; shortlist và RQ trong đó chưa được duyệt. `DECISION_DENSE_LLM_FINETUNE_PLAN_VI.md` cùng các kế hoạch/suy luận trong research-input docs là proposal lịch sử, không phải final plan. Hãy tạo protocol mới sau khi chốt metric và audit dữ liệu.

