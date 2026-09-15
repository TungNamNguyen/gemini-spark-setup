---
name: vn-data-job-radar
description: Quét các trang tuyển dụng Việt Nam tìm tin tuyển dụng mới ngành dữ liệu (Data Analyst, Analytics Engineer, Data Engineer, Business Intelligence, Business Analyst), lọc theo thành phố và ngày đăng, khử trùng lặp bằng Google Sheet, rồi gửi email tổng hợp. Dùng khi cần theo dõi thị trường việc làm data tại Hà Nội hoặc TP.HCM.
---

# VN Data Job Radar

**Phiên bản skill: 3.3.** Luôn ghi số này vào dòng `Skill:` trong khối báo cáo cuối email,
để user biết task đang chạy đúng bản skill mới nhất.

## Mục tiêu

Tìm các tin tuyển dụng ngành dữ liệu **đăng trong 72 giờ gần nhất**, tại thành phố
được chỉ định, loại bỏ những tin đã gửi cho user trước đó, rồi gửi một email tổng hợp.

## Tham số đầu vào

| Tham số | Giá trị hợp lệ | Mặc định | Ghi chú |
| --- | --- | --- | --- |
| `city` | `Hà Nội` \| `TP.HCM` | `Hà Nội` | Nếu user không chỉ định, dùng mặc định và **ghi rõ trong mở đầu email** là đang dùng thành phố mặc định |
| `mode` | `full` \| `quick` \| `cleanup` | `full` | `full` = quét đủ Gmail + 5 nguồn web + career page. `quick` = chỉ Gmail + Xóm Jobs + LinkedIn (cửa sổ 24 giờ), bỏ career page — dùng cho buổi chiều. `cleanup` = **không quét gì**, chỉ dọn dẹp sheet và gửi email báo cáo riêng — xem mục "Chế độ cleanup" |

Không hỏi lại user khi thiếu tham số — skill chạy tự động, không có ai trả lời.

Lọc trùng ở mode `full` và `quick` đều dựa vào `seen_urls`. **Không** lọc theo "tin đăng kể từ lần
chạy trước" — chỉ cần URL chưa có trong SEEN là gửi.

## Định dạng dữ liệu chuẩn

Áp dụng thống nhất cho mọi chỗ ghi vào sheet và mọi phép so sánh ngày:

| Trường | Định dạng | Ví dụ |
| --- | --- | --- |
| `first_sent_at` | ISO 8601 có múi giờ | `2026-09-12T08:30:00+07:00` |
| `posted_date` | `yyyy-MM-dd`; không rõ thì `không rõ` | `2026-09-11` |
| `city` | Đúng một trong: `Hà Nội`, `TP.HCM`, `Remote`, `Hybrid` | `Hà Nội` |
| `role_group` | Đúng một trong: `DA`, `AE`, `DE`, `BI`, `BA` | `DE` |
| `source` | Đúng một trong: `xomjobs`, `linkedin`, `topcv`, `itviec`, `vietnamworks`, `gmail`, `career_page` | `topcv` |
| `alias_urls` | Các URL trùng đã gộp ở Bước 5, ngăn nhau bằng dấu phẩy, không có khoảng trắng. Rỗng nếu không có bản trùng | `https://linkedin.com/jobs/view/4123456789,https://itviec.com/it-jobs/data-engineer-abc` |

Mọi phép tính "72 giờ", "90 ngày" đều dùng múi giờ Asia/Ho_Chi_Minh (GMT+7).

### Quy đổi ngày đăng tương đối

Phần lớn nguồn hiển thị ngày đăng dạng tương đối. Quy đổi ngay khi trích xuất, lấy
mốc là **ngày chạy theo GMT+7**:

| Hiển thị trên tin | Quy đổi |
| --- | --- |
| "hôm nay", "today", "vừa xong", "x giờ trước", "x hours ago" | ngày chạy |
| "hôm qua", "yesterday", "1 ngày trước" | ngày chạy − 1 |
| "x ngày trước", "x days ago" | ngày chạy − x |
| "x tuần trước", "x weeks ago" | ngày chạy − 7x → luôn ngoài 72 giờ, loại |
| "30+ days ago", "hơn 30 ngày" | loại thẳng, không cần quy đổi |
| Không thấy ngày ở bất kỳ đâu | `không rõ` → xử lý theo Bước 5 |

## Chuẩn hoá URL

Áp dụng **trước khi so với SEEN** và **trước khi ghi vào sheet**. Cùng một job trên
cùng một trang phải luôn cho ra cùng một chuỗi URL.

**Quy tắc chung (áp dụng cho mọi URL, theo thứ tự):**

1. Bỏ fragment (`#...`)
2. Bỏ query string, **trừ** các tham số định danh job trong allowlist:
   `currentJobId`, `jobId`, `job_id`, `job`, `id`, `gh_jid`, `lever_id`, `requisitionId`, `jk`.
   Giữ các tham số này (sắp xếp theo a→z), bỏ tất cả phần còn lại: `utm_*`, `ref`, `src`,
   `fbclid`, `gclid`, `trk`, `refId`, `trackingId`, ID phiên
3. **Chốt chặn:** nếu sau khi bỏ query mà path không còn chuỗi định danh nào (không có
   dãy số ≥ 4 chữ số và không có slug dài), **giữ nguyên query string gốc**. Thà dư tham
   số còn hơn để link chết hoặc để hai job khác nhau rơi về cùng một URL rồi biến mất
   vì bị coi là trùng
4. Chuyển `http://` → `https://`
5. Bỏ `www.` ở đầu host
6. Lowercase toàn bộ host (không lowercase path)
7. Bỏ dấu `/` cuối cùng

**Riêng LinkedIn:** ID job là dãy số ở cuối path hoặc nằm trong `?currentJobId=`, có thể có
hoặc không có slug tiêu đề đứng trước, và subdomain có thể là `www.`, `vn.` hoặc không có.
Chuẩn hoá về `https://linkedin.com/jobs/view/{id}`. Ví dụ:

- `vn.linkedin.com/jobs/view/data-analyst-at-vng-4123456789?trackingId=abc` → `https://linkedin.com/jobs/view/4123456789`
- `linkedin.com/jobs/search/?currentJobId=4123456789&f_TPR=r259200` → `https://linkedin.com/jobs/view/4123456789`

**Các nguồn khác:** chỉ áp dụng quy tắc chung, giữ nguyên path. Không tự cắt slug hay
đoán ID — URL sau chuẩn hoá vẫn phải là link click được, vì nó được dùng làm link
trong email. Trùng do slug đổi hoặc cùng tin trên nhiều trang đã được xử lý bằng
khoá phụ `company|title` ở Bước 5.

## Cấu trúc bộ nhớ: Google Sheet `Job Radar Tracker`

Sheet có 3 tab với vai trò tách bạch. Tuân thủ đúng vai trò này là bắt buộc,
vì nó quyết định hiệu năng của mọi lần chạy.

| Tab | Cột | Vai trò |
| --- | --- | --- |
| `seen_urls` | A: job_url | **CHỈ ĐỌC + append.** Tab duy nhất được đọc trong lần chạy thường |
| `jobs_detail` | A–I | **CHỈ GHI** ở mode `full`/`quick`. **Chỉ mode `cleanup`** được đọc tab này |
| `archive` | A–I | **CHỈ GHI.** Không bao giờ đọc, không bao giờ xoá |

Header của `jobs_detail` và `archive`:
`job_url | title | company | city | role_group | posted_date | source | first_sent_at | alias_urls`

## Bước 1 — Đọc bộ nhớ chống trùng (LÀM ĐẦU TIÊN, KHÔNG BỎ QUA)

Mở `Job Radar Tracker` → tab `seen_urls` → đọc **chỉ cột A** vào danh sách SEEN.

**Không đọc tab `jobs_detail` hoặc `archive` ở bước này.** Đọc chúng làm chậm task
và không mang lại thông tin gì thêm.

Ghi lại số lượng URL đọc được (gọi là `SEEN_COUNT`) để báo cáo ở cuối task.

Nếu sheet hoặc tab không tồn tại, tạo mới theo đúng cấu trúc trên rồi đi tiếp
với SEEN rỗng.

## Bước 2 — Quét Gmail Job Alerts (quét TRƯỚC khi duyệt web)

Đọc email trong 72 giờ gần nhất từ 5 Gmail label sau:

| Gmail Label | Nguồn tương ứng |
| --- | --- |
| `ITViec Job Alerts` | ITviec |
| `LinkedIn Job Alerts` | LinkedIn Jobs |
| `VietnamWorks Job Alert` | VietnamWorks |
| `TopCV` | TopCV |
| `Xom Job Alerts` | Xóm Jobs |

Với mỗi email, trích xuất danh sách job URL + tên vị trí + công ty.
Chuẩn hoá URL (mục "Chuẩn hoá URL" ở trên) rồi so với SEEN — chỉ giữ URL chưa có.

**Tại sao quét Gmail trước:**

- Nhanh hơn browse web rất nhiều (chỉ đọc text)
- Không bị captcha, rate-limit, hay giới hạn guest
- Là nguồn duy nhất cho TopCV (web chặn bot), và bắt được tin LinkedIn chỉ hiện khi
  đăng nhập mà endpoint guest ở Bước 3 không thấy
- Bước 3 (quét web) sau đó chỉ cần bổ sung những gì email chưa có

**Lưu ý:** chỉ trích URL job từ email, không đánh dấu email là đã đọc hay xoá.
Nếu label không tồn tại hoặc không có email mới, bỏ qua và đi tiếp.

## Bước 3 — Quét web

Quét lần lượt 5 nguồn dưới đây. **Mở đúng URL đã cho** thay vì vào trang chủ rồi gõ ô
tìm kiếm — mỗi lần điều hướng thừa tốn 10–20 giây quota và dễ dính bộ lọc mặc định
sai. Nếu gặp lỗi ở nguồn nào thì ghi nhận và đi tiếp.

**Nếu `mode = quick`:** chỉ quét Xóm Jobs và LinkedIn (dùng bộ URL `quick` ở mục 3.2),
bỏ 3 nguồn còn lại và bỏ hẳn mục "Công ty ưu tiên". Vẫn đánh dấu ⭐ nếu công ty nằm
trong danh sách ưu tiên.

Giá trị `{city}` trong các URL bên dưới thay theo bảng này (mã hoá URL khi cần —
khoảng trắng → `%20`, dấu phẩy → `%2C`):

| `city` | Xóm Jobs `location=` | LinkedIn `location=` | ITviec path |
| --- | --- | --- | --- |
| Hà Nội | `Hà Nội` | `Hanoi, Vietnam` | `ha-noi` |
| TP.HCM | `TP.Hồ Chí Minh` | `Ho Chi Minh City, Vietnam` | `ho-chi-minh-hcm` |

### 3.1 Xóm Jobs (jobs.xomdata.com) — quét đầu tiên

Chuyên Data & AI Việt Nam, không chặn bot, bộ lọc nằm ngay trên URL. Mở lần lượt 4 URL,
mỗi URL đọc 1 trang là đủ (đã lọc 3 ngày + thành phố nên thường dưới 20 tin):

```
https://jobs.xomdata.com/?category=data-analyst&location={city}&posted_days=3
https://jobs.xomdata.com/?category=data-engineer&location={city}&posted_days=3
https://jobs.xomdata.com/?category=analytics-engineer&location={city}&posted_days=3
https://jobs.xomdata.com/?category=business-analyst&location={city}&posted_days=3
```

Xóm Jobs không có category BI riêng — tin BI nằm trong `data-analyst`. Mỗi tin trên trang
danh sách có tiêu đề, công ty, địa điểm, level, ngày đăng tương đối.

**Về URL:** Xóm Jobs là nguồn tổng hợp; trang chi tiết `jobs.xomdata.com/jobs/{id}` có
link gốc, phần lớn trỏ về LinkedIn hoặc ITviec (**không** có TopCV). **Luôn lưu URL gốc**
(đã chuẩn hoá theo mục "Chuẩn hoá URL") thay vì URL `jobs.xomdata.com/…`, để cùng một tin
không bị lưu 2 URL khác nhau rồi lọt qua bộ lọc trùng. Tin không có link gốc thì giữ
URL Xóm Jobs.

### 3.2 LinkedIn — dùng endpoint guest, KHÔNG mở trang `jobs/search`

`linkedin.com/jobs/search` là trang cho người dùng: cần JavaScript, và guest bị tường
đăng nhập ngay sau trang đầu. Thay vào đó mở thẳng endpoint mà chính trang đó gọi ngầm
để tải thêm tin cho khách — trả về danh sách thẻ tin HTML thuần, **không cần đăng nhập**,
phân trang bằng `start`:

```
https://www.linkedin.com/jobs-guest/jobs/api/seeMoreJobPostings/search?keywords={từ khoá}&location={city}&f_TPR={cửa sổ}&start={0|10|20}
```

- `f_TPR=r259200` = 72 giờ. Mode `quick` dùng `f_TPR=r86400` (24 giờ): lần `full` buổi
  sáng đã phủ 72 giờ, buổi chiều chỉ cần tin mới trong ngày
- Mỗi trang đúng 10 tin, `start` tăng từng 10. Trang không có tin nào = hết kết quả, dừng
- `keywords` hỗ trợ boolean với dấu ngoặc kép: `"Data Analyst" OR "Data Engineer"`

**Bộ URL theo mode:**

| Mode | `keywords=` | `start=` | `f_TPR=` | Số lần mở tối đa |
| --- | --- | --- | --- | --- |
| `full` | 5 query riêng, mỗi query một nhóm: `Data Analyst` · `Data Engineer` · `Business Intelligence` · `Analytics Engineer` · `Business Analyst` | `0`, `10` | `r259200` | 10 |
| `quick` | 1 query gộp: `"Data Analyst" OR "Data Engineer" OR "Business Intelligence" OR "Analytics Engineer" OR "Business Analyst"` | `0`, `10`, `20` | `r86400` | 3 |

Ví dụ `full`, Hà Nội, nhóm DA, trang 2:

```
https://www.linkedin.com/jobs-guest/jobs/api/seeMoreJobPostings/search?keywords=Data%20Analyst&location=Hanoi%2C%20Vietnam&f_TPR=r259200&start=10
```

Ví dụ `quick`, TP.HCM, trang 1:

```
https://www.linkedin.com/jobs-guest/jobs/api/seeMoreJobPostings/search?keywords=%22Data%20Analyst%22%20OR%20%22Data%20Engineer%22%20OR%20%22Business%20Intelligence%22%20OR%20%22Analytics%20Engineer%22%20OR%20%22Business%20Analyst%22&location=Ho%20Chi%20Minh%20City%2C%20Vietnam&f_TPR=r86400&start=0
```

**Đọc mỗi thẻ tin:**

- Tiêu đề, công ty, địa điểm. Các dạng `Hanoi Capital Region`, `Hanoi, Hanoi, Vietnam`,
  `Hoàn Kiếm district, Hanoi, Vietnam`, `Ho Chi Minh City, Vietnam`,
  `Ho Chi Minh City Metropolitan Area` đều tính là đúng thành phố
- Link dạng `linkedin.com/jobs/view/{slug}-{id}?position=…&trackingId=…` → chuẩn hoá về
  `https://linkedin.com/jobs/view/{id}`. ID cũng nằm trong thuộc tính
  `data-entity-urn="urn:li:jobPosting:{id}"` của thẻ
- **Ngày đăng:** thẻ `<time>` có thuộc tính `datetime="yyyy-MM-dd"` — nếu đọc được DOM
  thì lấy thẳng giá trị này, không cần quy đổi. Nếu chỉ thấy chữ "12 hours ago" thì quy
  đổi theo bảng ở đầu skill
- Kết quả có lẫn tin ngoài thành phố (tỉnh lân cận) và tin không liên quan (Python
  Developer, Data Scientist, Growth Specialist…) — lọc ở Bước 4–5 như bình thường

**Nếu lần mở đầu tiên trả về trang đăng nhập, trang lỗi, hoặc trang trống trong khi
các nguồn khác vẫn bình thường:** IP của remote browser đang bị LinkedIn chặn. Ghi
LinkedIn là "không truy cập được (IP bị chặn)", **không thử các URL còn lại**, dựa vào
Gmail alert ở Bước 2. Không dùng `linkedin.com/jobs/search` làm phương án dự phòng —
nó bị chặn nặng hơn.

### 3.3 TopCV (topcv.vn)

Chặn bot bằng Cloudflare: remote browser gần như luôn gặp captcha, và Xóm Jobs
**không** aggregate TopCV. Nguồn thực sự cho TopCV là Gmail label `TopCV` ở Bước 2.

Chỉ thử **một lần**: mở `https://www.topcv.vn/tim-viec-lam-data-analyst?sort_by=new`,
chờ tối đa 30 giây. Nếu ra danh sách tin (thường chỉ khi Spark chạy trên Chrome local đã
đăng nhập) → quét thêm `tim-viec-lam-data-engineer` và `tim-viec-lam-business-analyst`
cùng tham số, lọc địa điểm theo `{city}`, mỗi từ khoá 1 trang. Nếu gặp captcha /
"Attention Required" / trang trống → ghi "không truy cập được (captcha)", đi tiếp ngay.
**Không** chờ captcha, không reload, không thử URL khác.

### 3.4 ITviec (itviec.com)

Tốt nhất cho Data Engineer / Analytics Engineer. URL đã lọc sẵn từ khoá + thành phố +
sắp xếp mới nhất:

```
https://itviec.com/it-jobs/data-analyst/{city}?sort=newest
https://itviec.com/it-jobs/data-engineer/{city}?sort=newest
https://itviec.com/it-jobs/business-intelligence/{city}?sort=newest
```

Mỗi URL 1 trang. Tin hiện "Posted x days ago" và đã sắp xếp mới nhất trước — gặp tin
đầu tiên quá 3 ngày thì dừng URL đó.

### 3.5 VietnamWorks (vietnamworks.com)

Mở `https://www.vietnamworks.com/viec-lam?q=data%20analyst`, chọn sắp xếp "Ngày đăng
mới nhất", lọc địa điểm theo `{city}`. Lặp lại với `q=data%20engineer`. Trang render
bằng JavaScript nên chờ tải xong rồi mới đọc; mỗi từ khoá 1 trang.

### Trần khối lượng cho mỗi nguồn

Chạm bất kỳ ngưỡng nào thì dừng nguồn đó, đi tiếp nguồn sau, và ghi vào báo cáo cuối
email là nguồn đó "chạm trần":

- Tối đa số lần mở URL đã ghi ở từng mục (LinkedIn `full` 10, `quick` 3; Xóm Jobs 4;
  ITviec 3; TopCV 3; VietnamWorks 2)
- Hoặc tối đa **40 tin** đã trích xuất từ nguồn đó
- Hoặc tối đa **3 phút** cho một nguồn

Ngưỡng này để một nguồn chậm không nuốt hết thời gian của cả task. Tin bỏ lỡ hôm nay
vẫn nằm trong 72 giờ nên lần chạy sau còn bắt được.

**Về cách duyệt web:** skill thường chạy tự động khi user không mở máy, nên
**mặc định dùng remote browser**. Nếu gặp captcha hoặc tường đăng nhập, ghi nhận nguồn
đó là "không truy cập được" và đi tiếp — không dừng cả task vì một nguồn.

### Công ty ưu tiên

Sau khi quét 5 nguồn chính, quét thêm trang tuyển dụng (career page) của các
công ty dưới đây. Chỉ tìm vị trí liên quan đến data/analytics.

NAB Innovation Centre Vietnam, Crossian, VNG, Grab Vietnam, Shopee Vietnam,
MoMo, ZaloPay, VNPAY, Techcombank, VPBank, MB Bank, Vingroup, VinAI, VinBigData,
One Mount, Be Group, Lazada Vietnam, GreenSM.

**Cách khớp tên công ty (dùng cho cả việc đánh ⭐):** tin đăng thật hiếm khi ghi đúng
tên trong danh sách. Trước khi so khớp, chuẩn hoá cả hai phía: lowercase, bỏ dấu, bỏ
hậu tố/tiền tố pháp nhân (`công ty`, `cổ phần`, `cp`, `tnhh`, `jsc`, `corporation`,
`corp`, `co.,ltd`, `ltd`, `vietnam`, `việt nam`), bỏ phần trong ngoặc. Sau đó khớp
kiểu **chứa** theo cả hai chiều. Ví dụ khớp: "VNG Corporation", "Công ty CP VNG" → `VNG`;
"MoMo (M_Service JSC)" → `MoMo`; "Công ty CP Giải pháp Thanh toán Việt Nam (VNPAY)" → `VNPAY`.

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

**Khi một tin khớp nhiều nhóm** (ví dụ "Analytics Engineer (Data Platform)" khớp cả AE
lẫn DE; "BI Analyst" khớp cả BI lẫn DA): xét theo thứ tự **AE → DE → BI → DA → BA**,
nhóm khớp đầu tiên thắng. Mỗi tin chỉ được xuất hiện ở **đúng một** section trong email.

## Bước 5 — Quy tắc lọc

Ở bước này chỉ áp dụng những luật quyết định được từ trang danh sách. Các luật cần đọc
JD được đánh dấu **[cần JD]** và áp dụng ở Bước 6.

**GIỮ nếu:**

- Ngày đăng nằm trong 72 giờ tính đến thời điểm chạy (sau khi quy đổi ngày tương đối)
- Địa điểm khớp `city` được chỉ định (chấp nhận "Hybrid" và "Remote — Vietnam")
- URL đã chuẩn hoá KHÔNG nằm trong SEEN

**LOẠI nếu:**

- Không hiển thị ngày đăng ở bất kỳ đâu → loại.
  **Ngoại lệ duy nhất:** tin từ career page của công ty ưu tiên (Bước 3) được giữ với `posted_date = "không rõ"`.
- Tên có chữ "Analyst" nhưng thực chất không liên quan dữ liệu: Financial Analyst
  thuần kế toán, Credit Analyst thẩm định hồ sơ, Investment Analyst.
  Nếu JD không nhắc SQL / Python / Excel nâng cao / BI tool / data warehouse thì loại. **[cần JD]**
- Thực tập không lương **[cần JD]**

**Về tin `Remote` / `Hybrid`:** hai task Hà Nội và TP.HCM dùng chung một `seen_urls`, nên
một tin Remote sẽ về email của task nào chạy trước trong ngày (Hà Nội 08:00) và không
xuất hiện ở task còn lại. Đây là hành vi cố ý để không gửi trùng — không phải lỗi.

**Gộp tin trùng nội dung trong cùng lần chạy:**

Cùng một JD có thể xuất hiện trên nhiều nguồn (TopCV, ITviec, LinkedIn) hoặc được
nhiều headhunter đăng lại → URL khác nhau nhưng là một tin. Trước khi ghi sheet và
soạn email, gộp theo khoá phụ:

`lowercase(company) + "|" + lowercase(title đã bỏ ký tự đặc biệt và khoảng trắng thừa)`

Với mỗi nhóm trùng, tạo **một bản hợp nhất**:

1. **Chọn bản đại diện** (lấy `job_url` và `source` từ đây) theo thứ tự nguồn:
   `career_page` > `itviec` > `topcv` > `vietnamworks` > `linkedin` > `xomjobs` > `gmail`
2. **Hợp nhất từng trường còn lại**, không lấy nguyên bản đại diện: với
   `posted_date`, `Lương`, `YOE`, `Stack chính`, `city` — lấy giá trị **cụ thể** từ bất kỳ
   bản nào có, ưu tiên bản đại diện nếu nhiều bản đều có. Không để `không rõ` đè lên một
   giá trị thật. Đây là trường hợp hay gặp: bản career page thắng ở bước 1 nhưng lại là
   bản duy nhất thiếu ngày đăng và lương
3. **Gom URL của mọi bản còn lại** vào trường `alias_urls`

- Với headhunter: nếu company là tên công ty headhunt (Navigos, Robert Walters, Adecco,
  ManpowerGroup, HR2B, Talentnet…) và JD giống hệt tin của công ty thật → giữ tin của công ty thật
- Kết quả: **1 dòng** trong `jobs_detail`, **1 dòng** trong email, nhưng **tất cả URL**
  đều vào `seen_urls` ở Bước 7 (để lần sau không hiện lại)

**Riêng Business Analyst:** ở Việt Nam phần lớn BA là IT BA (viết tài liệu,
gom requirement), không đụng dữ liệu. Chỉ giữ tin có nhắc SQL, dashboard, data,
reporting, hoặc analytics trong JD. Trong email, đánh dấu rõ tin nào là
"BA thiên data" và tin nào là "IT BA thuần". **[cần JD]**

## Bước 6 — Mở JD lấy chi tiết

Email ở Bước 8 bắt buộc phải có Lương, YOE, Stack chính; các luật **[cần JD]** ở Bước 5
cũng chỉ quyết được sau khi đọc JD. Trang danh sách và Gmail alert hầu như không đủ
thông tin đó, nên phải mở JD của từng tin còn lại.

**Trần số JD mở trong một lần chạy:** `full` = 40 tin, `quick` = 20 tin.

Nếu số tin còn lại vượt trần, xếp thứ tự ưu tiên rồi mở đến khi chạm trần:

1. Công ty nằm trong danh sách ưu tiên (⭐)
2. `posted_date` mới nhất
3. Nhóm `AE` → `DE` → `BI` → `DA` → `BA`

**Riêng tin LinkedIn** — kể cả tin đến từ Gmail alert hay từ Xóm Jobs mà URL gốc là
LinkedIn: **không** mở `linkedin.com/jobs/view/{id}` (guest bị tường đăng nhập). Mở
endpoint guest của JD:

```
https://www.linkedin.com/jobs-guest/jobs/api/jobPosting/{id}
```

Trả về JD đầy đủ, kèm Seniority level, Employment type, Job function — không cần đăng
nhập. URL này **chỉ để đọc**; `job_url` ghi vào sheet và link trong email vẫn là
`https://linkedin.com/jobs/view/{id}`.

Với mỗi JD mở được: lấy Lương, YOE, Stack chính (tối đa 4 công nghệ), và áp dụng nốt
các luật **[cần JD]** ở Bước 5.

**Tin không mở được JD** (timeout, tường đăng nhập, trang lỗi) hoặc **tin vượt trần**:

- Vẫn giữ và vẫn gửi, các trường thiếu ghi `không rõ`
- **Trừ** tin thuộc nhóm `BA` và tin có chữ "Analyst" thuộc diện nghi ngờ ở Bước 5 —
  hai loại này cần JD mới quyết được, không đọc được thì **loại**, để email không bị
  lẫn tin không liên quan dữ liệu

Ghi lại số JD đã mở và số tin bị bỏ qua vì chạm trần để báo cáo ở cuối email.

## Bước 7 — Ghi vào sheet (LÀM TRƯỚC KHI GỬI EMAIL)

Với mỗi job mới, ghi **đúng thứ tự này**:

1. Append một dòng đầy đủ 9 cột vào tab **`jobs_detail`**:
   `job_url | title | company | city | role_group | posted_date | source | first_sent_at | alias_urls`
   với `first_sent_at` = timestamp hiện tại theo ISO 8601 (xem mục Định dạng dữ liệu chuẩn)
2. Append URL đã chuẩn hoá vào tab **`seen_urls`**, cột A — **cả `job_url` lẫn từng URL
   trong `alias_urls`**, mỗi URL một dòng

**Tại sao `jobs_detail` trước, `seen_urls` sau:** `seen_urls` là cái chặn. Nếu ghi nó
trước rồi hỏng giữa chừng, tin vừa nằm trong SEEN vừa không có trong `jobs_detail` —
mất im lặng, không bao giờ được gửi. Theo thứ tự này, trường hợp xấu nhất là lần sau
gửi lại một tin — nhìn thấy được và vô hại.

Ghi trước, gửi sau. Nếu email lỗi thì lần chạy tới cũng không gửi trùng.

## Bước 8 — Định dạng email

Gửi tới email của user. Dùng HTML, không dùng markdown thô.

**Tiêu đề:** `[Job Radar] {Thành phố} — {N} tin mới — {dd/MM HH:mm}`

**Thân email** — dựng đúng theo khung dưới đây. Thay phần trong `{}`; các hàng
trong bảng chỉ là ví dụ minh hoạ định dạng.

```html
<p>{Mở đầu 2–3 câu: tổng số tin mới, nhóm vị trí nào nhiều nhất, một nhận xét đáng chú ý
— công ty lớn mở nhiều slot, mức lương bất thường, xu hướng stack.
Nếu city đang dùng giá trị mặc định, nói rõ ở đây.}</p>

<h3>Data Analyst ({n} tin)</h3>
<table border="1" cellpadding="6" cellspacing="0" style="border-collapse:collapse">
  <tr><th>Vị trí</th><th>Công ty</th><th>Lương</th><th>YOE</th><th>Stack chính</th><th>Ngày đăng</th><th>Link</th></tr>
  <tr><td>Senior Data Analyst</td><td>⭐ MoMo</td><td>Thoả thuận</td><td>3</td><td>SQL, Python, Looker, BigQuery</td><td>2026-09-14</td><td><a href="{job_url}">Xem tin</a></td></tr>
  <tr><td>Product Analyst</td><td>Base.vn</td><td>20–28 triệu</td><td>2</td><td>SQL, Metabase, Excel</td><td>2026-09-13</td><td><a href="{job_url}">Xem tin</a></td></tr>
</table>

<h3>Analytics Engineer ({n} tin)</h3>
<!-- bảng cùng cấu trúc -->

<h3>Data Engineer ({n} tin)</h3>
<!-- bảng cùng cấu trúc -->

<h3>Business Intelligence ({n} tin)</h3>
<!-- bảng cùng cấu trúc -->

<h3>Business Analyst ({n} tin)</h3>
<!-- bảng cùng cấu trúc; cột Vị trí ghi thêm loại, ví dụ "Business Analyst — BA thiên data" -->

<hr>
<p><b>Báo cáo lần chạy</b></p>
<ul>
  <li>Nguồn không truy cập được: {TopCV (captcha), LinkedIn (IP bị chặn) | không có}</li>
  <li>Nguồn chạm trần: {ITviec (30 tin, còn tin chưa quét) | không có}</li>
  <li>Career page lỗi: {VinBigData (timeout) | không có}</li>
  <li>Đã mở {n} JD, bỏ qua {m} tin vì chạm trần</li>
  <li>SEEN_COUNT: {SEEN_COUNT}</li>
  <li>Skill: v3.3</li>
</ul>
<p>📋 Link tới Google Sheet: <a href="{url của sheet}">Job Radar Tracker</a> — tab <code>jobs_detail</code> có đủ mọi tin từ trước tới nay, tab <code>archive</code> có tin cũ hơn 90 ngày.</p>
```

**Quy tắc dựng:**

- Dùng đúng các thẻ và thứ tự trong khung. **Không** thêm CSS, màu nền, font, ảnh,
  `<div>`/`<span>` hay bất kỳ thẻ nào khác — Gmail bỏ phần lớn CSS, và email nào cũng
  phải trông giống nhau
- Thứ tự section cố định: Data Analyst → Analytics Engineer → Data Engineer →
  Business Intelligence → Business Analyst. **Section không có tin thì bỏ hẳn** cả `<h3>`
  lẫn bảng — đừng viết "không có tin"
- Trong mỗi bảng: tin ⭐ xếp lên đầu, sau đó theo `posted_date` mới nhất trước
- **Công ty:** thêm `⭐ ` trước tên nếu công ty nằm trong danh sách **công ty ưu tiên**
  (khớp theo quy tắc ở Bước 3). Không có cột ⭐ riêng
- **Lương:** ba trường hợp, không có trường hợp thứ tư:
  - Tin ghi con số / khoảng → chép đúng như tin đăng (giữ đơn vị, ví dụ `25–35 triệu`, `$1,500–2,000`)
  - Tin ghi "Thoả thuận" / "Negotiable" / "Cạnh tranh" → ghi `Thoả thuận`
  - Tin không nhắc gì đến lương, hoặc không mở được JD → ghi `không rõ`
- **YOE:** số năm kinh nghiệm yêu cầu; không nhắc → `không rõ`
- **Stack chính:** tối đa 4 công nghệ nổi bật nhất trong JD
- **Ngày đăng:** `yyyy-MM-dd` hoặc `không rõ`
- **Link:** anchor text ngắn "Xem tin" trỏ tới `job_url`, không dán URL trần
- **Vị trí (nhóm BA):** ghi thêm ` — BA thiên data` hoặc ` — IT BA thuần` sau tên vị trí,
  theo kết quả Bước 5
- **Khối báo cáo:** đủ 6 dòng, đúng thứ tự, luôn có mặt; dòng nào không có gì thì ghi
  `không có`, không bỏ dòng. `SEEN_COUNT` là để user tự kiểm tra bộ nhớ chống trùng còn
  sống — nếu đột nhiên về 0 trong khi trước đó vẫn lớn, tức là có lỗi. `Skill:` là số
  phiên bản ghi ở đầu skill — user đối chiếu để biết Spark đã nhận bản mới chưa
- **Link sheet:** trỏ tới đúng file `Job Radar Tracker` đã mở ở Bước 1. Không bịa URL

**Nếu không có tin mới nào:** vẫn gửi email, tiêu đề
`[Job Radar] {Thành phố} — không có tin mới`. Thân email chỉ gồm một dòng
`<p>Không có tin mới trong 72 giờ qua tại {Thành phố}.</p>`, rồi `<hr>`, khối báo cáo
và link sheet y như khung trên. Để user biết hệ thống vẫn chạy chứ không phải đã chết,
và nhìn khối báo cáo là biết "không có tin" là thật hay do nguồn hỏng.

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
3. Tính `CUTOFF = now − 90 ngày`. Chọn các dòng có `first_sent_at < CUTOFF` → tập `OLD`,
   sắp xếp theo `first_sent_at` tăng dần (cũ nhất trước)
4. **Guard an toàn — kiểm tra trước khi xoá bất kỳ thứ gì:**
   - Nếu `DETAIL_BEFORE < 50` → kết quả `SKIPPED`, lý do "chưa đủ dữ liệu để dọn"
   - Nếu `OLD` rỗng → kết quả `SKIPPED`, lý do "không có dòng nào quá 90 ngày"
   - Nếu `|OLD| > 80%` × `DETAIL_BEFORE` → kết quả `BLOCKED`, không xoá gì,
     lý do "{|OLD|}/{DETAIL_BEFORE} dòng vượt ngưỡng 80%, bất thường, cần user kiểm tra thủ công"
5. **Trần mỗi lần dọn:** đặt `BATCH = floor(30% × DETAIL_BEFORE)`.
   Nếu `|OLD| > BATCH`, chỉ xử lý `BATCH` dòng **cũ nhất** trong lần này; số dòng còn lại
   gọi là `PENDING` và sẽ được dọn ở các tuần sau. Gọi tập thực sự xử lý là `BATCH_SET`.
   - `PENDING > 0` → kết quả cuối là `PARTIAL` thay vì `DONE`
   - Không bao giờ để tình trạng quá nhiều dòng cũ làm task đứng im: luôn dọn được
     `BATCH` dòng mỗi tuần cho đến hết
6. Thứ tự thao tác (phải đúng thứ tự này để lỗi giữa chừng không mất dữ liệu):
   1. Append toàn bộ `BATCH_SET` vào `archive` **trước**
   2. Xác nhận số dòng đã append vào `archive` bằng `|BATCH_SET|` — nếu không khớp → kết quả `FAILED`,
      không xoá gì, ghi rõ số dòng đã append
   3. Xoá các dòng `BATCH_SET` khỏi `jobs_detail`
   4. Xoá các URL tương ứng khỏi `seen_urls`: với mỗi dòng trong `BATCH_SET`, xoá `job_url`
      **và** từng URL trong `alias_urls`, khớp chuỗi chính xác.
      **Ngoại lệ — không xoá:** dòng có `posted_date = "không rõ"` (tin career page).
      Loại tin này không bị bộ lọc 72 giờ chặn, nên nếu xoá khỏi `seen_urls` nó sẽ được
      gửi lại như tin mới ở lần quét kế tiếp. Giữ URL của chúng trong `seen_urls` vĩnh viễn;
      đếm số URL giữ lại vào `KEPT_URLS`
   5. Kết quả `DONE` (hoặc `PARTIAL` nếu `PENDING > 0`)
7. **KHÔNG bao giờ xoá dữ liệu khỏi `archive`**
8. Đếm lại `DETAIL_AFTER`, `SEEN_AFTER`, `ARCHIVE_TOTAL`

### Email báo cáo cleanup

**Tiêu đề:** `[Job Radar] Dọn dẹp tuần — {DONE|PARTIAL|SKIPPED|BLOCKED|FAILED} — {dd/MM}`

**Thân email** (HTML, ngắn, không có bảng job): một `<p>` mở đầu nếu cần, rồi một `<ul>`
mỗi mục một `<li>` theo đúng thứ tự dưới, cuối cùng là dòng link sheet. Không thêm thẻ khác.

- Kết quả: `DONE` / `PARTIAL` / `SKIPPED` / `BLOCKED` / `FAILED` + lý do (nếu không phải `DONE`)
- Đã archive: `{|BATCH_SET|}` dòng (0 nếu không dọn)
- Còn chờ tuần sau: `{PENDING}` dòng (chỉ ghi nếu > 0)
- `jobs_detail`: `{DETAIL_BEFORE}` → `{DETAIL_AFTER}` dòng
- `seen_urls`: `{SEEN_BEFORE}` → `{SEEN_AFTER}` URL
- `archive`: tổng `{ARCHIVE_TOTAL}` dòng
- URL giữ lại vì tin không có ngày đăng: `{KEPT_URLS}` (chỉ ghi nếu > 0)
- Dòng lỗi định dạng ngày bị bỏ qua: `{BAD_ROWS}` (chỉ ghi nếu > 0)
- Dòng cũ nhất còn lại trong `jobs_detail`: `{first_sent_at nhỏ nhất}`
- Dòng cuối, ngoài `<ul>`: `<p>📋 Link tới Google Sheet: <a href="{url của sheet}">Job Radar Tracker</a></p>`

**Luôn gửi email**, kể cả `SKIPPED` — để user biết task còn sống. Với `BLOCKED` và `FAILED`,
mở đầu email bằng một câu nói rõ cần user vào sheet kiểm tra.

Việc này giữ `seen_urls` ổn định ở mức vài nghìn dòng thay vì phình vô hạn,
nên mỗi lần đọc ở Bước 1 luôn nhanh.

## Ràng buộc chung

- Không bịa. Không tìm thấy lương thì ghi "không rõ", không suy đoán.
- Không tái sử dụng kết quả lần chạy trước. Mỗi lần phải quét lại thật.
- Output tiếng Việt, giữ nguyên tên vị trí và tên công ty theo tiếng Anh.
- Múi giờ tham chiếu: Asia/Ho_Chi_Minh (GMT+7).
