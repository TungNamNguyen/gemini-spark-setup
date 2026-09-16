# Gemini Spark — Tech News Digest

**Phiên bản 1.0** — quét Hacker News (Algolia API), Reddit RSS (5 subs), Dev.to API (6 tags), Medium topic pages; phân loại 6 chủ đề, chấm điểm lọc bài hay, gửi email digest hàng ngày. Không cần Google Sheet, không cần Gmail label.

Thời gian setup: **10 phút**.

- [Tổng quan](#tổng-quan)
- [Setup](#setup)
- [Nội dung Task](#nội-dung-task)
- [Xử lý sự cố](#xử-lý-sự-cố)

---

# Tổng quan

| Thành phần   | Số lượng | Ghi chú                          |
| ------------ | -------- | -------------------------------- |
| Spark Skill  | 1        | `tech-news-digest`               |
| Spark Task   | 1        | Digest hàng ngày                 |
| Schedule     | 1        | 07:00 hàng ngày                  |
| Google Sheet | 0        | Không cần                        |
| Gmail label  | 0        | Không cần                        |

Luồng đơn giản:

```
HN Algolia API ─┐
Reddit RSS      ├─► phân loại / chấm điểm ─► email digest
Dev.to API      │
Medium pages   ─┘
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

- ☑ **Google Workspace** — cần Gmail để gửi email digest
- ☑ **Google Search**

## Bước 3 — Tạo Skill (3 phút)

1. gemini.google.com → **Menu → Spark → Skills → Create skill**
2. Mở file `SKILL.md` cùng thư mục, copy **toàn bộ** (cả khối `---`), dán vào, Lưu

Kiểm tra: thấy skill `tech-news-digest` trong danh sách.

## Bước 4 — Tạo Task & Test (3 phút)

1. **Menu → Spark** → ô nhập
2. Gõ: `Quét tin công nghệ hôm nay và gửi email digest tới [ĐIỀN EMAIL CỦA BẠN]`
3. Gõ `/` rồi chọn `tech-news-digest`
4. Submit — chạy ngay, chờ ~10 phút

Kiểm tra email: tìm `[Tech Digest]` trong Gmail.

## Bước 5 — Đặt lịch (1 phút)

Đặt schedule chạy **07:00 hàng ngày**.

Setup xong! 🎉

---

# Nội dung Task

```
Quét tin công nghệ hôm nay và gửi email digest.

Gửi email tới: [ĐIỀN EMAIL CỦA BẠN]
```

---

# Xử lý sự cố

| Vấn đề                    | Giải pháp                                                  |
| -------------------------- | ---------------------------------------------------------- |
| Email không đến            | Kiểm tra Spark activity log, kiểm tra skill đã gắn chưa   |
| Không có bài trong email   | Xem dòng "Sources failed" — nếu tất cả fail → remote browser có vấn đề |
| Medium luôn fail           | Bình thường (Cloudflare captcha), 3 nguồn kia đủ dùng     |
| Reddit luôn fail           | Rate-limit, chấp nhận được, HN + Dev.to bù                |
| Muốn thêm nguồn web       | Sửa `SKILL.md`, thêm URL vào bước tương ứng, dán lại Spark |
| Muốn đổi chủ đề ưu tiên   | Sửa mục 5.2 trong `SKILL.md`: đổi `DE`/`AI` thành chủ đề muốn |
