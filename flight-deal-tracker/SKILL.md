---
name: air-deal-radar
description: Theo dõi giá vé máy bay cho các route trong Google Sheet, quét khuyến mãi các hãng bay VN (VietJet, Vietnam Airlines, Bamboo) và tìm kiếm vé giá rẻ, gửi email digest hàng ngày kèm alert khi có deal hot.
---
# Air Deal Radar

**Phiên bản skill: 1.0.** Luôn ghi số này vào dòng `Skill:` trong khối báo cáo cuối email.

## Mục tiêu

Theo dõi giá vé máy bay cho các route user đã cấu hình trong Google Sheet, phát hiện deal rẻ (giảm giá mạnh hoặc dưới ngân sách), quét khuyến mãi hãng bay, rồi gửi email digest tổng hợp.

## Tham số đầu vào

| Tham số | Giá trị hợp lệ | Mặc định | Ghi chú |
| ------- | --------------- | -------- | ------- |
| `mode` | `full` / `promo` / `cleanup` | `full` | `full` = quét đủ nguồn. `promo` = chỉ quét khuyến mãi hãng bay. `cleanup` = dọn price_log cũ hơn 90 ngày |

Không hỏi lại user khi thiếu tham số — skill chạy tự động, không có ai trả lời.

## Quy tắc tự chủ — KHÔNG BAO GIỜ dừng chờ user

Skill này chạy theo lịch lúc user offline. Vì vậy, trong suốt task:
- Không hỏi bất kỳ câu nào — không hỏi xác nhận, không hỏi "có tiếp tục không"
- Không yêu cầu take control / không đề nghị user đăng nhập hộ. Gặp trang đăng nhập, captcha thì ghi nguồn đó là "không truy cập được" và đi tiếp ngay
- Không tự đoán hay tự tìm domain mới
- Không dùng Chrome local — chỉ remote browser
- Gặp lỗi ở một nguồn thì ghi nhận, đi tiếp nguồn sau

## Cấu trúc bộ nhớ: Google Sheet `Flight Deal Tracker`

Sheet có 3 tab:
- `routes`: Cấu hình route muốn theo dõi (origin, destination, route_name, travel_month, return_month, adults, max_budget, active)
- `price_log`: Lưu lịch sử giá mỗi lần quét (route, checked_at, source, cheapest_price, airline, depart_date, return_date, url)
- `deals`: Lưu các deal đã phát hiện để chống gửi alert trùng (route, price, source, sent_at, url)

## Bước 1 — Đọc cấu hình route

Mở `Flight Deal Tracker` -> tab `routes` -> đọc tất cả dòng có `active = TRUE`.
- Bỏ qua các route không hợp lệ hoặc đã hết hạn (travel_month trước tháng hiện tại).
- Đọc tab `deals` để biết deal nào đã gửi trong 3 ngày gần nhất.

## Bước 2 — Tìm kiếm giá vé máy bay

Với mỗi route active:
1. Dùng Google Search hoặc remote browser tra cứu giá vé máy bay rẻ nhất cho chặng bay trong tháng chỉ định.
2. Ghi nhận: giá rẻ nhất (VND), hãng bay, ngày bay.
3. Nếu không tìm thấy hoặc bị lỗi thì ghi "không rõ", chuyển sang route tiếp theo.

## Bước 3 — Quét khuyến mãi hãng bay

Mở lần lượt 3 trang khuyến mãi chính thức của các hãng:
- VietJet: `https://www.vietjetair.com/vi/khuyen-mai`
- Vietnam Airlines: `https://www.vietnamairlines.com/vn/vi/offers`
- Bamboo Airways: `https://www.bambooairways.com/vn/vi/explore/offers`

Đọc danh sách chương trình đang diễn ra: tên chương trình, thời hạn, giá vé từ (nếu có), link chi tiết.

## Bước 4 — So sánh giá & Phát hiện deal

1. So sánh giá hôm nay (`today_price`) với ngân sách (`max_budget`).
2. Tính trend so với các lần quét trước trong `price_log` (lấy 5 dòng gần nhất):
   - Giảm mạnh: giảm từ 15% trở lên so với trung bình 5 ngày
   - Giảm: giảm từ 5% đến dưới 15%
   - Ổn định: thay đổi trong khoảng -5% đến +5%
   - Tăng: tăng từ 5% trở lên
3. Điều kiện Deal:
   - Dưới budget: `today_price <= max_budget`
   - Giảm mạnh: giảm từ 15% trở lên
   - Thấp nhất 5 ngày: giá hôm nay thấp hơn mọi giá trong 5 lần gần nhất
4. Chống trùng: Không gửi alert nếu cùng route và giá sai lệch dưới 50.000đ đã gửi trong 3 ngày qua.

## Bước 5 — Ghi Sheet (Làm trước khi gửi email)

1. Ghi mỗi route một dòng vào cuối tab `price_log`.
2. Ghi các deal mới vào cuối tab `deals`.

## Bước 6 — Gửi email digest

Gửi tới email của user. Dùng HTML sạch:
- Tiêu đề: `[Flight Deals] {dd/MM} — {N} route, {M} deal mới`
- Nội dung gồm:
  - Bảng giá hôm nay (Route, Giá rẻ nhất, Hãng, Ngày bay, Budget, Trend, Link)
  - Danh sách Deal nổi bật (nêu rõ vì sao là deal: dưới budget / giảm mạnh)
  - Danh sách Khuyến mãi hãng bay
  - Khối báo cáo lần chạy (Số route quét, lỗi, số deal mới, Skill: v1.0)
  - Link mở Google Sheet `Flight Deal Tracker`

## Chế độ cleanup — Dọn dẹp

Khi `mode = cleanup`: Xóa các dòng cũ hơn 90 ngày trong `price_log` và `deals` (nếu dữ liệu trên 30 dòng và số dòng xóa không vượt quá 80% tổng số dòng). Gửi email báo cáo kết quả.
