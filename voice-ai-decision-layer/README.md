# Voice AI — Decision Layer

Nghiên cứu này tìm một **LLM nhỏ, dense và có thể fine-tune** để đảm nhận tác vụ Decision thay cho phần Decision hiện giao cho Qwen3.6. Cùng bộ trọng số phải chạy được grammar 2.0.0 (đích tương lai) và grammar 1.3.0 (điều kiện chuyển tiếp), với prompt/output contract riêng cho mỗi grammar. Qwen3.6 vẫn là LLM chính cho các tác vụ khác.

Scope hiện hành đã được xác nhận trong [research-question-brief.md](research-question-brief.md). [research-brief.md](research-brief.md) là bản khởi tạo cũ; shortlist và RQ trong đó chưa được duyệt. Candidate model từ trao đổi trước chưa được xác minh độc lập hoặc benchmark trên dữ liệu Decision của project.

## Bắt đầu bằng skill

Mở Codex tại workspace `researching/` hoặc thư mục project này. Dùng prompt sau để bắt đầu vòng đầu tiên:

```text
Use $academic-research-suite.
Đọc research-question-brief.md trong project voice-ai-decision-layer/ (nếu
workspace đang mở tại project này, file nằm ở thư mục hiện tại) và tiếp tục từ
scope đã được xác nhận. Giúp tôi xác định metric chất lượng Decision và ngưỡng
chấp nhận; tách dữ kiện, giả thuyết và thông tin còn thiếu.
Chưa tìm thêm model, chưa chốt shortlist, chưa bắt đầu benchmark hay viết bài.
```

Khi câu hỏi nghiên cứu đã được xác nhận, tiếp tục từng bước dưới đây. Mỗi lần chỉ giao một đầu ra và yêu cầu dừng tại checkpoint.

## Lộ trình nghiên cứu đề xuất

### 1. Chốt bài toán và tiêu chí quyết định

Làm rõ:

- Decision Layer nhận **audio thô**, transcript từ ASR, transcript kèm ngữ cảnh hội thoại, hay một dạng biểu diễn khác?
- Nó dự đoán intent, chọn tool/action, điền tham số, quyết định hỏi lại/abstain, hay kết hợp các chức năng trên?
- Danh sách action/label có đóng và ổn định không? Output có schema/JSON bắt buộc không?
- Hậu quả của false positive, false negative, action nguy hiểm và abstention là gì?
- Ràng buộc triển khai: phần cứng đích (ví dụ GB10 nếu vẫn là mục tiêu), latency, throughput, bộ nhớ, chi phí và yêu cầu riêng tư là gì?

**Đầu ra:** Decision Task Specification, taxonomy/action schema, tiêu chí thành công và ma trận chi phí lỗi. Chưa so sánh model nếu các định nghĩa này chưa rõ.

### 2. Tổng quan tài liệu và xác minh ứng viên

Yêu cầu skill tìm nguồn chính thức và nghiên cứu độc lập cho:

- Mô hình nhỏ làm intent/action classification hoặc structured decision trong voice/conversational systems.
- Fine-tuning (ví dụ SFT/LoRA) cho tác vụ phân loại/quyết định theo schema.
- Phương pháp đánh giá, calibration/abstention, robustness với lỗi ASR và latency triển khai.

Các model trong shortlist cũ được ghi ở brief như **ứng viên cần xác minh**. Hãy kiểm tra đúng model card, license, biến thể instruct, benchmark và khả năng chạy trên phần cứng thực tế; không coi benchmark tổng quát là Decision accuracy.

**Đầu ra:** bảng evidence có nguồn/DOI/URL, ngày và phạm vi kiểm tra; ghi riêng kết luận đã xác minh, chưa xác minh và hạn chế. Có thể bắt đầu bằng `ars-3w` cho một nhóm paper nhỏ, rồi dùng literature review nếu cần tổng hợp rộng hơn.

### 3. Xây dựng benchmark trước fine-tuning

- Chốt guideline gán nhãn, xử lý trường hợp mơ hồ, out-of-scope và cần chuyển cho người.
- Giữ tập test tách biệt theo người dùng/cuộc hội thoại/phiên (và thời gian nếu phù hợp) để tránh rò rỉ paraphrase hoặc cùng một người sang cả train lẫn test.
- Tạo baseline có thể tái lập; cố định prompt, context, decoding, schema, phiên bản model và môi trường chạy.
- So sánh model với cùng dữ liệu và cùng điều kiện. Đưa ra chỉ số theo từng lớp, không chỉ một accuracy tổng thể.

**Chỉ số nên cân nhắc:** accuracy, macro-F1, precision/recall từng action, confusion matrix, tỷ lệ output sai schema/tham số, tỷ lệ abstain/escalate, calibration nếu có confidence, robustness với nhiễu/lỗi transcript ASR, p50/p95 latency, throughput, bộ nhớ và chi phí.

### 4. Thử fine-tuning có đối chứng

So sánh zero-shot/in-context baseline và fine-tuned model trên **cùng test set giữ kín**. Ghi rõ tập dữ liệu, phiên bản, cách tạo nhãn, recipe, hyperparameters và seed. Báo cáo mức tăng/giảm theo metric và nhóm lỗi, không chỉ benchmark tổng.

Model “cải thiện nhiều nhất” và model “tốt nhất sau fine-tuning” là hai câu hỏi khác nhau; nên báo cáo cả:

- Mức thay đổi tuyệt đối và tương đối từ baseline.
- Chất lượng cuối cùng sau fine-tuning.
- Khoảng bất định/độ ổn định giữa các lần chạy nếu thiết kế cho phép.
- Trade-off latency, footprint và chi phí.

### 5. Kiểm tra ngoài benchmark và chốt khuyến nghị

Kiểm tra trên dữ liệu/điều kiện chưa thấy, lỗi ASR, câu nói ngắn/mơ hồ, ngôn ngữ/accent thực tế, và action rủi ro cao. Nếu phù hợp, chạy shadow evaluation trước khi quyết định triển khai. Xác định ngưỡng chấp nhận, fallback/abstain và rollback trước khi thử nghiệm.

## Cách yêu cầu các giai đoạn tiếp theo

Khi đã duyệt phạm vi, có thể tiếp tục bằng những yêu cầu cụ thể như:

- `Use $academic-research-suite for a literature review on small dense LLMs for Decision in Voice AI. Phạm vi đã chốt nằm trong research-question-brief.md. Trả về evidence matrix, tách bằng chứng theo grammar và dừng trước khi đề xuất model.`
- `ars-3w: so sánh các paper tôi cung cấp theo WHY/HOW/WHAT; đánh dấu điều chưa xác minh.`
- `Thiết kế benchmark zero-shot cho các model trong shortlist theo Decision Task Specification; chưa chạy inference và chưa sửa dữ liệu.`
- `Lập protocol thử SFT/LoRA để đo mức cải thiện trên Decision dataset, gồm split tránh leakage, baseline, metric, ablation và điều kiện dừng.`
- `Review protocol/benchmark này về leakage, nhãn không nhất quán, metric không phù hợp và rủi ro triển khai.`

Không yêu cầu `ars-full` ngay từ đầu. Trước tiên hãy chốt task và RQ; sau đó dùng pipeline end-to-end nếu thật sự cần đi từ research đến manuscript.

## Cấu trúc thư mục

```text
voice-ai-decision-layer/
├── README.md                 # hướng dẫn làm project
├── research-question-brief.md # scope/RQ đã xác nhận
├── research-brief.md         # bản khởi tạo lịch sử, chưa duyệt
├── decision-log.md           # quyết định và thay đổi có xác nhận
├── sources/                  # model cards, papers, tài liệu nguồn
├── data/                     # mô tả/schema; hạn chế lưu dữ liệu nhạy cảm
├── protocols/                # annotation, benchmark, fine-tuning protocols
├── results/                  # kết quả thực nghiệm và phân tích
└── outputs/                  # báo cáo/bản thảo đã duyệt
```

Không đưa dữ liệu giọng nói hoặc transcript nhạy cảm vào project nếu chưa được phép theo quy định của tổ chức. Lưu ý hướng dẫn về cách dùng skill chung trong [README gốc](../README.md).

