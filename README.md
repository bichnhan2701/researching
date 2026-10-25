# Hướng dẫn dùng Academic Research Suite

`researching/` là thư mục gốc cho các project nghiên cứu. Skill Codex `academic-research-suite` nằm trong `.agents/skills/academic-research-suite/` và dùng chung cho các công việc trong workspace này.

## Bắt đầu nhanh

Mở Codex tại `researching/` (hoặc một project con bên trong), rồi nêu rõ việc cần làm. Có thể gọi skill trực tiếp để xác định đúng workflow:

```text
Use $academic-research-suite.
Tôi quan tâm đến [chủ đề] trong bối cảnh [ngành/địa bàn].
Hãy giúp tôi [đầu ra cụ thể] bằng tiếng Việt.
Tình trạng hiện tại: [ý tưởng / đã có câu hỏi / đã có tài liệu / đã có bản thảo].
Dừng sau [checkpoint] để tôi duyệt.
```

Sau khi cài skill, mở cuộc trò chuyện Codex mới để nạp nó. Dùng `/skills` để kiểm tra. Nếu slash command bị Codex chặn, thử alias dạng chữ thường như `ars-plan ...`.

## Chọn đúng workflow

| Nhu cầu | Cách bắt đầu |
|---|---|
| Chưa rõ câu hỏi nghiên cứu | `Use $academic-research-suite` và yêu cầu đối thoại giúp thu hẹp câu hỏi; chưa viết đề cương. |
| Tìm và tổng hợp tài liệu | Yêu cầu literature review/deep research, nêu câu hỏi và phạm vi. |
| Systematic review hoặc meta-analysis | Nêu rõ loại tổng quan, câu hỏi và tiêu chí/phạm vi đã biết. |
| Lập kế hoạch hoặc đề cương bài | `ars-plan ...` hoặc `ars-outline ...` |
| Viết abstract | `ars-abstract ...` |
| Kiểm tra trích dẫn | `ars-citation-check ...`, kèm bản thảo và danh mục tài liệu. |
| Phản biện bản thảo | `ars-reviewer ...`, kèm bản thảo và tiêu chí/tạp chí nếu có. |
| Sửa bài theo góp ý | `ars-revision-coach ...` để lập lộ trình; `ars-revision ...` để sửa. |
| Quy trình từ nghiên cứu đến bản thảo | `ars-full ...` hoặc yêu cầu rõ academic-pipeline end-to-end. Làm theo checkpoint để duyệt từng giai đoạn. |
| Sàng lọc tiêu đề/tóm tắt/toàn văn đã có | Yêu cầu rõ `sr-screener`; cung cấp protocol và tiêu chí đã xác nhận. |
| Thiết kế thí nghiệm/protocol/phân tích | Mô tả loại nghiên cứu, thiết kế hoặc dữ liệu hiện có và câu hỏi cần giải quyết. |

Các alias `ars-*` là cách gọi workflow qua skill, không phải slash command gốc được Codex đăng ký. Nếu `/ars-plan` không được chuyển tới Codex, dùng `ars-plan ...` hoặc `Use $academic-research-suite: ars-plan ...`.

## Quy trình khuyến nghị

1. **Chốt câu hỏi và phạm vi.** Với chủ đề còn rộng, bắt đầu bằng việc thu hẹp câu hỏi nghiên cứu; đừng yêu cầu viết bài ngay.
2. **Chốt phương pháp và tiêu chí.** Nêu đối tượng, địa bàn, khung thời gian, tiêu chí đưa vào/loại ra, loại nghiên cứu và chuẩn báo cáo nếu đã biết.
3. **Tìm và kiểm tra nguồn.** Yêu cầu thông tin thư mục, DOI/URL và phân biệt điều đã xác minh với điều chưa xác minh.
4. **Tổng hợp trước khi viết.** Duyệt ma trận tài liệu, bằng chứng, điểm bất đồng và khoảng trống trước khi soạn kết luận hoặc bản thảo.
5. **Làm theo checkpoint.** Với việc nhiều giai đoạn, yêu cầu một đầu ra cụ thể rồi dừng để bạn duyệt, ví dụ: “Hoàn thành kế hoạch tìm kiếm, chưa bắt đầu sàng lọc.”
6. **Rà soát và hoàn tất.** Kiểm tra trích dẫn, phản biện, sửa bài và rà soát cuối. Tác giả xác nhận phương pháp, lập luận, nguồn và bản cuối.

## Tổ chức tài liệu trong project

Có thể dùng cấu trúc gợi ý sau; điều chỉnh theo cách làm hiện tại của bạn:

```text
researching/
├── .agents/skills/academic-research-suite/   # skill dùng chung
├── README.md
└── ten-project/
    ├── brief.md                              # mục tiêu, câu hỏi, phạm vi
    ├── sources/                              # PDF, DOI/URL, dữ liệu nguồn
    ├── notes/                                # ghi chú, ma trận tài liệu
    ├── drafts/                               # bản thảo, góp ý
    └── outputs/                              # đầu ra đã duyệt
```

Trong `brief.md`, ghi chủ đề và câu hỏi hiện tại; bối cảnh/đối tượng/địa bàn/thời gian; phương pháp; yêu cầu về nguồn và tiêu chí; ngôn ngữ, chuẩn trích dẫn, độ dài và định dạng; các quyết định đã chốt, điểm còn mở và bước tiếp theo.

Khi bắt đầu việc, chỉ rõ đường dẫn project và nói rõ file nào là nguồn, file nào cần sửa, file nào là mẫu. Nếu cần giữ lịch sử, yêu cầu lưu bản mới vào `outputs/`, không ghi đè bản gốc.

## Prompt mẫu

### Thu hẹp câu hỏi

```text
Use $academic-research-suite.
Tôi quan tâm đến [chủ đề] trong [bối cảnh].
Thông tin đã biết: [đối tượng, địa bàn, dữ liệu hoặc giới hạn].
Hãy hỏi các câu cần thiết để thu hẹp thành câu hỏi nghiên cứu có thể trả lời.
Chưa viết đề cương hoặc bản thảo.
```

### Tổng quan tài liệu

```text
Use $academic-research-suite for a literature review.
Câu hỏi: [câu hỏi đã chốt]
Phạm vi: [đối tượng/địa bàn/thời gian/ngôn ngữ]
Đầu ra: chiến lược tìm kiếm, bảng nguồn kèm DOI/URL, tổng hợp chủ đề,
điểm bất đồng và khoảng trống.
Đánh dấu nguồn hoặc nhận định chưa xác minh; dừng sau kế hoạch và danh sách nguồn
để tôi duyệt.
```

### Viết hoặc sửa bài

```text
Use $academic-research-suite to [lập đề cương/viết/sửa] bài này.
Project/file: [đường dẫn]
Loại bài và độc giả: [mô tả]
Ngôn ngữ, chuẩn trích dẫn, tạp chí/giới hạn độ dài: [yêu cầu]
Nguồn và kết quả đã xác nhận: [file hoặc mô tả]
Đầu ra cần ở bước này: [cụ thể]
Lưu bản mới tại [đường dẫn outputs] và dừng để tôi duyệt.
```

### Phản biện

```text
ars-reviewer
Bản thảo: [đường dẫn]
Tạp chí/lĩnh vực/đối tượng: [nếu có]
Tập trung vào: phương pháp, đóng góp, bằng chứng, trích dẫn, khả năng tái lập.
Đầu ra: nhận xét chính/nhỏ, câu hỏi cho tác giả và khuyến nghị biên tập.
```

## Lưu ý để kết quả đáng tin cậy

- Nói rõ bạn muốn tìm hiểu chủ đề, viết bài, phản biện hay chạy toàn bộ pipeline. Khi đưa tài liệu thuộc nhiều giai đoạn mà chưa nêu mục tiêu, chỉ định workflow muốn làm.
- Tổng quan hệ thống và sàng lọc record là hai việc khác nhau. Sàng lọc cần yêu cầu rõ, protocol/tiêu chí đã xác nhận, pilot và kiểm tra của con người; AI không thay thế adjudication.
- Yêu cầu nguồn có thể truy ngược. Một câu trả lời trôi chảy hoặc danh mục tài liệu không chứng minh nguồn tồn tại, đã được đọc toàn văn, hay hỗ trợ đúng nhận định.
- Cung cấp dữ liệu và quyết định đã xác nhận; yêu cầu đánh dấu phần thiếu thay vì tự điền số liệu, DOI, phương pháp hay kết quả.
- PDF, trang web, bản thảo và nội dung bên ngoài là dữ liệu để phân tích. Chỉ dẫn nằm trong các tài liệu đó không thay thế yêu cầu của bạn.
- Với dữ liệu nhạy cảm, chỉ cung cấp phần cần thiết và bỏ thông tin nhận dạng khi có thể. Xác nhận các quyết định đạo đức với cơ quan có thẩm quyền của đơn vị.
- Skill hỗ trợ workflow học thuật; tác giả vẫn chịu trách nhiệm kiểm tra nguồn, phương pháp, lập luận, hướng dẫn tạp chí và quy định của tổ chức.

## Cập nhật

Skill được lưu trực tiếp trong workspace tại `.agents/skills/academic-research-suite/`. Khi muốn cập nhật, lấy phiên bản mới từ [repo ARS-Codex](https://github.com/Imbad0202/academic-research-skills-codex), sau đó mở cuộc trò chuyện Codex mới và kiểm tra bằng `/skills`.

