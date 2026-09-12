# gemini-spark-setup

Bộ Skill và hướng dẫn setup để tự động hoá công việc bằng **Gemini Spark**.

Mỗi automation nằm trong một thư mục riêng, gồm:

- `SKILL.md` — nội dung skill, dán thẳng vào **Spark → Skills → Create skill**
- Tài liệu setup từng bước (tạo Sheet, Gmail label, Task, Schedule, xử lý sự cố)

## Automation hiện có

| Thư mục | Việc làm | Trạng thái |
|---|---|---|
| [`vn-data-job-radar/`](vn-data-job-radar/) | Quét tin tuyển dụng ngành data (DA / AE / DE / BI / BA) tại Hà Nội và TP.HCM từ Gmail Job Alerts + 5 trang tuyển dụng + career page, chống trùng bằng Google Sheet, gửi email tổng hợp 2 lần/ngày; task dọn dẹp sheet riêng mỗi thứ Hai | Bản 3.1 |

## Bắt đầu

1. Vào thư mục automation muốn dùng
2. Đọc file hướng dẫn (`gemini-spark-*.md`) và làm theo **Phần 0 — Setup từng bước**
3. Dán `SKILL.md` vào Spark khi hướng dẫn yêu cầu

Cần gói Google AI Pro/Ultra, tài khoản Google cá nhân, và đã bật Connected Apps (Google Workspace).

## Nguyên tắc chung khi viết skill

- **Bộ nhớ nằm ngoài agent** — trạng thái giữa các lần chạy lưu ở Google Sheet, không dựa vào trí nhớ thread.
- **Ghi trước, gửi sau** — ghi sheet xong mới gửi email, để lỗi gửi không gây trùng ở lần sau.
- **Remote browser mặc định** — task chạy lúc máy tắt, không phụ thuộc Chrome local. Nguồn nào chặn bot thì bù bằng email alert.
- **Thao tác xoá phải có guard** — mọi bước xoá dữ liệu trên sheet đều có ngưỡng an toàn và điều kiện chạy rõ ràng.
- **Có health check trong output** — email luôn kèm số liệu để user tự biết hệ thống còn sống (ví dụ `SEEN_COUNT`).

## Thêm automation mới

Tạo thư mục mới cùng cấu trúc: `<tên-skill>/SKILL.md` + `<tên-skill>/gemini-spark-<tên-skill>.md`, rồi thêm một dòng vào bảng trên.
