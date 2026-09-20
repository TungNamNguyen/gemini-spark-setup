# Gemini Spark — Flight Deal Tracker

**Phiên bản 1.0** — theo dõi giá vé máy bay cho các route cấu hình sẵn trong Google Sheet, quét Skyscanner + Traveloka + trang khuyến mãi hãng bay VN, so sánh giá theo ngày, alert khi có deal rẻ. Không cần Gmail label.

Thời gian setup: **15 phút**.

- [Tổng quan](#tổng-quan)
- [Setup](#setup)
- [Nội dung Task](#nội-dung-task)
- [Email nhận được trông thế nào](#email-nhận-được-trông-thế-nào)
- [Xử lý sự cố](#xử-lý-sự-cố)

---

# Tổng quan

| Thành phần   | Số lượng | Ghi chú                                          |
| ------------ | -------- | ------------------------------------------------- |
| Spark Skill  | 1        | `air-deal-radar`                                  |
| Spark Task   | 2        | Quét giá hàng ngày + Dọn dẹp hàng tháng          |
| Schedule     | 2        | 14:00 hàng ngày + 06:00 mùng 1 hàng tháng        |
| Google Sheet | 1        | `Flight Deal Tracker` (3 tab)                     |
| Gmail label  | 0        | Không cần                                         |

Luồng:

```
Google Sheet (routes)
        │
        ▼
Skyscanner ─────┐
Traveloka ──────┤
Promo pages ────┘
        │
        ▼
So sánh giá ──► ghi price_log ──► email digest
        │
        ▼ (nếu deal)
ghi deals tab ──► alert trong email
```

---

# Setup

## Bước 1 — Kiểm tra Spark (2 phút)

| Điều kiện                            | Cách kiểm tra                                          |
| ------------------------------------ | ------------------------------------------------------ |
| Gói Google AI **Pro** hoặc **Ultra** | gemini.google.com → avatar → xem gói                   |
| Tài khoản Google **cá nhân**         | Không dùng tài khoản công ty/trường học                |
| Bật **Keep Activity**                | myactivity.google.com/product/gemini                   |

gemini.google.com → Menu → tìm mục **Spark**. Thấy → sang Bước 2.

## Bước 2 — Bật Connected Apps (1 phút)

gemini.google.com → **Settings & help → Connected Apps**

- ☑ **Google Workspace** — cần Sheets (lưu giá) + Gmail (gửi email)
- ☑ **Google Search**

## Bước 3 — Tạo Google Sheet (5 phút)

### 3.1 Tạo file

sheets.google.com → tạo sheet trống → đặt tên:

```
Flight Deal Tracker
```

### 3.2 Tab 1: `routes`

Đây là tab bạn tự điền — skill chỉ đọc.

1. Đổi tên tab mặc định "Sheet1" thành `routes`
2. Dòng 1 điền header:

| A      | B           | C          | D            | E            | F      | G          | H      |
| ------ | ----------- | ---------- | ------------ | ------------ | ------ | ---------- | ------ |
| origin | destination | route_name | travel_month | return_month | adults | max_budget | active |

3. Thêm route muốn theo dõi. Ví dụ:

| origin | destination | route_name         | travel_month | return_month | adults | max_budget | active |
| ------ | ----------- | ------------------ | ------------ | ------------ | ------ | ---------- | ------ |
| HAN    | BKK         | Hà Nội → Bangkok  | 2026-12      |              | 1      | 3000000    | TRUE   |
| SGN    | ICN         | TP.HCM → Seoul    | 2027-01      | 2027-01      | 2      | 8000000    | TRUE   |
| HAN    | SGN         | Hà Nội → TP.HCM   | 2026-11      |              | 1      | 1500000    | TRUE   |
| HAN    | DAD         | Hà Nội → Đà Nẵng  | 2026-12      |              | 2      | 2000000    | TRUE   |
| SGN    | PQC         | TP.HCM → Phú Quốc | 2026-12      |              | 1      | 1200000    | TRUE   |

> **Ghi chú:**
> - `origin`, `destination`: Mã IATA 3 chữ viết hoa (xem bảng trong SKILL.md). Dùng `BKKT` để tìm cả 2 sân bay Bangkok.
> - `travel_month`: tháng muốn đi, định dạng `yyyy-MM`.
> - `return_month`: tháng về. **Để trống nếu bay 1 chiều** (skill sẽ tìm vé one-way).
> - `max_budget`: ngân sách tối đa tính bằng VND. Skill alert khi giá dưới mức này.
> - `active`: đổi thành `FALSE` để tạm dừng theo dõi route mà không cần xoá.

### 3.3 Tab 2: `price_log`

1. Tạo tab mới, đặt tên `price_log`
2. Dòng 1 điền header:

| A     | B          | C      | D              | E       | F           | G           | H   |
| ----- | ---------- | ------ | -------------- | ------- | ----------- | ----------- | --- |
| route | checked_at | source | cheapest_price | airline | depart_date | return_date | url |

3. Xoá cột I trở đi, xoá dòng 501 trở xuống

### 3.4 Tab 3: `deals`

1. Tạo tab mới, đặt tên `deals`
2. Dòng 1 điền header:

| A     | B     | C      | D       | E   |
| ----- | ----- | ------ | ------- | --- |
| route | price | source | sent_at | url |

3. Xoá cột F trở đi

### 3.5 Kiểm tra

File `Flight Deal Tracker` phải có đúng 3 tab: `routes`, `price_log`, `deals`. Không có tab "Sheet1" thừa. Tab `routes` có ít nhất 1 route với `active = TRUE`.

## Bước 4 — Tạo Skill (3 phút)

1. gemini.google.com → **Menu → Spark → Skills → Create skill**
2. Mở file `SKILL.md` cùng thư mục, copy **toàn bộ** (cả khối `---`), dán vào, Lưu

Kiểm tra: thấy skill `air-deal-radar` trong danh sách.

## Bước 5 — Tạo Task & Test (5 phút)

1. **Menu → Spark** → ô nhập
2. Copy instruction **Task 1** ở mục [Nội dung Task](#nội-dung-task)
3. **Thay `[ĐIỀN EMAIL CỦA BẠN]` bằng email thật**
4. Gõ `/` rồi chọn `air-deal-radar`
5. Submit — chạy ngay, chờ ~10 phút

**Lần chạy đầu Spark sẽ hỏi:**

| Nó hỏi gì                              | Bạn làm gì                              |
| --------------------------------------- | ---------------------------------------- |
| Xin quyền kết nối Chrome local          | **Từ chối.** Dùng remote browser         |
| Xác nhận danh sách website sẽ truy cập  | Xem qua rồi **duyệt**                   |
| Xin quyền đọc Sheets                    | **Cho phép**                             |
| Xác nhận trước khi gửi email            | **Duyệt**                               |

**Sau khi xong, kiểm tra:**

1. ☑ Email đã về, tiêu đề `[Flight Deals]...`
2. ☑ Bảng giá có các route đã cấu hình trong Sheet
3. ☑ Tab `price_log` có dòng mới, cột `cheapest_price` là số hợp lý
4. ☑ Cuối email có khối báo cáo, dòng `Skill: v1.0`
5. ☑ Link "Flight Deal Tracker" cuối email mở đúng file Sheet
6. ☑ Xem "Nguồn không truy cập được" — Skyscanner/Traveloka có bị captcha không?
   Nếu cả 2 đều bị chặn ở lần chạy đầu, đợi vài giờ rồi thử lại (IP remote browser
   có thể đổi)

## Bước 6 — Đặt lịch (2 phút)

Mở thread Task 1, nhắn:

```
Tạo lịch: mỗi ngày lúc 14:00 giờ Việt Nam, chạy với mode: full.
```

> **Tại sao 14:00?** Để so le hoàn hảo với các skill khác trong repo (Job Radar HN 08:00, HCM 09:30, Tech News 11:00, Job Radar chiều 17:30). Đồng thời 14:00 là thời điểm vừa kết thúc đợt flash sale trưa (12:00–14:00) của các hãng bay (như VietJet), giá vé trong ngày đã cập nhật ổn định nhất để quét và chốt giá.

Tạo Task 2 (Dọn dẹp) theo instruction ở mục [Nội dung Task](#nội-dung-task), rồi nhắn:

```
Tạo lịch: mùng 1 hàng tháng lúc 06:00 giờ Việt Nam, chạy với mode: cleanup.
```

Setup xong! 🎉

---

# Nội dung Task

## Task 1 — Quét giá hàng ngày

```
Quét giá vé máy bay và gửi email digest.

Tham số:
- mode: full
- Email gửi về: [ĐIỀN EMAIL CỦA BẠN]

Làm đúng theo skill, đặc biệt:
- Bước 1 đọc tab "routes" của Google Sheet "Flight Deal Tracker" TRƯỚC mọi việc khác
- Bước 6 ghi Sheet TRƯỚC khi gửi email
- Dùng remote browser, không cần Chrome local
```

## Task 2 — Dọn dẹp hàng tháng

```
Dùng skill /air-deal-radar.

Tham số:
- mode: cleanup
- Email gửi về: [ĐIỀN EMAIL CỦA BẠN]

Không quét gì. Chỉ dọn price_log và deals cũ hơn 90 ngày trong
Google Sheet "Flight Deal Tracker". Nếu chưa đủ 30 dòng thì SKIPPED.
```

## Bảng lịch

| Task         | Lịch                       |
| ------------ | -------------------------- |
| Quét giá     | 14:00 hàng ngày            |
| Dọn dẹp      | 06:00 mùng 1 hàng tháng   |

---

# Email nhận được trông thế nào

## Email quét giá (hàng ngày)

**Tiêu đề:** `[Flight Deals] 20/09 — 5 route, 2 deal mới`

> Hôm nay có 5 route đang theo dõi. Route Hà Nội → Bangkok giảm 12% xuống 2.45 triệu, dưới budget.
>
> **Bảng giá hôm nay (5 route)**
>
> | Route | Giá rẻ nhất | Hãng | Ngày bay | Budget | Trend | Link |
> |---|---|---|---|---|---|---|
> | Hà Nội → Bangkok | 2.450.000đ | VietJet | 05/12 | 3.000.000đ | ↓ -12% | Xem |
> | TP.HCM → Seoul | 7.800.000đ | VN Airlines | 15/01 | 8.000.000đ | → | Xem |
> | Hà Nội → TP.HCM | 890.000đ | VietJet | 20/11 | 1.500.000đ | ↓ -25% (mạnh) | Xem |
> | Hà Nội → Đà Nẵng | 1.200.000đ | Bamboo | 10/12 | 2.000.000đ | → | Xem |
> | TP.HCM → Phú Quốc | 980.000đ | VietJet | 18/12 | 1.200.000đ | mới | Xem |
>
> **Deal nổi bật (2)**
>
> | Route | Giá | Hãng | Vì sao là deal | Link |
> |---|---|---|---|---|
> | Hà Nội → Bangkok | 2.450.000đ | VietJet | Dưới budget 3tr — ↓ -12% | Xem |
> | Hà Nội → TP.HCM | 890.000đ | VietJet | Giảm 25% — Thấp nhất 5 ngày | Xem |
>
> **Khuyến mãi hãng bay**
> - **VietJet:** Bay khắp Việt Nam từ 0đ — đến 30/09 — Xem
> - **Vietnam Airlines:** Không có khuyến mãi mới
> - **Bamboo Airways:** Côn Đảo mùa đông từ 499k — đến 15/10 — Xem
>
> ---
> **Báo cáo lần chạy**
> - Route quét: 5
> - Nguồn không truy cập được: không có
> - Route không có dữ liệu: không có
> - Deal mới: 2
> - Skill: v1.0
>
> [Flight Deal Tracker](#) — tab `price_log` có lịch sử giá, tab `routes` để thêm/sửa route.

## Email dọn dẹp (hàng tháng)

**Tiêu đề:** `[Flight Deals] Dọn dẹp — 01/10`

> - Kết quả: DONE
> - price_log: 150 → 120 dòng
> - deals: 45 → 32 dòng
>
> [Flight Deal Tracker](#)

---

# Xử lý sự cố

| Vấn đề | Giải pháp |
| ------ | --------- |
| Email không đến | Mở Spark → Tasks, xem có task đang chờ confirm không |
| Bảng giá toàn `không rõ` | Cả 2 nguồn bị chặn. Đợi vài giờ rồi chạy lại (IP remote browser đổi) |
| Skyscanner luôn bị captcha | Bình thường với remote browser — Traveloka vẫn hoạt động. Nếu cả 2 đều bị, xem bên dưới |
| Cả 2 nguồn đều bị chặn liên tục | Nhắn: `Chạy với mode: promo, chỉ quét khuyến mãi hãng bay` trong khi chờ IP đổi |
| Route mới thêm không được quét | Kiểm tra `active = TRUE`, `travel_month` chưa qua, IATA code đúng 3 chữ |
| Giá trong email khác giá trên web | Bình thường — giá vé thay đổi liên tục, email chỉ ghi snapshot lúc quét |
| Muốn thêm route | Mở Sheet → tab `routes` → thêm dòng mới. Lần chạy sau tự nhận |
| Muốn tạm dừng route | Đổi `active` thành `FALSE`. Muốn theo dõi lại → đổi về `TRUE` |
| Muốn thêm hãng bay | Sửa Bước 4 trong `SKILL.md`, thêm URL promo, dán lại Spark |
| Deal trùng gửi lại | Xoá deal cũ trong tab `deals` nếu muốn nhận lại alert |
| Muốn đổi ngưỡng deal | Sửa `max_budget` trong tab `routes` |
| Muốn quét 2 lần/ngày | Thêm lịch 17:00, mode: `promo` (nhẹ, chỉ quét khuyến mãi chiều) |
