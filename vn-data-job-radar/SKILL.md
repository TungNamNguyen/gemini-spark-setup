---
name: vn-data-job-radar
description: Quét các trang tuyển dụng Việt Nam tìm tin tuyển dụng mới ngành dữ liệu (Data Analyst, Analytics Engineer, Data Engineer, Business Intelligence, Business Analyst), lọc theo thành phố và ngày đăng, khử trùng lặp bằng Google Sheet, rồi gửi email tổng hợp. Dùng khi cần theo dõi thị trường việc làm data tại Hà Nội hoặc TP.HCM.
---
# VN Data Job Radar

## Mục tiêu

Tìm các tin tuyển dụng ngành dữ liệu **đăng trong 72 giờ gần nhất**, tại thành phố
được chỉ định, loại bỏ những tin đã gửi cho user trước đó, rồi gửi một email tổng hợp.

## Tham số đầu vào

| Tham số | Giá trị hợp lệ         | Mặc định  | Ghi chú                                                                                                                             |
| -------- | -------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `city` | `Hà Nội` \| `TP.HCM` | `Hà Nội` | Nếu user không chỉ định, dùng mặc định và **ghi rõ trong mở đầu email** là đang dùng thành phố mặc định |
| `mode` | `full` \| `quick` \| `cleanup` | `full` | `full` = quét đủ Gmail + 5 nguồn web + career page. `quick` = chỉ Gmail + Xóm Jobs + LinkedIn, bỏ career page — dùng cho buổi chiều. `cleanup` = **không quét gì**, chỉ dọn dẹp sheet và gửi email báo cáo riêng — xem mục "Chế độ cleanup" |

Không hỏi lại user khi thiếu tham số — skill chạy tự động, không có ai trả lời.

Lọc trùng ở mode `full` và `quick` đều dựa vào `seen_urls`. **Không** lọc theo "tin đăng kể từ lần
chạy trước" — chỉ cần URL chưa có trong SEEN là gửi.

## Định dạng dữ liệu chuẩn

Áp dụng thống nhất cho mọi chỗ ghi vào sheet và mọi phép so sánh ngày:

| Trường          | Định dạng                                                                                                     | Ví dụ                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | ----------------------------- |
| `first_sent_at` | ISO 8601 có múi giờ                                                                                           | `2026-09-12T08:30:00+07:00` |
| `posted_date`   | `yyyy-MM-dd`; không rõ thì `không rõ`                                                                   | `2026-09-11`                |
| `city`          | Đúng một trong:`Hà Nội`, `TP.HCM`, `Remote`, `Hybrid`                                               | `Hà Nội`                  |
| `role_group`    | Đúng một trong:`DA`, `AE`, `DE`, `BI`, `BA`                                                         | `DE`                        |
| `source`        | Đúng một trong:`xomjobs`, `linkedin`, `topcv`, `itviec`, `vietnamworks`, `gmail`, `career_page` | `topcv`                     |

Mọi phép tính "72 giờ", "90 ngày" đều dùng múi giờ Asia/Ho_Chi_Minh (GMT+7).

## Cấu trúc bộ nhớ: Google Sheet `Job Radar Tracker`

Sheet có 3 tab với vai trò tách bạch. Tuân thủ đúng vai trò này là bắt buộc,
vì nó quyết định hiệu năng của mọi lần chạy.

| Tab             | Cột       | Vai trò                                                                                                                            |
| --------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `seen_urls`   | A: job_url | **CHỈ ĐỌC + append.** Tab duy nhất được đọc trong lần chạy thường                                                |
| `jobs_detail` | A–H       | **CHỈ GHI** ở mode `full`/`quick`. **Chỉ mode `cleanup`** được đọc tab này |
| `archive`     | A–H       | **CHỈ GHI.** Không bao giờ đọc, không bao giờ xoá                                                                     |

Header của `jobs_detail` và `archive`:
`job_url | title | company | city | role_group | posted_date | source | first_sent_at`

## Bước 1 — Đọc bộ nhớ chống trùng (LÀM ĐẦU TIÊN, KHÔNG BỎ QUA)

Mở `Job Radar Tracker` → tab `seen_urls` → đọc **chỉ cột A** vào danh sách SEEN.

**Không đọc tab `jobs_detail` hoặc `archive` ở bước này.** Đọc chúng làm chậm task
và không mang lại thông tin gì thêm.

Ghi lại số lượng URL đọc được (gọi là `SEEN_COUNT`) để báo cáo ở cuối task.

Nếu sheet hoặc tab không tồn tại, tạo mới theo đúng cấu trúc trên rồi đi tiếp
với SEEN rỗng.

## Bước 2 — Quét Gmail Job Alerts (quét TRƯỚC khi duyệt web)

Đọc email trong 72 giờ gần nhất từ 5 Gmail label sau:

| Gmail Label                | Nguồn tương ứng |
| -------------------------- | ------------------- |
| `ITViec Job Alerts`      | ITviec              |
| `LinkedIn Job Alerts`    | LinkedIn Jobs       |
| `VietnamWorks Job Alert` | VietnamWorks        |
| `TopCV`                  | TopCV               |
| `Xom Job Alerts`         | Xóm Jobs           |

Với mỗi email, trích xuất danh sách job URL + tên vị trí + công ty.
Chuẩn hoá URL (Bước 6) rồi so với SEEN — chỉ giữ URL chưa có.

**Tại sao quét Gmail trước:**

- Nhanh hơn browse web rất nhiều (chỉ đọc text)
- Không bị captcha, rate-limit, hay giới hạn guest
- Bắt được tin LinkedIn mà remote browser bỏ lỡ do không đăng nhập
- Bước 3 (quét web) sau đó chỉ cần bổ sung những gì email chưa có

**Lưu ý:** chỉ trích URL job từ email, không đánh dấu email là đã đọc hay xoá.
Nếu label không tồn tại hoặc không có email mới, bỏ qua và đi tiếp.

## Bước 3 — Quét web

Quét lần lượt 5 nguồn dưới đây. Nếu gặp lỗi ở nguồn nào thì ghi nhận và đi tiếp.

**Nếu `mode = quick`:** chỉ quét Xóm Jobs và LinkedIn, bỏ 3 nguồn còn lại và bỏ hẳn
mục "Công ty ưu tiên". Vẫn đánh dấu ⭐ nếu công ty nằm trong danh sách ưu tiên.

| Nguồn                          | Ghi chú                                                                                                                                                                                                                                                                                                                         |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Xóm Jobs (jobs.xomdata.com)    | Chuyên Data & AI VN. Lọc theo category, location, ngày đăng. Không cần đăng nhập.**Quét đầu tiên**                                                                                                                                                                                                           |
| LinkedIn Jobs                   | Dùng bộ lọc thời gian trong URL:`f_TPR=r259200` (= 72 giờ). Ví dụ: `linkedin.com/jobs/search/?keywords=Data%20Analyst&location=Hanoi%2C%20Vietnam&f_TPR=r259200`. **Best-effort:** guest gần như chắc chắn gặp tường đăng nhập sau 1–2 trang; nguồn chính cho LinkedIn là Gmail alert ở Bước 2 |
| TopCV (topcv.vn)                | Sắp xếp "Tin mới nhất", lọc địa điểm                                                                                                                                                                                                                                                                                    |
| ITviec (itviec.com)             | Tốt nhất cho Data Engineer / Analytics Engineer                                                                                                                                                                                                                                                                                |
| VietnamWorks (vietnamworks.com) | Sắp xếp "Ngày đăng mới nhất"                                                                                                                                                                                                                                                                                              |

**Về URL từ nguồn tổng hợp:** Xóm Jobs aggregate tin từ TopCV, LinkedIn, Vieclam24h.
Nếu tin trên Xóm Jobs có link gốc trỏ về nguồn (TopCV, LinkedIn…), **ưu tiên lưu URL
nguồn gốc** vào `seen_urls` thay vì URL `jobs.xomdata.com/…`. Điều này tránh cùng
một tin bị lưu 2 URL khác nhau và lọt qua bộ lọc trùng.

**Về cách duyệt web:** skill thường chạy tự động khi user không mở máy, nên
**mặc định dùng remote browser**. Một số nguồn (đặc biệt LinkedIn) sẽ giới hạn
kết quả cho guest — đây là bình thường, không cần xử lý.
Nếu gặp captcha hoặc tường đăng nhập, ghi nhận nguồn đó là "không truy cập được"
và đi tiếp — không dừng cả task vì một nguồn.

### Công ty ưu tiên

Sau khi quét 5 nguồn chính, quét thêm trang tuyển dụng (career page) của các
công ty dưới đây. Chỉ tìm vị trí liên quan đến data/analytics.

NAB Innovation Centre Vietnam, Crossian, VNG, Grab Vietnam, Shopee Vietnam,
MoMo, ZaloPay, VNPAY, Techcombank, VPBank, MB Bank, Vingroup, VinSmart Future,
One Mount, Be Group, Lazada Vietnam, GreenSM.

**Quy tắc:**

- Nếu tin đã thấy ở 5 nguồn chính → không ghi trùng, chỉ đánh dấu ⭐ trong email
- Nếu tin chỉ có trên career page → thêm mới, ghi `source = "career_page"`
- Nếu career page không hiển thị ngày đăng → ghi `posted_date = "không rõ"` và
  **vẫn giữ** (đây là ngoại lệ duy nhất của quy tắc "không có ngày đăng → loại" ở Bước 5)
- Nếu gặp lỗi (trang đổi cấu trúc, timeout…) → bỏ qua, ghi vào báo cáo cuối email

## Bước 4 — Từ khoá theo nhóm vị trí

Tìm cả tiếng Anh lẫn tiếng Việt. Mã `role_group` ghi trong ngoặc.

**Data Analyst (`DA`)**
`Data Analyst`, `Chuyên viên Phân tích Dữ liệu`, `Data Analytics Specialist`,
`Product Analyst`, `Marketing Analyst`, `Risk Analyst`, `Fraud Analyst`,
`Growth Analyst`, `Operations Analyst`, `Quantitative Analyst`

**Analytics Engineer (`AE`)**
`Analytics Engineer`, `Analytics Developer`, `dbt Engineer`, `Data Modeler`,
`Data Modelling Engineer`, `Semantic Layer Engineer`

**Data Engineer (`DE`)**
`Data Engineer`, `Kỹ sư Dữ liệu`, `Big Data Engineer`, `ETL Developer`,
`ETL Engineer`, `Data Platform Engineer`, `DataOps Engineer`,
`Data Infrastructure Engineer`, `Data Warehouse Engineer`

**Business Intelligence (`BI`)**
`Business Intelligence`, `BI Developer`, `BI Engineer`, `BI Analyst`,
`Power BI Developer`, `Tableau Developer`, `Looker Developer`,
`Chuyên viên BI`, `Chuyên viên Báo cáo Quản trị`, `MIS Analyst`

**Business Analyst (`BA`)**
`Business Analyst`, `IT Business Analyst`, `Chuyên viên Phân tích Nghiệp vụ`,
`Systems Analyst`, `Chuyên viên Phân tích Kinh doanh`, `Process Analyst`

## Bước 5 — Quy tắc lọc

**GIỮ nếu:**

- Ngày đăng nằm trong 72 giờ tính đến thời điểm chạy
- Địa điểm khớp `city` được chỉ định (chấp nhận "Hybrid" và "Remote — Vietnam")
- URL đã chuẩn hoá KHÔNG nằm trong SEEN

**LOẠI nếu:**

- Không hiển thị ngày đăng ở bất kỳ đâu → loại.
  **Ngoại lệ duy nhất:** tin từ career page của công ty ưu tiên (Bước 3) được giữ với `posted_date = "không rõ"`.
- Tên có chữ "Analyst" nhưng thực chất không liên quan dữ liệu: Financial Analyst
  thuần kế toán, Credit Analyst thẩm định hồ sơ, Investment Analyst.
  Nếu JD không nhắc SQL / Python / Excel nâng cao / BI tool / data warehouse thì loại.
- Thực tập không lương

**Gộp tin trùng nội dung trong cùng lần chạy:**

Cùng một JD có thể xuất hiện trên nhiều nguồn (TopCV, ITviec, LinkedIn) hoặc được
nhiều headhunter đăng lại → URL khác nhau nhưng là một tin. Trước khi ghi sheet và
soạn email, gộp theo khoá phụ:

`lowercase(company) + "|" + lowercase(title đã bỏ ký tự đặc biệt và khoảng trắng thừa)`

- Nếu nhiều tin cùng khoá phụ → giữ **1 bản**, ưu tiên theo thứ tự nguồn:
  `career_page` > `itviec` > `topcv` > `vietnamworks` > `linkedin` > `xomjobs` > `gmail`
- Với headhunter: nếu company là tên công ty headhunt (Navigos, Robert Walters, Adecco,
  ManpowerGroup, HR2B, Talentnet…) và JD giống hệt tin của công ty thật → giữ tin của công ty thật
- **Vẫn append tất cả URL** của các bản trùng vào `seen_urls` (để lần sau không hiện lại),
  nhưng chỉ ghi **1 dòng** vào `jobs_detail` và chỉ hiện 1 dòng trong email

**Riêng Business Analyst:** ở Việt Nam phần lớn BA là IT BA (viết tài liệu,
gom requirement), không đụng dữ liệu. Chỉ giữ tin có nhắc SQL, dashboard, data,
reporting, hoặc analytics trong JD. Trong email, đánh dấu rõ tin nào là
"BA thiên data" và tin nào là "IT BA thuần".

## Bước 6 — Chuẩn hoá URL

Áp dụng **trước khi so với SEEN** và **trước khi ghi vào sheet**. Cùng một job trên
cùng một trang phải luôn cho ra cùng một chuỗi URL.

**Quy tắc chung (áp dụng cho mọi URL, theo thứ tự):**

1. Bỏ toàn bộ query string và fragment (`?...`, `#...`) — bao gồm `utm_*`, `ref`, `src`,
   `fbclid`, `gclid`, `trk`, `refId`, `trackingId`, ID phiên
2. Chuyển `http://` → `https://`
3. Bỏ `www.` ở đầu host
4. Lowercase toàn bộ host (không lowercase path)
5. Bỏ dấu `/` cuối cùng

**Riêng LinkedIn:** ID job là dãy số ở cuối path, có thể có hoặc không có slug tiêu đề
đứng trước, và subdomain có thể là `www.`, `vn.` hoặc không có. Chuẩn hoá về
`https://linkedin.com/jobs/view/{id}`. Ví dụ:
`vn.linkedin.com/jobs/view/data-analyst-at-vng-4123456789?trackingId=abc`
→ `https://linkedin.com/jobs/view/4123456789`

**Các nguồn khác:** chỉ áp dụng quy tắc chung, giữ nguyên path. Không tự cắt slug hay
đoán ID — URL sau chuẩn hoá vẫn phải là link click được, vì nó được dùng làm link
trong email. Trùng do slug đổi hoặc cùng tin trên nhiều trang đã được xử lý bằng
khoá phụ `company|title` ở Bước 5.

## Bước 7 — Ghi vào sheet (LÀM TRƯỚC KHI GỬI EMAIL)

Với mỗi job mới:

1. Append URL đã chuẩn hoá vào tab **`seen_urls`**, cột A
   (kể cả URL của các bản trùng đã gộp ở Bước 5)
2. Append một dòng đầy đủ 8 cột vào tab **`jobs_detail`**:
   `job_url | title | company | city | role_group | posted_date | source | first_sent_at`
   với `first_sent_at` = timestamp hiện tại theo ISO 8601 (xem mục Định dạng dữ liệu chuẩn)

Ghi trước, gửi sau. Nếu email lỗi thì lần chạy tới cũng không gửi trùng.

## Chế độ `cleanup` — Dọn dẹp hàng tuần (task riêng)

Khi `mode = cleanup`, **bỏ qua toàn bộ Bước 1–8**. Không đọc Gmail, không quét web,
không ghi `seen_urls`/`jobs_detail`. Chỉ làm đúng mục này rồi gửi một email báo cáo riêng.

Task này được schedule riêng (sáng thứ Hai, trước giờ chạy của các task quét) nên
không cần tự kiểm tra ngày. Nếu được gọi bất kỳ lúc nào khác, vẫn chạy bình thường.

Đây là chế độ duy nhất được phép **đọc** tab `jobs_detail`, và là thao tác **không hoàn
tác được** trên sheet. Vì vậy phải qua đủ các guard dưới đây.

### Các bước

1. Đếm số dòng hiện có: `DETAIL_BEFORE` (jobs_detail), `SEEN_BEFORE` (seen_urls)
2. Đọc `jobs_detail`, parse cột `first_sent_at` theo ISO 8601.
   - Dòng nào không parse được → **bỏ qua dòng đó**, không đụng vào, đếm vào `BAD_ROWS`
3. Tính `CUTOFF = now − 90 ngày`. Chọn các dòng có `first_sent_at < CUTOFF` → tập `OLD`
4. **Guard an toàn — kiểm tra trước khi xoá bất kỳ thứ gì:**
   - Nếu `DETAIL_BEFORE < 50` → kết quả `SKIPPED`, lý do "chưa đủ dữ liệu để dọn"
   - Nếu `OLD` rỗng → kết quả `SKIPPED`, lý do "không có dòng nào quá 90 ngày"
   - Nếu `|OLD| > 30%` × `DETAIL_BEFORE` → kết quả `BLOCKED`, không xoá gì,
     lý do "{|OLD|}/{DETAIL_BEFORE} dòng vượt ngưỡng 30%, cần user kiểm tra thủ công"
5. Thứ tự thao tác (phải đúng thứ tự này để lỗi giữa chừng không mất dữ liệu):
   1. Append toàn bộ `OLD` vào `archive` **trước**
   2. Xác nhận số dòng đã append vào `archive` bằng `|OLD|` — nếu không khớp → kết quả `FAILED`,
      không xoá gì, ghi rõ số dòng đã append
   3. Xoá các dòng `OLD` khỏi `jobs_detail`
   4. Xoá các URL tương ứng khỏi `seen_urls` (**chỉ xoá URL có trong `OLD`**, khớp chuỗi chính xác)
   5. Kết quả `DONE`
6. **KHÔNG bao giờ xoá dữ liệu khỏi `archive`**
7. Đếm lại `DETAIL_AFTER`, `SEEN_AFTER`, `ARCHIVE_TOTAL`

### Email báo cáo cleanup

**Tiêu đề:** `[Job Radar] Dọn dẹp tuần — {DONE|SKIPPED|BLOCKED|FAILED} — {dd/MM}`

**Thân email** (HTML, ngắn, không có bảng job):

- Kết quả: `DONE` / `SKIPPED` / `BLOCKED` / `FAILED` + lý do (nếu không phải `DONE`)
- Đã archive: `{|OLD|}` dòng (0 nếu không dọn)
- `jobs_detail`: `{DETAIL_BEFORE}` → `{DETAIL_AFTER}` dòng
- `seen_urls`: `{SEEN_BEFORE}` → `{SEEN_AFTER}` URL
- `archive`: tổng `{ARCHIVE_TOTAL}` dòng
- Dòng lỗi định dạng ngày bị bỏ qua: `{BAD_ROWS}` (chỉ ghi nếu > 0)
- Dòng cũ nhất còn lại trong `jobs_detail`: `{first_sent_at nhỏ nhất}`

**Luôn gửi email**, kể cả `SKIPPED` — để user biết task còn sống. Với `BLOCKED` và `FAILED`,
mở đầu email bằng một câu nói rõ cần user vào sheet kiểm tra.

Việc này giữ `seen_urls` ổn định ở mức vài nghìn dòng thay vì phình vô hạn,
nên mỗi lần đọc ở Bước 1 luôn nhanh.

## Bước 8 — Định dạng email

Gửi tới email của user. Dùng HTML, không dùng markdown thô.

**Tiêu đề:** `[Job Radar] {Thành phố} — {N} tin mới — {dd/MM HH:mm}`

**Thân email:**

Mở đầu 2–3 câu: tổng số tin mới, nhóm vị trí nào nhiều nhất, và một nhận xét
đáng chú ý (công ty lớn mở nhiều slot, mức lương bất thường, xu hướng tech stack).
Nếu `city` đang dùng giá trị mặc định, nói rõ ở đây.

Sau đó chia 5 section theo thứ tự: Data Analyst → Analytics Engineer →
Data Engineer → Business Intelligence → Business Analyst.
**Bỏ hẳn section nào không có tin mới** — đừng viết "không có tin".

Mỗi section là một bảng:

| ⭐ | Vị trí | Công ty | Lương | YOE | Stack chính | Ngày đăng | Link |

- **⭐:** đánh dấu nếu công ty nằm trong danh sách **công ty ưu tiên**. Để trống nếu không
- **Lương:** ba trường hợp, không có trường hợp thứ tư:
  - Tin ghi con số / khoảng → chép đúng như tin đăng (giữ đơn vị, ví dụ `25–35 triệu`, `$1,500–2,000`)
  - Tin ghi "Thoả thuận" / "Negotiable" / "Cạnh tranh" → ghi `Thoả thuận`
  - Tin không nhắc gì đến lương → ghi `không rõ`
- **YOE:** số năm kinh nghiệm yêu cầu; không nhắc → `không rõ`
- **Stack chính:** tối đa 4 công nghệ nổi bật nhất trong JD
- **Ngày đăng:** `yyyy-MM-dd` hoặc `không rõ`
- **Link:** anchor text ngắn "Xem tin", không dán URL trần

**Cuối email**, ghi các dòng báo cáo:

- Nguồn nào KHÔNG truy cập được lần này (captcha, lỗi, tường đăng nhập)
- Career page nào lỗi hoặc đổi cấu trúc
- `SEEN_COUNT`: số URL đã đọc được từ `seen_urls` ở Bước 1

Dòng `SEEN_COUNT` là để user tự kiểm tra bộ nhớ chống trùng còn sống. Nếu con số này
đột nhiên về 0 trong khi trước đó vẫn lớn, tức là có lỗi.

**Nếu không có tin mới nào:** vẫn gửi email, tiêu đề
`[Job Radar] {Thành phố} — không có tin mới`, thân email 1 dòng kèm `SEEN_COUNT`.
Để user biết hệ thống vẫn chạy chứ không phải đã chết.

## Ràng buộc chung

- Không bịa. Không tìm thấy lương thì ghi "không rõ", không suy đoán.
- Không tái sử dụng kết quả lần chạy trước. Mỗi lần phải quét lại thật.
- Output tiếng Việt, giữ nguyên tên vị trí và tên công ty theo tiếng Anh.
- Múi giờ tham chiếu: Asia/Ho_Chi_Minh (GMT+7).
