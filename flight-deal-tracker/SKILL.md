---
name: flight-deal-tracker
description: Theo dõi giá vé máy bay cho các route do user cấu hình trong Google Sheet, quét Skyscanner, Traveloka và trang khuyến mãi hãng bay VN, so sánh giá với lần quét trước, gửi email digest hàng ngày kèm alert khi giá giảm hoặc dưới ngân sách. Dùng khi cần theo dõi giá vé để mua đúng lúc rẻ.
---
# Flight Deal Tracker

**Phiên bản skill: 1.0.** Luôn ghi số này vào dòng `Skill:` trong khối báo cáo cuối email.

## Mục tiêu

Theo dõi giá vé máy bay cho các route user đã cấu hình trong Google Sheet,
phát hiện deal rẻ (giảm giá mạnh hoặc dưới ngân sách), quét khuyến mãi hãng
bay, rồi gửi email digest tổng hợp.

## Tham số đầu vào

| Tham số | Giá trị hợp lệ | Mặc định | Ghi chú |
| ------- | --------------- | -------- | ------- |
| `mode` | `full` / `promo` / `cleanup` | `full` | `full` = quét Skyscanner + Traveloka + promo hãng bay. `promo` = chỉ quét trang khuyến mãi, bỏ Skyscanner/Traveloka — nhanh, dùng khi chỉ muốn xem flash sale. `cleanup` = dọn `price_log` cũ hơn 90 ngày, không quét gì |

Không hỏi lại user khi thiếu tham số — skill chạy tự động, không có ai trả lời.

## Quy tắc tự chủ — KHÔNG BAO GIỜ dừng chờ user

Skill này chạy theo lịch lúc user offline. Vì vậy, trong suốt task:

- **Không hỏi bất kỳ câu nào** — không hỏi xác nhận, không hỏi "có tiếp tục không"
- **Không yêu cầu "take control"** / không đề nghị user đăng nhập hộ. Gặp trang đăng
  nhập, captcha → ghi nguồn đó là "không truy cập được" và đi tiếp ngay
- **Không tự đoán hay tự tìm domain mới.** Chỉ mở đúng những domain đã liệt kê sẵn:
  `skyscanner.com.vn`, `traveloka.com`, `vietjetair.com`, `vietnamairlines.com`,
  `bambooairways.com`, và `google.com` (Gmail/Sheets). Danh sách cố định.
- **Không dùng Chrome local** — chỉ remote browser
- Gặp lỗi ở một nguồn → ghi nhận, đi tiếp nguồn sau

## Định dạng dữ liệu chuẩn

| Trường | Định dạng | Ví dụ |
| ------ | --------- | ----- |
| `checked_at`, `sent_at` | ISO 8601 có múi giờ | `2026-09-20T08:00:00+07:00` |
| `cheapest_price`, `max_budget` | Số nguyên VND, không dấu phân cách | `2450000` |
| `route` | `{ORIGIN}-{DEST}` viết hoa | `HAN-BKK` |
| `travel_month`, `return_month` | `yyyy-MM` | `2026-12` |
| `depart_date`, `return_date` | `yyyy-MM-dd` hoặc `không rõ` | `2026-12-05` |
| `source` | Đúng một trong: `skyscanner`, `traveloka`, `vietjet_promo`, `vna_promo`, `bamboo_promo` | `skyscanner` |

Trong email, giá hiển thị có dấu chấm phân cách + đ: `2.450.000đ`.
Trong Sheet, giá là số nguyên thuần: `2450000`.

Mọi phép tính dùng múi giờ Asia/Ho_Chi_Minh (GMT+7).

## Cấu trúc bộ nhớ: Google Sheet `Flight Deal Tracker`

Sheet có 3 tab:

| Tab | Cột | Vai trò |
| --- | --- | ------- |
| `routes` | A–H | **CHỈ ĐỌC.** User cấu hình route muốn theo dõi |
| `price_log` | A–H | **GHI** mỗi lần quét + **đọc 5 dòng gần nhất** mỗi route (cho trend). Mode `cleanup` được đọc toàn bộ |
| `deals` | A–E | **ĐỌC + GHI.** Chống gửi alert trùng |

Header của `routes`:
`origin | destination | route_name | travel_month | return_month | adults | max_budget | active`

Header của `price_log`:
`route | checked_at | source | cheapest_price | airline | depart_date | return_date | url`

Header của `deals`:
`route | price | source | sent_at | url`

## Mã sân bay

Dùng IATA code 3 chữ cái viết hoa (HAN, SGN, BKK, ICN…). Khi construct URL
Skyscanner, lowercase toàn bộ code.

**Mã Skyscanner đặc biệt cho thành phố nhiều sân bay:**

| Thành phố | Mã | Gồm sân bay |
| --------- | -- | ----------- |
| Bangkok | `BKKT` | BKK + DMK |
| Tokyo | `TYOA` | NRT + HND |
| Seoul | `SELA` | ICN + GMP |
| Kuala Lumpur | `KULM` | KUL + SZB |

User ghi `BKK` → Skyscanner dùng `bkk`. User ghi `BKKT` → dùng `bkkt`.
Traveloka không hỗ trợ mã thành phố — gặp 4 chữ thì lấy sân bay chính
(ví dụ `BKKT` → `BKK`).

## Bước 1 — Đọc cấu hình route (LÀM ĐẦU TIÊN, KHÔNG BỎ QUA)

Mở `Flight Deal Tracker` → tab `routes` → đọc tất cả dòng có `active = TRUE`.

Với mỗi route, validate:

- `origin` và `destination`: 3 chữ cái viết hoa (IATA) hoặc 4 chữ (mã thành phố
  Skyscanner, xem bảng trên)
- `travel_month`: đúng `yyyy-MM` và **chưa qua** (>= tháng hiện tại)
- `return_month`: rỗng (one-way) hoặc đúng `yyyy-MM` và >= `travel_month`
- `adults`: số nguyên 1–9
- `max_budget`: số nguyên > 0 (VND)

Route không hợp lệ → bỏ qua, ghi vào báo cáo cuối email (tên route + lý do).

**Route hết hạn:** nếu `travel_month` đã qua (trước tháng hiện tại), route đó
không hợp lệ. Đếm số route hết hạn (`EXPIRED_COUNT`). Nếu `EXPIRED_COUNT > 0`,
ghi dòng riêng trong khối báo cáo cuối email:
`Route hết hạn: {EXPIRED_COUNT} ({danh sách route_name}) — cập nhật travel_month trong Sheet`

Đọc thêm tab `deals` để biết deal nào đã gửi (chống alert trùng).

Ghi lại số route hợp lệ (`ROUTE_COUNT`) để báo cáo cuối email.

Nếu sheet hoặc tab không tồn tại, tạo mới theo đúng cấu trúc rồi gửi email
"chưa có route nào — vui lòng thêm route vào tab routes".

## Bước 2 — Quét Skyscanner (nguồn chính)

**Nếu `mode = promo`: bỏ qua bước này.**

Với mỗi route active, construct URL monthly cheapest view:

**One-way** (khi `return_month` rỗng):

```
https://www.skyscanner.com.vn/transport/flights/{origin_lower}/{dest_lower}/{yymm}/?adults={adults}&adultsv2={adults}&cabinclass=economy&currency=VND&locale=vi-VN&market=VN&rtn=0
```

**Round-trip** (khi `return_month` có giá trị):

```
https://www.skyscanner.com.vn/transport/flights/{origin_lower}/{dest_lower}/{depart_yymm}/{return_yymm}/?adults={adults}&adultsv2={adults}&cabinclass=economy&currency=VND&locale=vi-VN&market=VN&rtn=1
```

Trong đó `{yymm}` là 2 số cuối năm + 2 số tháng. Ví dụ `2026-12` → `2612`.

**Ví dụ:**

Route HAN → BKK, tháng 12/2026, 1 người, one-way:
```
https://www.skyscanner.com.vn/transport/flights/han/bkk/2612/?adults=1&adultsv2=1&cabinclass=economy&currency=VND&locale=vi-VN&market=VN&rtn=0
```

Route SGN → ICN, đi 01/2027 về 01/2027, 2 người, round-trip:
```
https://www.skyscanner.com.vn/transport/flights/sgn/icn/2701/2701/?adults=2&adultsv2=2&cabinclass=economy&currency=VND&locale=vi-VN&market=VN&rtn=1
```

**Cách đọc trang:**

1. Mở URL bằng remote browser
2. Chờ trang tải xong (JavaScript render) — tối đa 30 giây
3. Đọc danh sách kết quả: hãng bay, giá, thời gian bay, số chặng dừng, ngày
   khởi hành, ngày về (nếu round-trip)
4. **Chỉ ghi nhận 5 kết quả rẻ nhất** — không cuộn, không bấm "Load more"
5. Lấy giá rẻ nhất làm `cheapest_price` của route ở nguồn này

**Chờ 5 giây giữa mỗi route** để tránh rate-limit.

**Trần:**

- Tối đa **3 phút** cho mỗi route
- Tối đa **10 route** mỗi lần quét Skyscanner. Route thứ 11+ → ghi "bỏ qua vì
  chạm trần" trong báo cáo
- Nếu trang hiện captcha, "Access Denied", "Attention Required", hoặc trang trống
  sau 30 giây → ghi "Skyscanner không truy cập được" cho route đó, đi tiếp route sau.
  **Không** reload, không chờ captcha, không thử URL khác

## Bước 3 — Quét Traveloka (nguồn phụ, cross-check)

**Nếu `mode = promo`: bỏ qua bước này.**

Traveloka cần ngày cụ thể, không có monthly view. Với mỗi route, tạo **2 URL**
cho ngày 1 và ngày 15 của `travel_month`:

**One-way:**

```
https://www.traveloka.com/vi-vn/flight/fullsearch?ap={ORIGIN}.{DEST}&dt={DD-MM-YYYY}&ps={adults}.0.0&sc=ECONOMY
```

**Round-trip:** thêm ngày về (ngày 1 hoặc ngày 15 của `return_month`):

```
https://www.traveloka.com/vi-vn/flight/fullsearch?ap={ORIGIN}.{DEST}&dt={DEPART_DD-MM-YYYY}.{RETURN_DD-MM-YYYY}&ps={adults}.0.0&sc=ECONOMY
```

Trong đó `{DD-MM-YYYY}` là ngày-tháng-năm. Ví dụ `01-12-2026`.

**Ví dụ:**

Route HAN → BKK, ngày 01/12/2026, one-way:
```
https://www.traveloka.com/vi-vn/flight/fullsearch?ap=HAN.BKK&dt=01-12-2026&ps=1.0.0&sc=ECONOMY
```

Route HAN → BKK, ngày 15/12/2026, one-way:
```
https://www.traveloka.com/vi-vn/flight/fullsearch?ap=HAN.BKK&dt=15-12-2026&ps=1.0.0&sc=ECONOMY
```

Nếu destination dùng mã thành phố 4 chữ (ví dụ `BKKT`), lấy sân bay chính
(ví dụ `BKK` cho Bangkok).

**Cách đọc trang:**

1. Mở URL bằng remote browser, chờ tải xong — tối đa 30 giây
2. Đọc **3 kết quả rẻ nhất**: hãng, giá, thời gian, số chặng
3. So sánh 2 ngày mẫu, ghi nhận giá rẻ hơn

**Chờ 5 giây giữa mỗi URL.**

**Trần:**

- Tối đa **2 phút** mỗi route (cả 2 URL)
- Tối đa **2 URL** mỗi route (ngày 1 + ngày 15)
- Nếu Skyscanner đã có giá cho route này → Traveloka chỉ để cross-check.
  Lỗi ở Traveloka không ảnh hưởng deal detection miễn Skyscanner có dữ liệu
- Captcha / trang trống / lỗi → ghi "Traveloka không truy cập được", đi tiếp

## Bước 4 — Quét khuyến mãi hãng bay (Deal Discovery)

Mở lần lượt 3 trang khuyến mãi cố định:

```
https://www.vietjetair.com/vi/khuyen-mai
https://www.vietnamairlines.com/vn/vi/offers
https://www.bambooairways.com/vn/vi/explore/offers
```

Mỗi trang:

1. Chờ trang tải xong — tối đa 30 giây
2. Đọc danh sách chương trình khuyến mãi **đang diễn ra** trên trang danh sách
3. Với mỗi chương trình: tên, mô tả ngắn (1 câu), thời hạn (nếu có),
   giá từ (nếu có), link tới trang chi tiết
4. **Không bấm vào từng chương trình** — chỉ đọc trang danh sách

**Trần:** tối đa **2 phút** mỗi trang. Captcha / trang lỗi / không tải được →
ghi "không truy cập được", đi tiếp.

**Chờ 3 giây giữa mỗi trang.**

**URL trên tra cứu tại thời điểm viết skill, có thể đổi theo thời gian.** Lần
chạy test đầu tiên, kiểm tra từng URL còn mở được không; URL nào lỗi thì user
tự thay bằng URL đúng, sửa trực tiếp trong Skill.

**Nếu `mode = promo`:** chỉ làm bước này, bỏ Bước 2 và 3. Vẫn đọc routes từ
Sheet ở Bước 1 (để biết route nào user quan tâm), nhưng không quét giá.

## Bước 5 — So sánh giá & Phát hiện deal

### 5.1 Chọn giá tốt nhất hôm nay

Với mỗi route, so sánh giá từ Skyscanner và Traveloka:

- Lấy giá rẻ nhất → `today_price`
- Nguồn cho giá rẻ nhất → `today_source`
- Hãng bay → `today_airline`
- Ngày bay → `today_depart`, `today_return`

Nếu cả 2 nguồn đều lỗi → route này không có dữ liệu, ghi `today_price = không rõ`.

### 5.2 Tính trend (so với các lần quét trước)

Với mỗi route, tìm giá các lần quét trước bằng cách: cuộn xuống cuối tab
`price_log`, đọc ngược lên, lấy **5 dòng gần nhất** có cùng giá trị cột `route`.
**Bỏ qua dòng có `cheapest_price = 0`** (đó là lần quét lỗi, không phải giá thật).
Dừng đọc ngay khi đã đủ 5 dòng hoặc đã đọc qua 50 dòng — không đọc toàn bộ tab.

Tính:

```
avg_5d = trung bình cheapest_price của các dòng tìm được (bỏ giá 0)
yesterday_price = cheapest_price của dòng gần nhất (bỏ giá 0)
change_pct = (today_price - yesterday_price) / yesterday_price * 100
```

Phân loại trend:

| Trend | Điều kiện | Hiển thị |
| ----- | --------- | -------- |
| Giảm mạnh | `change_pct <= -15%` | `- {change_pct}% (giảm mạnh)` |
| Giảm | `-15% < change_pct <= -5%` | `- {change_pct}% (giảm)` |
| Ổn định | `-5% < change_pct < +5%` | `→` |
| Tăng | `+5% <= change_pct < +15%` | `+ {change_pct}% (tăng)` |
| Tăng mạnh | `change_pct >= +15%` | `+ {change_pct}% (tăng mạnh)` |
| Lần đầu | Không có dữ liệu trước | `mới` |

### 5.3 Phát hiện deal

Một kết quả là **deal** nếu thoả **ít nhất 1** điều kiện:

| Loại deal | Điều kiện | Ghi chú |
| --------- | --------- | ------- |
| Dưới budget | `today_price <= max_budget` | Dưới budget |
| Giảm mạnh | `change_pct <= -15%` so với `avg_5d` | Giảm mạnh |
| Giá thấp lịch sử | `today_price` thấp hơn **mọi** giá trong 5 dòng gần nhất cùng route | Thấp nhất 5 ngày |

**Chống alert trùng:** deal chỉ gửi nếu **không** có dòng nào trong tab `deals`
với cùng `route` và `price` sai lệch <= 50.000đ và `sent_at` trong 3 ngày gần nhất.

Ví dụ: route `HAN-BKK`, giá hôm nay 2.450.000đ, tab `deals` có dòng
`HAN-BKK | 2420000 | skyscanner | 2026-09-19T08:00:00+07:00` → sai lệch 30.000đ (dưới ngưỡng 50.000đ) và mới gửi hôm qua → **không gửi lại**.

## Bước 6 — Ghi Sheet (LÀM TRƯỚC KHI GỬI EMAIL)

Thứ tự bắt buộc:

1. **Append vào `price_log`** — mỗi route một dòng với giá tốt nhất hôm nay:
   `route | checked_at | source | cheapest_price | airline | depart_date | return_date | url`

   Route không có dữ liệu (cả 2 nguồn lỗi) → vẫn ghi dòng, `cheapest_price = 0`,
   `source = "lỗi"`. Để user nhìn vào Sheet biết ngày đó không có data, không phải
   là task không chạy.

2. **Append deal mới vào `deals`** — chỉ deal chưa có (sau khi check trùng ở 5.3):
   `route | price | source | sent_at | url`

Ghi trước, gửi sau. Email lỗi thì lần sau không gửi deal trùng.

## Bước 7 — Gửi email digest

Gửi tới email của user. Dùng HTML, không dùng markdown thô.

**Tiêu đề:**

- Có deal: `[Flight Deals] {dd/MM} — {N} route, {M} deal mới`
- Không deal: `[Flight Deals] {dd/MM} — {N} route theo dõi`
- Chỉ promo: `[Flight Deals] Khuyến mãi {dd/MM}`

**Thân email:**

```html
<p>{Mở đầu 1–2 câu: tổng route, route nào có deal, route nào giảm/tăng mạnh nhất.}</p>

<h3>Bảng giá hôm nay ({n} route)</h3>
<table border="1" cellpadding="6" cellspacing="0" style="border-collapse:collapse">
  <tr><th>Route</th><th>Giá rẻ nhất</th><th>Hãng</th><th>Ngày bay</th><th>Budget</th><th>Trend</th><th>Link</th></tr>
  <tr><td>Hà Nội → Bangkok</td><td>2.450.000đ</td><td>VietJet</td><td>05/12</td><td>3.000.000đ</td><td>-12%</td><td><a href="{url}">Xem</a></td></tr>
  <tr><td>TP.HCM → Seoul</td><td>7.800.000đ</td><td>VN Airlines</td><td>15/01</td><td>8.000.000đ</td><td>→</td><td><a href="{url}">Xem</a></td></tr>
</table>

<h3>Deal nổi bật ({m})</h3>
<table border="1" cellpadding="6" cellspacing="0" style="border-collapse:collapse">
  <tr><th>Route</th><th>Giá</th><th>Hãng</th><th>Vì sao là deal</th><th>Link</th></tr>
  <tr><td>Hà Nội → Đà Nẵng</td><td>890.000đ</td><td>VietJet</td><td>Giảm 25% — Thấp nhất 5 ngày</td><td><a href="{url}">Xem</a></td></tr>
</table>

<h3>Khuyến mãi hãng bay</h3>
<ul>
  <li><b>VietJet:</b> Bay khắp Việt Nam từ 0đ — đến 30/09 — <a href="{url}">Xem</a></li>
  <li><b>Vietnam Airlines:</b> Ưu đãi mùa đông Tokyo, Seoul — đến 15/10 — <a href="{url}">Xem</a></li>
  <li><b>Bamboo Airways:</b> Không có khuyến mãi mới</li>
</ul>

<hr>
<p><b>Báo cáo lần chạy</b></p>
<ul>
  <li>Route quét: {ROUTE_COUNT}</li>
  <li>Nguồn không truy cập được: {Skyscanner (captcha), Traveloka (timeout) | không có}</li>
  <li>Route không có dữ liệu: {HAN-NRT (cả 2 nguồn lỗi) | không có}</li>
  <li>Deal mới: {M} ({đã gửi alert} + {trùng, bỏ qua})</li>
  <li>Skill: v1.0</li>
</ul>
<p><a href="{url}">Flight Deal Tracker</a> — tab <code>price_log</code> có lịch sử giá, tab <code>routes</code> để thêm/sửa route.</p>
```

**Quy tắc dựng:**

- Dùng đúng các thẻ và thứ tự trong khung. **Không** thêm CSS, màu nền, font, ảnh,
  `<div>`/`<span>` — Gmail bỏ phần lớn CSS
- **Giá:** format `X.XXX.XXXđ` có dấu chấm phân cách ngàn. Ví dụ `2.450.000đ`
- **Route:** dùng `route_name` từ Sheet (ví dụ "Hà Nội → Bangkok"), không dùng code
- **Không dùng icon/emoji** trong tiêu đề, bảng giá hay email để giữ giao diện sạch sẽ, chuyên nghiệp
- **Section không có dữ liệu → bỏ hẳn** cả `<h3>` lẫn bảng:
  - "Bảng giá hôm nay": bỏ nếu `mode = promo`
  - "Deal nổi bật": bỏ nếu không có deal mới
  - "Khuyến mãi hãng bay": **luôn hiện** (ghi "Không có khuyến mãi mới" nếu trống)
- **Trong bảng giá:** sắp xếp: route có deal lên trước (thấp nhất 5 ngày → giảm mạnh → dưới budget),
  sau đó route giảm giá, cuối cùng route tăng giá hoặc ổn định
- **Khối báo cáo:** luôn đủ 5 dòng, đúng thứ tự. Dòng nào trống ghi `không có`
- **Link sheet:** trỏ tới đúng file `Flight Deal Tracker`

**Nếu không có route active nào:** tiêu đề `[Flight Deals] {dd/MM} — chưa có route`.
Thân email: một dòng `<p>Chưa có route nào trong Sheet. Thêm route vào tab routes để bắt đầu theo dõi.</p>`, rồi `<hr>`, khối báo cáo, link sheet.

## Chế độ `cleanup` — Dọn dẹp

Khi `mode = cleanup`, **bỏ qua toàn bộ Bước 1–7**. Chỉ làm:

1. Đếm `LOG_BEFORE` (dòng trong `price_log`) và `DEALS_BEFORE` (dòng trong `deals`)
2. **Guard 1:** nếu `LOG_BEFORE < 30` → SKIPPED, lý do "chưa đủ dữ liệu để dọn"
3. Tính `CUTOFF = now - 90 ngày`
4. Đếm số dòng cần xoá trong `price_log` (có `checked_at < CUTOFF`) → `OLD_COUNT`
5. **Guard 2:** nếu `OLD_COUNT > 80%` * `LOG_BEFORE` → BLOCKED, lý do
   "{OLD_COUNT}/{LOG_BEFORE} dòng vượt ngưỡng 80%, bất thường, cần user kiểm tra"
6. Xoá dòng cũ trong `price_log`
7. Xoá dòng trong `deals` có `sent_at < CUTOFF`
8. Đếm lại `LOG_AFTER`, `DEALS_AFTER`

**Tiêu đề email:** `[Flight Deals] Dọn dẹp — {DONE|SKIPPED|BLOCKED} — {dd/MM}`

**Thân email:** HTML, ngắn, một `<ul>` với:

- Kết quả: `DONE` / `SKIPPED` / `BLOCKED` + lý do
- `price_log`: `{LOG_BEFORE}` → `{LOG_AFTER}` dòng
- `deals`: `{DEALS_BEFORE}` → `{DEALS_AFTER}` dòng
- Link sheet

**Luôn gửi email**, kể cả SKIPPED — để user biết task còn sống.

## Ràng buộc chung

- Không bịa giá. Không tìm thấy giá thì ghi `không rõ`, không suy đoán.
- Không tái sử dụng kết quả lần chạy trước. Mỗi lần quét lại thật.
- Output tiếng Việt, giữ nguyên tên hãng bay và tên sân bay tiếng Anh.
- Múi giờ tham chiếu: Asia/Ho_Chi_Minh (GMT+7).
