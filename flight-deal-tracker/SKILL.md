---
name: flight-deal-tracker
description: Theo dõi giá vé máy bay cho các route do user cấu hình trong Google Sheet, dùng 100% remote browser quét Google Flights và trang khuyến mãi của VietJet, Vietnam Airlines, Bamboo Airways, tính trend 5 ngày, phát hiện deal rẻ, khử trùng lặp và gửi email digest HTML.
---
# Flight Deal Tracker

**Phiên bản skill: 1.0.** Luôn ghi số này vào dòng `Skill:` trong khối báo cáo cuối email, để user biết task đang chạy đúng bản skill mới nhất.

## Mục tiêu

Theo dõi giá vé máy bay cho các route user đã cấu hình trong Google Sheet `Flight Deal Tracker`, phát hiện deal rẻ (dưới ngân sách, giảm từ 15% trở lên, hoặc thấp nhất 5 ngày), quét khuyến mãi từ 3 hãng bay nội địa, loại bỏ alert trùng lặp, rồi gửi một email digest tổng hợp.

## Tham số đầu vào

| Tham số | Giá trị hợp lệ | Mặc định | Ghi chú |
| ------- | --------------- | -------- | ------- |
| `mode` | `full` / `promo` / `cleanup` | `full` | `full` = quét bảng giá Google Flights + khuyến mãi hãng bay. `promo` = chỉ quét trang khuyến mãi 3 hãng bay (dùng khi chỉ muốn săn flash sale). `cleanup` = dọn dẹp dữ liệu cũ hơn 90 ngày trong sheet, không quét gì |

Không hỏi lại user khi thiếu tham số — skill chạy tự động theo lịch, không có ai trả lời.

## Quy tắc tự chủ — KHÔNG BAO GIỜ dừng chờ user

Skill này chạy theo lịch lúc user offline. Mọi lần dừng để hỏi đều đồng nghĩa với một lần chạy bị mất. Vì vậy, trong suốt task:

- **Không hỏi bất kỳ câu nào** — không hỏi xác nhận, không hỏi "có tiếp tục không", không hỏi tham số. Thiếu gì thì dùng mặc định và ghi vào khối báo cáo cuối email.
- **Không yêu cầu take control** / không đề nghị user đăng nhập hộ. Gặp trang bắt đăng nhập, captcha, Cloudflare chặn -> ghi nguồn đó là "không truy cập được" và đi tiếp ngay sang nguồn khác.
- **Không tự đoán hay tự tìm domain mới.** Chỉ mở đúng những domain quy định: `google.com` (Google Flights, Google Sheets, Gmail), `vietjetair.com`, `vietnamairlines.com`, `bambooairways.com`. Danh sách cố định.
- **100% remote browser** — không dùng Chrome local, không dùng search hỏi đáp chung chung. Mọi thông tin giá vé và khuyến mãi đều phải do remote browser mở trang web thực tế và trích xuất trực tiếp.
- Gặp lỗi ở một nguồn -> ghi nhận, đi tiếp nguồn sau. Gặp lỗi ở bước ghi sheet -> vẫn gửi email và nói rõ bước nào lỗi. Chỉ dừng hẳn khi không đọc được tab `routes` trong Google Sheet.

## Định dạng dữ liệu chuẩn

| Trường | Định dạng | Ví dụ |
| ------ | --------- | ----- |
| `checked_at`, `sent_at` | ISO 8601 có múi giờ | `2026-09-20T14:00:00+07:00` |
| `travel_month`, `return_month` | `yyyy-MM` | `2026-11` |
| `depart_date`, `return_date` | `yyyy-MM-dd`; không rõ thì ghi `không rõ` | `2026-11-15` |
| `cheapest_price`, `max_budget` | Số nguyên thuần trong Sheet (VND); trong email hiển thị có dấu chấm và đuôi đ | Sheet: `1850000` / Email: `1.850.000đ` |
| `source` | Tên nguồn dữ liệu | `google_flights`, `vietjet`, `vietnam_airlines`, `bamboo` |

Mọi phép tính ngày và giờ đều dùng múi giờ Asia/Ho_Chi_Minh (GMT+7).

## Cấu trúc bộ nhớ: Google Sheet `Flight Deal Tracker`

File spreadsheet `Flight Deal Tracker` nằm trên Google Drive của user có 3 tab:

### 1. Tab `routes` (CHỈ ĐỌC)
User nhập các chặng bay muốn theo dõi. Các cột:
- Cột A (`origin`): Mã sân bay đi (IATA 3 chữ cái viết hoa, ví dụ: HAN, SGN, DAD)
- Cột B (`destination`): Mã sân bay đến (IATA 3 chữ cái viết hoa, ví dụ: BKK, SIN, ICN, NRT, PQC)
- Cột C (`route_name`): Tên hiển thị dễ đọc (ví dụ: Hà Nội -> Bangkok)
- Cột D (`travel_month`): Tháng bay (định dạng `yyyy-MM`)
- Cột E (`return_month`): Tháng về (định dạng `yyyy-MM`; để trống nếu là vé một chiều)
- Cột F (`adults`): Số lượng hành khách người lớn (số nguyên từ 1 đến 9, mặc định 1)
- Cột G (`max_budget`): Mức ngân sách tối đa mong muốn (số nguyên VND, ví dụ: 2500000)
- Cột H (`active`): Trạng thái theo dõi (`TRUE` = đang theo dõi, `FALSE` = tạm dừng)

### 2. Tab `price_log` (GHI & ĐỌC 5 LẦN QUÉT GẦN NHẤT)
Lưu lịch sử từng lần quét giá để tính xu hướng giá 5 ngày. Các cột:
- Cột A (`route`): Ghép mã sân bay dạng `{origin}-{destination}` (ví dụ: HAN-BKK)
- Cột B (`checked_at`): Thời điểm quét giá (ISO 8601)
- Cột C (`source`): Nguồn lấy giá (`google_flights`)
- Cột D (`cheapest_price`): Giá rẻ nhất tìm thấy (số nguyên VND)
- Cột E (`airline`): Tên hãng bay có giá rẻ nhất (ví dụ: VietJet Air, Vietnam Airlines)
- Cột F (`depart_date`): Ngày bay khởi hành (`yyyy-MM-dd`)
- Cột G (`return_date`): Ngày về (`yyyy-MM-dd`, hoặc để trống nếu 1 chiều)
- Cột H (`url`): Đường dẫn xem chuyến bay

### 3. Tab `deals` (ĐỌC & GHI)
Lưu lịch sử các deal đã gửi alert để chống gửi trùng lặp. Các cột:
- Cột A (`route`): Mã chặng bay `{origin}-{destination}`
- Cột B (`price`): Giá vé tại thời điểm phát hiện deal
- Cột C (`source`): Nguồn phát hiện
- Cột D (`sent_at`): Thời điểm gửi alert (ISO 8601)
- Cột E (`url`): Link xem chuyến bay

---

## Bước 1 — Đọc cấu hình route từ Google Sheet (BẮT BUỘC)

1. Mở file `Flight Deal Tracker` bằng Connected App Google Sheets.
2. Đọc toàn bộ các dòng trong tab `routes` có cột `active` là `TRUE`.
3. Kiểm tra tính hợp lệ của từng route:
   - `origin` và `destination`: Đúng 3 chữ cái viết hoa mã IATA chuẩn.
   - `travel_month`: Đúng định dạng `yyyy-MM` và không nhỏ hơn tháng hiện tại.
   - Nếu `travel_month` nhỏ hơn tháng hiện tại: Đánh dấu là route hết hạn (`EXPIRED`), bỏ qua không quét giá, và ghi nhận vào khối báo cáo cuối email.
   - `max_budget`: Phải là số nguyên dương lớn hơn 0.
4. Đọc toàn bộ tab `deals` để lấy danh sách các deal đã gửi trong vòng 3 ngày qua (phục vụ khử trùng lặp ở Bước 4).
5. Ghi lại số route hợp lệ (`ROUTE_COUNT`). Nếu không có route nào hợp lệ hoặc tab trống, chuyển thẳng đến Bước 6 gửi email thông báo nhắc user thêm route.

---

## Bước 2 — Quét giá vé máy bay qua Remote Browser (100% Remote Browser)

**Nếu `mode = promo`: Bỏ qua bước này, chuyển sang Bước 3.**

Với từng route hợp lệ trong danh sách active:

1. **Điều hướng:**
   - Dùng Remote Browser mở dịch vụ Google Flights tại địa chỉ: `https://www.google.com/travel/flights`
   - Nhập điểm đi (`origin`), điểm đến (`destination`), chọn loại vé (một chiều nếu `return_month` trống, hoặc khứ hồi nếu có `return_month`), chọn số lượng hành khách (`adults`), và chọn tháng bay (`travel_month`).
   - Mở màn hình biểu đồ lịch giá theo ngày (Date grid / Price graph) hoặc danh sách chuyến bay tốt nhất.

2. **Trích xuất dữ liệu:**
   - Tìm mức giá vé rẻ nhất trong toàn bộ tháng chỉ định: Ghi nhận `cheapest_price` (quy đổi về số nguyên VND, ví dụ 1.850.000đ -> 1850000).
   - Xác định hãng bay cung cấp giá đó (`airline`): Ví dụ VietJet Air, Vietnam Airlines, Bamboo Airways, Vietravel Airlines.
   - Xác định ngày bay khởi hành (`depart_date`) và ngày về (`return_date` nếu có).
   - Lấy URL kết quả tìm kiếm của chặng bay.

3. **Xử lý giới hạn & lỗi:**
   - Thời gian chờ tối đa cho mỗi route là 60 giây.
   - Tối đa quét 10 route trong một lần chạy để đảm bảo hoàn thành task đúng hạn.
   - Nếu gặp lỗi mạng, trang không hiển thị chuyến bay, hoặc bị timeout: Ghi nhận giá là `không rõ`, hãng là `không rõ`, và chuyển sang route tiếp theo. Không dừng task.

---

## Bước 3 — Quét khuyến mãi hãng bay qua Remote Browser (100% Remote Browser)

Dùng Remote Browser mở lần lượt 3 trang khuyến mãi chính thức của các hãng hàng không nội địa:

1. **VietJet Air:**
   - URL: `https://www.vietjetair.com/vi/khuyen-mai`
   - Đọc danh sách các chương trình khuyến mãi, vé 0 đồng, flash sale 12h-14h đang áp dụng.

2. **Vietnam Airlines:**
   - URL: `https://www.vietnamairlines.com/vn/vi/offers`
   - Đọc các chương trình ưu đãi Thứ 5 rực rỡ, giá vé nội địa ưu đãi, giảm giá chặng bay quốc tế.

3. **Bamboo Airways:**
   - URL: `https://www.bambooairways.com/vn/vi/explore/offers`
   - Đọc các chương trình ưu đãi ngày vàng, combo vé bay và khách sạn, mã giảm giá hội viên.

**Quy tắc lọc khuyến mãi:**
- Chỉ lấy các chương trình đang còn hạn áp dụng (thời gian bay hoặc thời gian mua vé bao gồm thời điểm hiện tại hoặc tương lai).
- Bỏ qua các tin tức tuyển dụng, tin tức hợp tác ngân hàng, hoặc quảng cáo dịch vụ bảo hiểm.
- Với mỗi chương trình, lấy 4 thông tin: Tên chương trình, Mức giá từ (nếu có), Thời hạn áp dụng, và Link chi tiết.
- Lấy tối đa 2 chương trình nổi bật nhất của mỗi hãng (tổng tối đa 6 chương trình).

---

## Bước 4 — So sánh giá, tính Trend 5 ngày & Phát hiện Deal

Với mỗi route đã quét được giá hôm nay (`today_price`):

1. **Tính xu hướng giá (Trend 5 ngày):**
   - Đọc từ tab `price_log` tối đa 5 lần quét gần nhất của route này.
   - Nếu có từ 2 lần quét trở lên: Tính giá trị trung bình 5 ngày (`AVG_5D`).
   - Tính tỷ lệ phần trăm thay đổi: `pct_change = ((today_price - AVG_5D) / AVG_5D) * 100`.
   - Phân loại:
     - `Giảm mạnh`: Khi `pct_change <= -15%` (giảm từ 15% trở lên)
     - `Giảm`: Khi `-15% < pct_change <= -5%`
     - `Ổn định`: Khi `-5% < pct_change < 5%`
     - `Tăng`: Khi `pct_change >= 5%`
   - Nếu chưa có đủ 2 lần quét trong lịch sử: Ghi nhận xu hướng là `Mới`.

2. **Tiêu chí xác định Deal:**
   Một mức giá được coi là Deal khi thỏa mãn ít nhất một trong các điều kiện sau:
   - **Dưới budget:** `today_price <= max_budget`
   - **Giảm mạnh:** `pct_change <= -15%`
   - **Thấp nhất 5 ngày:** `today_price` thấp hơn toàn bộ các mức giá trong 5 lần quét gần nhất.

3. **Cơ chế chống spam alert (Khử trùng lặp 3 ngày):**
   - Tra cứu trong dữ liệu tab `deals` đã đọc ở Bước 1.
   - Nếu cùng route này đã được gửi alert trong vòng 3 ngày gần nhất (tính theo `sent_at`) VÀ mức giá chênh lệch không quá 50.000 VND so với giá đã gửi trước đó:
     - Đánh dấu deal này là "trùng lặp" (`DUPLICATE`).
     - Không đưa vào danh sách alert Deal nổi bật trong email.
     - Vẫn hiển thị bình thường trong "Bảng giá hôm nay".

---

## Bước 5 — Ghi dữ liệu vào Google Sheet (Làm TRƯỚC khi gửi email)

Thực hiện tuần tự để đảm bảo dữ liệu luôn được lưu trữ an toàn:

1. **Ghi tab `price_log`:**
   - Với mỗi route đã quét được trong ngày hôm nay, thêm 1 dòng mới vào cuối tab `price_log`:
     `route | checked_at | source | cheapest_price | airline | depart_date | return_date | url`
   - Nếu route bị lỗi không lấy được giá: Ghi giá trị `cheapest_price` là `không rõ`.

2. **Ghi tab `deals`:**
   - Với các deal mới thỏa mãn điều kiện và không bị đánh dấu trùng lặp:
     Thêm 1 dòng mới vào cuối tab `deals`:
     `route | price | source | sent_at | url`

---

## Bước 6 — Dựng và gửi email Digest tổng hợp

Gửi tới email cá nhân của user qua Gmail Connected App.

### Tiêu đề email
- Nếu có deal mới: `[Flight Deals] {dd/MM} — {ROUTE_COUNT} route, {DEAL_COUNT} deal mới`
- Nếu không có deal mới: `[Flight Deals] {dd/MM} — Bảng giá {ROUTE_COUNT} route hôm nay`
- Nếu tab `routes` trống: `[Flight Deals] {dd/MM} — Chưa có route nào được cấu hình`

### Nội dung email (HTML chuẩn, giao diện chuyên nghiệp)
Tuân thủ nghiêm ngặt các quy tắc định dạng:
- Dùng thẻ HTML cơ bản (`<table>`, `<tr>`, `<td>`, `<th>`, `<h3>`, `<p>`, `<ul>`, `<li>`, `<a>`). Không dùng CSS ngoài, không dùng class. Tất cả style dùng inline.
- Bảng có viền nét mảnh xám nhạt (`border: 1px solid #e0e0e0; border-collapse: collapse;`), padding `8px 12px`, text căn chỉnh rõ ràng.
- Giá tiền format chuẩn có dấu chấm: ví dụ `1.850.000đ`.
- Giữ phong cách chuyên nghiệp, sạch sẽ: Không sử dụng icon hay emoji trang trí trong tiêu đề, bảng giá hoặc nội dung email.

**Cấu trúc email gồm 4 phần:**

1. **Bảng giá hôm nay (chỉ hiện khi `mode = full`):**
   Bảng gồm các cột: Chặng bay (`route_name`), Giá rẻ nhất, Hãng bay, Ngày bay, Ngân sách, Xu hướng, Link xem.
   Sắp xếp thứ tự: Route có deal lên đầu tiên, sau đó đến route giảm giá, cuối cùng là route ổn định hoặc tăng giá.

2. **Danh sách Deal nổi bật:**
   Nếu có deal mới không bị trùng: Liệt kê chi tiết từng deal, nêu rõ lý do (ví dụ: "Dưới ngân sách 350.000đ", "Giảm 18% so với trung bình 5 ngày", hoặc "Giá thấp nhất trong 5 ngày qua").
   Nếu không có deal mới: Không hiển thị mục này.

3. **Khuyến mãi hãng bay (Luôn hiển thị):**
   Danh sách các chương trình ưu đãi hiện có từ VietJet, Vietnam Airlines, Bamboo Airways kèm thời hạn và link xem chi tiết. Nếu hãng không có ưu đãi mới, ghi "Chưa ghi nhận chương trình mới".

4. **Khối báo cáo lần chạy (BẮT BUỘC luôn ở cuối email):**
   Một danh sách `<ul>` gồm đúng 5 dòng:
   - Route quét: `{N}` route active
   - Nguồn không truy cập được: `{Danh sách nguồn lỗi hoặc "không có"}`
   - Route hết hạn: `{Số lượng route đã qua tháng bay hoặc "không có"}`
   - Deal mới phát hiện: `{Số deal mới} ({Số deal đã alert} gửi alert, {Số deal trùng} trùng bỏ qua)`
   - Skill: `v1.0`

Kèm dòng chữ có link mở Google Sheet:
`<p><a href="{URL_SHEET}">Flight Deal Tracker</a> — xem lịch sử giá tại tab price_log và cấu hình chặng bay tại tab routes.</p>`

---

## Chế độ `cleanup` — Dọn dẹp dữ liệu cũ hàng tháng

Khi `mode = cleanup` (chạy vào 06:00 sáng ngày mùng 1 hàng tháng), **bỏ qua toàn bộ Bước 2 đến Bước 6**. Chỉ thực hiện quy trình dọn dẹp an toàn:

1. Đếm số dòng hiện tại trong `price_log` (`LOG_BEFORE`) và trong `deals` (`DEALS_BEFORE`).
2. **Chốt an toàn 1 (Guard 1):** Nếu `LOG_BEFORE < 30` dòng -> Hủy dọn dẹp (`SKIPPED`), lý do: "Chưa đủ dữ liệu tối thiểu (30 dòng) để dọn dẹp".
3. Xác định mốc thời gian: `CUTOFF = ngày hiện tại trừ 90 ngày`.
4. Đếm số dòng trong `price_log` có `checked_at < CUTOFF` (`OLD_COUNT`).
5. **Chốt an toàn 2 (Guard 2):** Nếu `OLD_COUNT > 80% * LOG_BEFORE` -> Chặn dọn dẹp (`BLOCKED`), lý do: "Số dòng cũ vượt quá 80% tổng dữ liệu, bất thường, cần user kiểm tra thủ công".
6. Nếu vượt qua cả 2 chốt an toàn: Xóa các dòng có `checked_at < CUTOFF` trong tab `price_log`, và xóa các dòng có `sent_at < CUTOFF` trong tab `deals`.
7. Đếm lại số dòng sau khi dọn: `LOG_AFTER`, `DEALS_AFTER`.
8. Gửi email báo cáo kết quả dọn dẹp:
   - Tiêu đề: `[Flight Deals] Dọn dẹp dữ liệu — {DONE | SKIPPED | BLOCKED} — {dd/MM}`
   - Nội dung: Liệt kê trạng thái, số dòng trước và sau khi dọn của tab `price_log` và `deals`, kèm link Sheet.

---

## Ràng buộc thực thi quan trọng

- **Không bịa giá:** Tuyệt đối không tự suy đoán giá vé nếu trang web không hiển thị hoặc bị lỗi. Không tìm thấy thì ghi rõ `không rõ`.
- **Không dùng kết quả cũ:** Mỗi lần chạy đều phải mở trang và trích xuất dữ liệu mới hoàn toàn.
- **Ngôn ngữ:** Nội dung email và thông báo bằng tiếng Việt; giữ nguyên mã sân bay IATA và tên hãng hàng không.
- **Múi giờ tham chiếu:** Asia/Ho_Chi_Minh (GMT+7).
