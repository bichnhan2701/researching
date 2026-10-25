# Research Brief — Decision Layer cho Voice AI

> **Bản khởi tạo đã được thay thế làm scope hiện hành.** Dùng [research-question-brief.md](research-question-brief.md) làm brief đã xác nhận ngày 2026-10-26. Các shortlist, RQ và giả thuyết bên dưới là ghi chú lịch sử/chưa duyệt; không xem là quyết định hoặc plan cuối.

**Trạng thái:** Bản khởi tạo lịch sử — scope hiện hành đã được xác nhận tại `research-question-brief.md`  
**Nguồn khởi tạo:** phần preview cuộc trò chuyện “Định hướng research Decision” được người dùng cung cấp. Preview bị giới hạn, vì vậy brief này chỉ tổng hợp các chi tiết nhìn thấy; cần bổ sung nếu cuộc trò chuyện còn quyết định khác chưa xuất hiện trong preview.

## 1. Bối cảnh đã biết từ trao đổi trước

- Đề tài thuộc project **Voice AI**, tập trung vào **Decision Layer**.
- Hướng thảo luận trước là so sánh các nhóm model nhỏ và fine-tuning, với hai mục tiêu riêng:
  1. Model có accuracy zero-shot/tổng quát cao nhất để làm mốc chất lượng.
  2. Model có thể cải thiện accuracy sau fine-tuning.
- Shortlist được đề xuất trong trao đổi trước:

| Nhóm theo trao đổi trước | Ứng viên zero-shot/accuracy cao | Ứng viên thử fine-tuning |
|---|---|---|
| A — 0.6B–2B | Qwen3.5-2B | Qwen3.5-0.8B |
| B — 3B–4B | Qwen3-4B-Instruct-2507 | Qwen2.5-3B-Instruct |
| C — 7B–9B | Qwen3.5-9B | Qwen3-8B |
| D — baseline nhỏ | SmolLM2-1.7B-Instruct | Qwen2.5-0.5B-Instruct |

- Trao đổi trước đã nhấn mạnh đây là shortlist ứng viên, **không phải kết luận từ benchmark thực tế trên dữ liệu Decision**.
- Quy trình sơ bộ đã nhắc đến zero-shot screening, benchmark theo task, fine-tuning, latency trên GB10 và shadow deployment. Cần xác nhận GB10 có còn là mục tiêu triển khai hay không.
- Trao đổi trước có nhắc đến khả năng sử dụng thương mại của một số model. Tình trạng license/version không được coi là đã xác minh cho project này; cần kiểm tra lại model repo và license đúng phiên bản trước khi dùng thương mại.

## 2. Giả thuyết cần kiểm chứng — chưa phải kết luận

1. Model lớn hơn có thể đạt chất lượng zero-shot cao hơn, nhưng lợi thế có thể thay đổi theo taxonomy, prompt, dữ liệu và điều kiện triển khai của Decision Layer.
2. Model nhỏ hơn có thể có mức tăng tương đối lớn sau fine-tuning, nhưng mức tăng đó không đảm bảo chất lượng cuối cùng cao nhất.
3. Accuracy tổng thể có thể che giấu các lỗi action nguy hiểm, lớp hiếm, output không hợp lệ hoặc thất bại với transcript lỗi.
4. Model phù hợp nhất phải được chọn theo chất lượng quyết định và ràng buộc latency/footprint/cost/safety thực tế, không chỉ theo số tham số hay benchmark tổng quát.

## 3. Câu hỏi nghiên cứu dự thảo

**RQ1.** Với cùng Decision task, dữ liệu, prompt/output schema và môi trường inference, ứng viên nào đạt trade-off tốt nhất giữa chất lượng quyết định, latency và chi phí?

**RQ2.** Fine-tuning trên dữ liệu Decision đã gán nhãn làm thay đổi chất lượng của từng ứng viên như thế nào so với baseline chưa fine-tune, trên một test set giữ kín và tách biệt?

**RQ3 (phụ, cần chốt).** Các model khác nhau ra sao về lỗi theo action/intent, output sai schema, khả năng abstain/escalate, robustness với lỗi ASR và trường hợp mơ hồ?

RQ1–RQ3 là khung khởi đầu. Cần xác nhận đơn vị dự đoán, action space, nhóm người dùng, dữ liệu, ngôn ngữ và deployment target trước khi chốt câu hỏi cuối.

## 4. Phạm vi cần làm rõ trước khi benchmark

- **Đầu vào:** audio thô hay transcript ASR? Có lịch sử hội thoại, metadata hoặc state từ tool không?
- **Đầu ra:** intent/class, chọn action/tool, tham số có cấu trúc, câu trả lời, hay quyết định hỏi lại/chuyển người?
- **Taxonomy:** các action/label hiện có là gì? Có label hiếm, out-of-scope hoặc “không chắc” không?
- **Sai số:** false action, bỏ sót action, tham số sai và trì hoãn khác nhau thế nào về chi phí/hậu quả?
- **Dữ liệu:** dataset hiện có bao nhiêu mẫu, nguồn, ngôn ngữ/accent, chất lượng nhãn, quyền sử dụng và cách tách train/validation/test?
- **Triển khai:** phần cứng (GB10 hay mục tiêu khác), latency/throughput, bộ nhớ, chi phí, offline/privacy và yêu cầu fallback?
- **Fine-tuning:** SFT/LoRA/QLoRA hay phương pháp khác; có giới hạn VRAM, thời gian và quyền phân phối adapter không?
- **Thương mại hóa:** phiên bản repo và license hiện hành của từng model/adapter; điều kiện notice/attribution và chính sách dữ liệu huấn luyện.

## 5. Thiết kế nghiên cứu sơ bộ

1. Chốt Decision Task Specification và annotation guideline.
2. Rà soát nguồn chính thức cho từng candidate: đúng biến thể instruct, model card, license, context length, kiến trúc và hỗ trợ inference.
3. Xây benchmark với split chống leakage (ví dụ theo speaker/session/customer, tùy dữ liệu), nhãn được kiểm tra và test set giữ kín.
4. Chạy baseline theo điều kiện cố định; lưu phiên bản model, prompt, decoding, phần cứng và mã đánh giá.
5. Chọn ứng viên fine-tune theo tiêu chí đã định trước; fine-tune và đánh giá lại trên cùng held-out test set.
6. Phân tích theo lớp và nhóm lỗi; đánh giá robustness, latency, memory và cost.
7. Nếu kết quả đạt ngưỡng, shadow evaluation/triển khai giới hạn với fallback, giám sát và rollback.

## 6. Quy tắc báo cáo

- Tách kết quả thực nghiệm khỏi benchmark do nhà phát triển model công bố.
- Ghi nguồn, phiên bản, ngày truy cập, cấu hình chạy và tập dữ liệu cho mọi con số.
- Không gọi một model là “tốt nhất” nếu chưa nêu metric, test distribution và ràng buộc.
- Báo cáo cả chất lượng sau fine-tuning lẫn mức thay đổi so với baseline.
- Ghi rõ dữ liệu thiếu, bất định, lớp ít mẫu và giới hạn khả năng khái quát.

