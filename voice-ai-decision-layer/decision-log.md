# Decision Log — Voice AI Decision Layer

Ghi các quyết định đã được người dùng xác nhận. Không xem giả thuyết hoặc gợi ý của AI là quyết định đã chốt.

| Ngày | Quyết định | Lý do/bằng chứng | Trạng thái |
|---|---|---|---|
| 2026-10-25 | Tạo project nghiên cứu riêng cho Decision Layer trong Voice AI | Yêu cầu của người dùng | Đã xác nhận |
| 2026-10-25 | Shortlist 8 model từ trao đổi trước | Cần xác minh dense architecture, phiên bản, license và benchmark; không phải shortlist được duyệt | Lịch sử/chưa duyệt |
| 2026-10-26 | Một LLM nhỏ, dense, fine-tune được cần đạt chất lượng Decision trên cả grammar 2.0.0 và 1.3.0 để cùng model đảm nhận Decision thay cho Qwen3.6 | Câu hỏi nghiên cứu được người dùng xác nhận | Đã xác nhận |
| 2026-10-26 | Grammar 2.0.0 là đích tương lai chính; grammar 1.3.0 là track chuyển tiếp; dùng cùng bộ trọng số nhưng prompt/output contract riêng | Người dùng xác nhận trong hội thoại | Đã xác nhận |
| 2026-10-26 | Phạm vi ngôn ngữ là tiếng Anh; đánh giá tổng thể mọi kind; có thể đề xuất bổ sung/gán nhãn dữ liệu thật cho kind còn thiếu | Người dùng xác nhận trong hội thoại | Đã xác nhận |
| 2026-10-26 | Brief khởi tạo, shortlist và plan cũ không phải scope/plan cuối | User note: plan có thể chưa cập nhật; scope hiện hành ở research-question-brief.md | Đã xác nhận |
| — | Đối chiếu prompt/grammar mô tả với code và trạng thái runtime khi bắt đầu benchmark | Reference docs chưa xác nhận live staging | Chưa làm |
| — | Chốt primary metric, test split và ngưỡng triển khai | Cần Decision Task Specification và chi phí lỗi | Chưa chốt |
| — | Chọn model/method cho fine-tuning | Chỉ quyết định sau zero-shot benchmark và tiêu chí đã thống nhất | Chưa chốt |

