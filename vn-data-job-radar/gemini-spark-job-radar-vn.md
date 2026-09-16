# Gemini Spark — Job Radar cho ngành Data (Hà Nội / TP.HCM)

**Phiên bản 3.4** — cấu trúc Sheet 3 tab, LinkedIn quét qua endpoint guest (không cần đăng nhập), Gmail Job Alerts là nguồn chính cho TopCV, remote browser, task dọn dẹp tách riêng, quy tắc tự chủ (không dừng chờ user giữa task), 16 URL career page cố định quét hết toàn bộ trang, kèm hướng dẫn setup từng bước.

Thời gian setup: **2 ngày**, tổng khoảng 60 phút thao tác.
Ngày 1 (~45 phút): Bước 1 → 3b.2. Ngày 2 (~15 phút + 25 phút chờ test): Bước 3b.3 → 9, sau khi email alert đầu tiên đã về.

- [Tổng quan hệ thống](#tổng-quan-hệ-thống)
- [Phần 0 — Setup từng bước](#phần-0--setup-từng-bước) ← bắt đầu ở đây
- [Phần 1 — Luồng chạy của Skill](#phần-1--luồng-chạy-của-skill)
- [Phần 2 — Nội dung Task](#phần-2--nội-dung-task)
- [Phần 3 — Email nhận được trông thế nào](#phần-3--email-nhận-được-trông-thế-nào)
- [Phần 4 — Phương án 10 task](#phần-4--phương-án-10-task)
- [Phần 5 — Xử lý sự cố](#phần-5--xử-lý-sự-cố)

**Nguyên tắc thiết kế của bản này:**

- Task chạy tự động lúc bạn không mở máy → **dùng remote browser**, không phụ thuộc Chrome local.
- TopCV chặn remote browser → **Gmail Job Alerts là nguồn chính cho TopCV**. LinkedIn quét qua endpoint guest `jobs-guest` (không cần đăng nhập), Gmail là dự phòng.
- Chống trùng bằng Google Sheet, không bằng trí nhớ của agent.
- Dọn dẹp sheet là **task riêng**, không nhét vào task quét.

---

# Tổng quan hệ thống

Sau khi setup xong, bạn sẽ có đúng những thứ sau:

| Thành phần | Số lượng   | Tên / giá trị                                                                                          | Tạo ở bước |
| ------------ | ------------- | --------------------------------------------------------------------------------------------------------- | -------------- |
| Google Sheet | 1 file, 3 tab | `Job Radar Tracker` → `seen_urls`, `jobs_detail`, `archive`                                      | 3              |
| Gmail label  | 5             | `ITViec Job Alerts`, `LinkedIn Job Alerts`, `VietnamWorks Job Alert`, `TopCV`, `Xom Job Alerts` | 3b             |
| Gmail filter | 5             | Mỗi filter gắn 1 label theo người gửi                                                                | 3b             |
| Spark Skill  | 1             | `vn-data-job-radar` (từ file `SKILL.md`)                                                             | 4              |
| Spark Task   | 3             | Hà Nội · TP.HCM · Dọn dẹp                                                                           | 5, 8, 8b       |
| Schedule     | 5             | HN 08:00 full / 17:30 quick · HCM 09:30 full / 17:50 quick · Dọn dẹp T2 07:00                         | 9              |

Email bạn nhận mỗi ngày: **4 email** (2 thành phố × sáng/chiều). Thứ Hai thêm **1 email dọn dẹp**.

Luồng dữ liệu:

```
Gmail alert (5 label) ─┐
Web (5 trang)          ├─► lọc / gộp trùng ─► so với seen_urls ─► ghi sheet ─► email
Career page (ưu tiên) ─┘                              ▲                 │
                                                       │                 ▼
                                             seen_urls ◄──── jobs_detail ──(>90 ngày, T2)──► archive
```

---

# Phần 0 — Setup từng bước

## Bước 1 — Kiểm tra bạn có dùng được Spark không (2 phút)

Cần đủ cả 4 điều kiện:

| Điều kiện                                      | Cách kiểm tra                                               |
| ------------------------------------------------- | ------------------------------------------------------------- |
| Gói Google AI**Pro** hoặc **Ultra** | gemini.google.com → avatar góc phải → xem gói hiện tại |
| Tài khoản Google**cá nhân**             | Không dùng được tài khoản công ty/trường học       |
| Trên 18 tuổi                                    | Theo thông tin tài khoản Google                            |
| Bật**Keep Activity**                       | myactivity.google.com/product/gemini                          |

Sau đó vào **gemini.google.com → Menu (góc trái) → tìm mục "Spark"**.

- **Thấy tab Spark** → sang Bước 2.
- **Không thấy** → Spark roll out cho AI Pro ở Mỹ trước rồi mở rộng dần sang các nước khác, nên VN có thể chưa tới lượt gói Pro. Ultra thường có sớm hơn. Nếu đang ở Pro mà chưa thấy, chờ thêm hoặc cân nhắc nâng Ultra.

> Lưu ý: Spark hiện chỉ có trên **app mobile**, **app Mac**, và **web gemini.google.com**. Riêng phần quản lý Skill thì **chỉ có trên web**, app mobile không tạo Skill được.

## Bước 2 — Bật Connected Apps (2 phút)

gemini.google.com → **Settings & help → Connected Apps**

Bật 2 mục:

- ☑ **Google Workspace** — bắt buộc. Skill cần **Gmail** (đọc job alert + gửi email tổng hợp) và **Sheets** (bộ nhớ chống trùng)
- ☑ **Google Search**

Các Connected Apps mặc định đang tắt, phải tự vào bật.

## Bước 3 — Tạo Google Sheet 3 tab (8 phút)

Đây là bước tốn công nhất nhưng làm một lần là xong. Làm chính xác, vì tên tab và tên cột sẽ được Skill gọi đúng theo chữ.

### 3.1 Tạo file

Vào sheets.google.com → tạo sheet trống → đặt tên file chính xác:

```
Job Radar Tracker
```

### 3.2 Tab 1: `seen_urls`

Đây là tab Spark **đọc** mỗi lần chạy. Giữ nó thật nhẹ.

1. Đổi tên tab mặc định "Sheet1" thành `seen_urls`
2. Ô **A1** gõ: `job_url`
3. **Xoá cột B đến Z:** click cột B, giữ Shift, click cột Z, chuột phải → *Delete columns B–Z*
4. **Xoá bớt dòng trống:** click dòng 501, Ctrl+Shift+↓ để chọn tới cuối, chuột phải → *Delete rows*

> **Tại sao phải xoá?** Ô trống vẫn tính vào hạn mức. Mỗi tab mới mặc định 1.000 dòng × 26 cột = 26.000 ô trống. Ba tab là 78.000 ô chưa có dữ liệu nào. Không chết ai, nhưng dọn thì Spark đọc nhanh hơn.

### 3.3 Tab 2: `jobs_detail`

Tab này Spark **chỉ ghi** trong task quét. Chỉ task dọn dẹp (`mode: cleanup`, sáng thứ Hai) mới đọc.

1. Tạo tab mới, đặt tên `jobs_detail`
2. Dòng 1 điền đúng 9 cột này:

| A       | B     | C       | D    | E          | F           | G      | H             | I          |
| ------- | ----- | ------- | ---- | ---------- | ----------- | ------ | ------------- | ---------- |
| job_url | title | company | city | role_group | posted_date | source | first_sent_at | alias_urls |

3. Xoá cột J trở đi

> Cột `alias_urls` chứa các URL trùng của cùng một tin (cùng JD đăng trên nhiều trang). Task dọn dẹp cần nó để xoá sạch cả nhóm URL khỏi `seen_urls`, nếu không `seen_urls` sẽ phình mãi. Thường rỗng.

### 3.4 Tab 3: `archive`

1. Tạo tab mới, đặt tên `archive`
2. Copy nguyên dòng header của `jobs_detail` sang dòng 1 (đủ 9 cột)
3. Xoá cột J trở đi

### 3.5 Kiểm tra

File `Job Radar Tracker` phải có đúng 3 tab: `seen_urls`, `jobs_detail`, `archive`. Không có tab "Sheet1" thừa.

## Bước 3b — Đăng ký Job Alert và tạo Gmail label (15 phút)

**Đây là bước quyết định chất lượng kết quả.** Skill đọc job alert từ Gmail *trước* khi duyệt web, vì email không dính captcha, là nguồn duy nhất cho TopCV, và bắt được tin LinkedIn chỉ hiện khi đăng nhập. Nếu bỏ qua bước này, skill vẫn chạy nhưng sẽ mỏng đi rất nhiều.

### 3b.1 Đăng ký nhận alert trên 5 trang

Với mỗi trang, tạo alert cho **cả 2 thành phố** (hoặc chỉ thành phố bạn theo dõi) với từ khoá rộng — để skill tự lọc, đừng lọc kỹ ở đây:

| Trang                  | Tạo alert ở đâu                                       | Từ khoá gợi ý                                                               | Tần suất |
| ---------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------- | ---------- |
| **LinkedIn**     | linkedin.com/jobs → tìm → bật chuông "Job alert"     | `Data Analyst`, `Data Engineer`, `Business Intelligence` (3 alert riêng) | Daily      |
| **ITviec**       | itviec.com → tìm → "Nhận việc làm qua email"        | `Data`                                                                        | Daily      |
| **VietnamWorks** | vietnamworks.com → tìm → "Tạo thông báo việc làm" | `Data Analyst`, `Data Engineer`                                             | Daily      |
| **TopCV**        | topcv.vn → tìm → "Nhận thông báo việc làm"        | `Data Analyst`, `Data Engineer`, `BI`                                     | Daily      |
| **Xóm Jobs**    | jobs.xomdata.com → đăng ký nhận tin                  | Category Data / Analytics                                                       | Daily      |

Nếu trang nào không có tính năng alert hoặc bạn không tìm thấy, bỏ qua — skill vẫn quét web trang đó.

### 3b.2 Tạo 5 label trong Gmail

Gmail → Settings (⚙) → **See all settings → Labels → Create new label**. Tạo **đúng tên** sau (skill tìm theo chữ):

```
ITViec Job Alerts
LinkedIn Job Alerts
VietnamWorks Job Alert
TopCV
Xom Job Alerts
```

### 3b.3 Tạo filter tự gắn label

**Làm vào ngày hôm sau.** Phải chờ nhận được email alert đầu tiên từ mỗi trang (thường trong 24h) thì Gmail mới có mẫu để tạo filter. Trong lúc chờ có thể làm tiếp Bước 4 (tạo Skill), nhưng **đừng chạy test ở Bước 6** trước khi label có email — kết quả test sẽ mỏng và bạn không phân biệt được lỗi thật với lỗi do chưa có alert.

Với mỗi email:

1. Mở email → menu ⋮ → **Filter messages like this**
2. Gmail tự điền địa chỉ người gửi vào ô *From*. Nếu tiêu đề email có mẫu cố định (ví dụ "việc làm mới cho bạn"), thêm vào ô *Subject* để lọc chính xác hơn
3. **Create filter** → tick **Apply the label** → chọn label tương ứng → tick **Also apply filter to matching conversations** → **Create filter**

Không tick "Skip the Inbox" nếu bạn vẫn muốn thấy email trong Inbox; skill không cần email nằm ở Inbox hay không.

### 3b.4 Kiểm tra

Gmail → click từng label ở cột trái → phải thấy ít nhất 1 email trong mỗi label. Label nào chưa có email thì chờ thêm 1 ngày rồi làm lại 3b.3 cho label đó.

## Bước 4 — Tạo Skill (5 phút)

**Chỉ làm được trên web**, không làm trên mobile.

1. gemini.google.com → **Menu → Spark → Skills**
2. Bấm **Create skill**
3. Mở file `SKILL.md` nằm cùng thư mục với tài liệu này, copy **toàn bộ** nội dung (kể cả khối `---` frontmatter ở đầu), dán vào
4. Lưu

Kiểm tra: quay lại trang Skills, phải thấy skill tên `vn-data-job-radar`.

> Từ bản 3, nội dung skill **chỉ nằm trong `SKILL.md`**, không chép lại vào tài liệu này nữa để tránh hai bản lệch nhau. Sửa skill thì sửa `SKILL.md` rồi dán lại vào Spark.
>
> **Cập nhật skill đã có:** Spark không đọc file trên máy hay trên git, và nhắn "dùng skill mới nhất" vào thread cũng không có tác dụng. Vào **Spark → Skills → mở skill `vn-data-job-radar` → Edit**, xoá hết nội dung cũ, dán toàn bộ `SKILL.md` mới, Lưu. **Không** tạo skill thứ hai cùng tên — Spark sẽ không biết chọn cái nào. Sau khi dán, chạy tay 1 lần và nhìn dòng `Skill: v3.4` ở cuối email: khớp số ở đầu `SKILL.md` là đã nhận bản mới.

## Bước 5 — Tạo Task đầu tiên (3 phút)

1. **Menu → Spark** → ô nhập ở giữa màn hình
2. Copy instruction **Task 1 (Hà Nội)** ở [Phần 2](#phần-2--nội-dung-task)
3. **Thay `[ĐIỀN EMAIL CỦA BẠN]` bằng email thật của bạn**
4. Gõ ký tự `/` rồi chọn `vn-data-job-radar` để gắn Skill vào task
5. **Chưa đặt lịch vội.** Cứ submit để nó chạy ngay một lần.

## Bước 6 — Test lần 1: kiểm tra nó chạy được (10 phút chờ)

Lần chạy đầu Spark sẽ hỏi vài thứ. Xử lý như sau:

| Nó hỏi gì                                | Bạn làm gì                                                                                                                                                                                                        |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Xin quyền kết nối Chrome local           | **Từ chối / bỏ qua.** Skill được thiết kế để chạy remote browser vì task sẽ chạy lúc bạn không mở máy. Cho phép Chrome local ở lần test sẽ làm kết quả test *đẹp hơn thực tế* |
| Xác nhận danh sách website sẽ truy cập | Xem qua rồi**duyệt**                                                                                                                                                                                         |
| Xin quyền đọc Gmail / Sheets             | **Cho phép**                                                                                                                                                                                                  |
| "Take control" khi gặp màn đăng nhập   | **Không đăng nhập.** Bấm bỏ qua / để nó tự ghi nhận "không truy cập được" và đi tiếp. Lý do như trên: schedule chạy lúc bạn offline, không ai đăng nhập hộ                       |
| Xác nhận trước khi gửi email           | **Duyệt**                                                                                                                                                                                                     |

Trong lúc chờ, mở **work panel** (bấm chip tiến độ ở đầu thread) để xem nó đang ở bước nào.

**Sau khi xong, kiểm tra 6 thứ:**

1. ☑ Tab `seen_urls` có URL mới, **không có URL nào chứa `?utm`, `?ref=`, `trackingId`**
2. ☑ Tab `jobs_detail` có dòng đầy đủ 9 cột (cột I `alias_urls` thường rỗng); cột `first_sent_at` dạng `2026-09-12T09:03:00+07:00`; cột `role_group` chỉ có `DA/AE/DE/BI/BA`
3. ☑ Email đã về, link "Xem tin" bấm được, công ty ưu tiên có ⭐ trước tên
4. ☑ Cuối email có khối "Báo cáo lần chạy" đủ 5 dòng, `SEEN_COUNT` lần đầu = 0 là đúng, và dòng **Link tới Google Sheet** mở đúng file `Job Radar Tracker`
5. ☑ Mở đầu email **không** ghi "đang dùng thành phố mặc định" (nếu có → Task chưa truyền `city` đúng)
6. ☑ Đọc phần "Career page lỗi" trong khối báo cáo — 16 URL career page (15 công ty,
   riêng NAB có 2 nguồn) ở
   mục 3.6 trong Skill là tra cứu sẵn, có thể đã đổi. Công ty nào báo lỗi thì mở thử
   URL đó bằng tay, thay URL đúng vào bảng trong Skill (Menu → Spark → Skills →
   `vn-data-job-radar` → Edit)

Nếu thiếu mục nào, xem [Phần 5](#phần-5--xử-lý-sự-cố). Mẫu email đúng ở [Phần 3](#phần-3--email-nhận-được-trông-thế-nào).

## Bước 7 — Test lần 2: kiểm tra chống trùng (BƯỚC QUAN TRỌNG NHẤT)

Ngay sau khi lần 1 xong, nhắn vào chính thread đó:

```
Chạy lại ngay bây giờ, mode: full.
```

**Kết quả đúng:** email trả về *"không có tin mới"*, hoặc chỉ 1–2 tin vừa mới đăng trong vài phút vừa rồi. Dòng `SEEN_COUNT` cuối email phải **bằng số URL lần 1 đã ghi** (khác 0).

**Kết quả sai:** nó gửi lại y nguyên danh sách lần 1. Nghĩa là bước đọc `seen_urls` không chạy. Nhắn vào thread:

```
Bạn vừa gửi lại tin đã gửi ở lần chạy trước. Trước khi làm bất cứ việc gì
ở lần chạy tới, hãy mở Google Sheet "Job Radar Tracker", tab "seen_urls",
đọc toàn bộ cột A vào danh sách SEEN, và loại bỏ mọi job có URL nằm trong SEEN.
Xác nhận lại cho tôi là bạn đã đọc được bao nhiêu URL từ sheet.
```

Nó phải trả lời được con số cụ thể. **Đừng sang bước tiếp theo cho tới khi test này pass** — nếu bỏ qua, bạn sẽ nhận cùng một job 4 lần mỗi ngày.

## Bước 7b — Test lần 3: chế độ quick (5 phút, tuỳ chọn)

Nhắn vào thread:

```
Chạy lại ngay bây giờ, mode: quick.
```

Kiểm tra: work panel cho thấy nó **chỉ** đọc Gmail, Xóm Jobs, LinkedIn (URL có `jobs-guest` và `f_TPR=r86400`, tối đa 3 lần mở) — không mở TopCV/ITviec/VietnamWorks, không mở career page. Nếu nó vẫn quét đủ, nhắn: *"Ở mode quick, chỉ quét Gmail + Xóm Jobs + LinkedIn theo đúng skill."*

## Bước 8 — Tạo Task thứ hai (2 phút)

Lặp lại Bước 5 với instruction **Task 2 (TP.HCM)**. Chạy tay 1 lần để chắc nó hoạt động. Không cần test chống trùng lại vì cùng một sheet.

## Bước 8b — Tạo Task dọn dẹp (3 phút)

Lặp lại Bước 5 với instruction **Task 3 (Dọn dẹp)**. Chạy tay 1 lần.

**Kết quả đúng:** một email riêng (mẫu ở Phần 3), tiêu đề `[Job Radar] Dọn dẹp tuần — SKIPPED — {dd/MM}` với lý do "chưa đủ dữ liệu để dọn" (vì `jobs_detail` mới có vài chục dòng). Sheet **không thay đổi gì**.

**Kết quả sai:** nó bắt đầu quét web hoặc đọc Gmail → nhắn: *"mode: cleanup không quét gì cả, chỉ làm mục 'Chế độ cleanup' trong skill."* Hoặc nó xoá dòng trong sheet dù chưa đủ 50 dòng → nhắn: *"Guard đầu tiên: jobs_detail dưới 50 dòng thì SKIPPED, không được xoá."*

## Bước 9 — Bật Schedule (3 phút)

Giờ mới đặt lịch. Với **mỗi** task, mở thread và nhắn 2 câu (nhắn riêng từng câu):

**Task Hà Nội:**

```
Tạo lịch: mỗi ngày lúc 08:00 giờ Việt Nam, chạy với mode: full.
```

```
Tạo lịch: mỗi ngày lúc 17:30 giờ Việt Nam, chạy với mode: quick.
```

**Task TP.HCM:** giống hệt, đổi giờ thành **09:30** và **17:50**.

**Task Dọn dẹp** (chỉ 1 câu):

```
Tạo lịch: mỗi thứ Hai lúc 07:00 giờ Việt Nam, chạy với mode: cleanup.
```

Lệch **90 phút** giữa hai task `full` để không chạy chồng nhau — mode `full` giờ quét
5 nguồn chính **cộng thêm 16 URL career page**, mỗi career page có thể lật nhiều trang
(trần 2 phút/công ty), nên một lần chạy `full` có thể mất 30–45 phút, không còn là
~15 phút như trước. Hai task `quick` (17:30/17:50) không quét career page nên nhẹ hơn
nhiều, giữ nguyên cách nhau 20 phút là đủ. Task dọn dẹp chạy **07:00, trước cả hai task
quét sáng**, để không có task nào đọc/ghi sheet trong lúc nó xoá dòng.

> Vì dùng remote browser nên không cần máy bật đúng giờ — chọn giờ nào cũng được. 08:00 để email có trước giờ làm.

> Task dọn dẹp gửi email riêng mỗi thứ Hai, kể cả khi không có gì để dọn (`SKIPPED`). Trong ~3 tháng đầu bạn sẽ chỉ thấy `SKIPPED` — đó là bình thường.

**Kiểm tra:** mở work panel → mục **Schedules** → phải thấy 2 lịch cho mỗi task quét + 1 lịch cho task dọn dẹp, tổng **5 lịch**.

## Bước 10 — Tuần đầu: theo dõi và tinh chỉnh

Đừng để chạy tự động rồi quên. Tuần đầu mỗi sáng kiểm nhanh:

| Dấu hiệu                                                        | Nghĩa là                                             | Sửa bằng cách                                                                                                |
| ----------------------------------------------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| Email không về                                                  | Lịch không chạy, hoặc task đang chờ bạn confirm | Mở Spark → Tasks, xem có task nào đang pending không                                                      |
| Nhận lại tin cũ                                                | Chống trùng hỏng                                    | Quay lại Bước 7                                                                                              |
| `SEEN_COUNT` đột nhiên về 0                                 | Không đọc được sheet                             | Kiểm tra tên file/tab, quyền Workspace                                                                       |
| Dòng "nguồn không truy cập được" luôn có TopCV            | Bình thường với remote browser                     | Kiểm tra label Gmail `TopCV` có email đều không (Bước 3b.4)                                          |
| Dòng "nguồn không truy cập được" có LinkedIn nhiều ngày   | IP remote browser bị LinkedIn chặn                 | Xem mục "LinkedIn không truy cập được" ở Phần 5; Gmail `LinkedIn Job Alerts` vẫn bù được               |
| Quá nhiều tin rác                                              | Bộ lọc lỏng                                         | Nhắn:`Loại hết tin từ công ty outsourcing và headhunt, chỉ giữ product company, ngân hàng, fintech` |
| Quá ít tin                                                      | Alert Gmail chưa về hoặc filter chưa gắn label    | Xem Bước 3b.4                                                                                                 |
| Toàn tin senior                                                  | Chưa lọc YOE                                         | Nhắn:`Chỉ giữ tin yêu cầu dưới 3 năm kinh nghiệm`                                                    |
| Cuối email không có dòng `Skill: v3.3` hoặc số khác          | Spark đang chạy bản skill cũ                       | Dán lại `SKILL.md` vào skill đã có (Bước 4), không tạo skill mới                                        |
| Thứ Hai không có email dọn dẹp                               | Task 3 chưa có lịch hoặc bị pause                 | Work panel → Schedules của Task 3                                                                             |
| Cùng 1 tin hiện 2–3 dòng                                      | Gộp theo`company\|title` chưa chạy                 | Nhắn:`Gộp các tin cùng công ty và cùng tiêu đề thành 1 dòng theo Bước 5 của skill`             |

## Bước 11 — Sau 1 tháng (tuỳ chọn)

Task dọn dẹp chạy mỗi thứ Hai, đẩy dòng cũ hơn 90 ngày sang tab `archive`. Bạn không phải làm gì ngoài liếc qua email báo cáo: khi kết quả chuyển từ `SKIPPED` sang `DONE` lần đầu, mở sheet kiểm tra một lần cho chắc.

Nhưng đừng xoá `archive`. Sau vài tháng đó là dataset dọc về thị trường tuyển dụng data ở VN — mức lương theo thời gian, công ty nào tuyển đều, stack nào đang lên. Với người làm phân tích, thứ đó có khi giá trị hơn cái email hàng ngày.

---

# Phần 1 — Luồng chạy của Skill

Nội dung skill nằm trong file **`SKILL.md`** cùng thư mục. Không chép lại ở đây.

Tóm tắt luồng để bạn đối chiếu khi đọc work panel. Task quét (`mode: full` / `quick`) đi từ Bước 1 → 8. Task dọn dẹp (`mode: cleanup`) **bỏ qua toàn bộ** 8 bước đó, chỉ làm mục cuối bảng.

| Bước trong skill    | Việc làm                                                                                                                                                                               |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1                     | Đọc`seen_urls` cột A → SEEN, đếm `SEEN_COUNT`                                                                                                                                  |
| 2                     | Đọc 5 Gmail label, trích URL job                                                                                                                                                      |
| 3                     | Quét web bằng URL lọc sẵn: Xóm Jobs (4 URL) → LinkedIn endpoint guest (`full` 10 URL / `quick` 3 URL) → TopCV (thử 1 lần) → ITviec (3 URL) → VietnamWorks → career page ưu tiên (mode`quick`: chỉ Xóm Jobs + LinkedIn) |
| 4                     | Từ khoá 5 nhóm vị trí                                                                                                                                                               |
| 5                     | Lọc 72h / thành phố / không trong SEEN; gộp trùng theo`company\|title`; lọc BA                                                                                                   |
| 6                     | Mở JD từng tin lấy Lương / YOE / Stack (trần: full 40 tin, quick 20 tin); tin LinkedIn mở qua `jobs-guest/jobs/api/jobPosting/{id}`                                              |
| 7                     | Ghi`jobs_detail` trước, rồi `seen_urls` (cả URL chính lẫn `alias_urls`)                                                                                                      |
| 8                     | Gửi email HTML theo khung cố định, cuối email có link tới sheet                                                                                                                   |
| **`cleanup`** | Task riêng, T2 07:00: đọc`jobs_detail`, archive dòng >90 ngày (guard: ≥50 dòng, tối đa 30% mỗi lần), xoá URL tương ứng khỏi `seen_urls`, gửi email báo cáo riêng |

---

# Phần 2 — Nội dung Task

## Task 1 — Hà Nội

```
Dùng skill /vn-data-job-radar để theo dõi thị trường việc làm ngành dữ liệu cho tôi.

Tham số:
- city: Hà Nội
- mode: full
- Email gửi về: [ĐIỀN EMAIL CỦA BẠN]

Làm đúng theo skill, đặc biệt:
- Bước 1 đọc tab "seen_urls" của Google Sheet "Job Radar Tracker" TRƯỚC mọi việc khác
- Bước 7 ghi sheet TRƯỚC khi gửi email
- Dùng remote browser, không cần Chrome local. Gặp captcha hay tường đăng nhập
  thì ghi nhận và đi tiếp, không chờ tôi.

Chưa đặt lịch. Chạy ngay một lần bây giờ để tôi kiểm tra.
```

## Task 2 — TP.HCM

```
Dùng skill /vn-data-job-radar để theo dõi thị trường việc làm ngành dữ liệu cho tôi.

Tham số:
- city: TP.HCM (chấp nhận cả tin ghi "Hồ Chí Minh", "HCMC", "Thủ Đức")
- mode: full
- Email gửi về: [ĐIỀN EMAIL CỦA BẠN]

Làm đúng theo skill, đặc biệt:
- Bước 1 đọc tab "seen_urls" của Google Sheet "Job Radar Tracker" TRƯỚC mọi việc khác
- Bước 7 ghi sheet TRƯỚC khi gửi email
- Dùng remote browser, không cần Chrome local. Gặp captcha hay tường đăng nhập
  thì ghi nhận và đi tiếp, không chờ tôi.

Chưa đặt lịch. Chạy ngay một lần bây giờ để tôi kiểm tra.
```

> Giá trị `city` trong Task 1 và 2 phải là đúng `Hà Nội` hoặc `TP.HCM` — đây là chuỗi skill ghi vào cột `city` của sheet. Đừng viết "TP. Hồ Chí Minh" hay "HCM". Task 3 không có `city`.

## Task 3 — Dọn dẹp hàng tuần

```
Dùng skill /vn-data-job-radar.

Tham số:
- mode: cleanup
- Email gửi về: [ĐIỀN EMAIL CỦA BẠN]

Không quét web, không đọc Gmail. Chỉ làm mục "Chế độ cleanup" trong skill:
đọc Google Sheet "Job Radar Tracker" tab "jobs_detail", chuyển các dòng có
first_sent_at cũ hơn 90 ngày sang tab "archive", xoá URL tương ứng khỏi
"seen_urls", rồi gửi một email báo cáo ngắn. Tuân thủ đủ các guard an toàn
trong skill — nếu không thoả guard thì không xoá gì và báo SKIPPED hoặc BLOCKED.

Chưa đặt lịch. Chạy ngay một lần bây giờ để tôi kiểm tra.
```

## Bảng lịch

| Task      | Sáng (`mode: full`) | Chiều (`mode: quick`) | Thứ Hai (`mode: cleanup`) |
| --------- | ---------------------- | ------------------------ | ---------------------------- |
| Hà Nội  | 08:00                  | 17:30                    | —                           |
| TP.HCM    | 09:30                  | 17:50                    | —                           |
| Dọn dẹp | —                     | —                       | 07:00                        |

---

# Phần 3 — Email nhận được trông thế nào

Dùng để đối chiếu khi test ở Bước 6, 7, 8b.

## Email quét (Task Hà Nội / TP.HCM)

**Tiêu đề:** `[Job Radar] Hà Nội — 12 tin mới — 15/09 08:14`

> Sáng nay có 12 tin mới, nhiều nhất là Data Engineer (5 tin). Đáng chú ý: Techcombank mở cùng lúc 3 vị trí Data Platform, stack Spark + Airflow.
>
> **Data Analyst (2 tin)**
>
> | Vị trí            | Công ty | Lương       | YOE | Stack chính                  | Ngày đăng | Link    |
> | ------------------- | -------- | ------------- | --- | ----------------------------- | ------------ | ------- |
> | Senior Data Analyst | ⭐ MoMo  | Thoả thuận  | 3   | SQL, Python, Looker, BigQuery | 2026-09-14   | Xem tin |
> | Product Analyst     | Base.vn  | 20–28 triệu | 2   | SQL, Metabase, Excel          | 2026-09-13   | Xem tin |
>
> **Data Engineer (5 tin)**
>
> *(bảng tương tự)*
>
> **Business Analyst (2 tin)**
>
> | Vị trí                            | Công ty | ... |
> | ----------------------------------- | -------- | --- |
> | Business Analyst — BA thiên data  | VNPAY    | ... |
> | IT Business Analyst — IT BA thuần | FPT IS   | ... |
>
> ---
>
> **Báo cáo lần chạy**
>
> - Nguồn không truy cập được: TopCV (captcha)
> - Nguồn chạm trần: ITviec (30 tin, còn tin chưa quét)
> - Career page lỗi: không có
> - Đã mở 12 JD, bỏ qua 0 tin vì chạm trần
> - SEEN_COUNT: 2.310
> - Skill: v3.4
>
> 📋 Link tới Google Sheet: [Job Radar Tracker](#) — tab `jobs_detail` có đủ mọi tin từ trước tới nay, tab `archive` có tin cũ hơn 90 ngày.

Những gì **không** xuất hiện: section không có tin (bỏ hẳn), cột ⭐ riêng (⭐ nằm trước tên công ty), URL trần, lương suy đoán, màu sắc hay CSS. Khối báo cáo luôn đủ 5 dòng — dòng nào không có gì thì ghi "không có".

**Khi không có tin mới:** tiêu đề `[Job Radar] Hà Nội — không có tin mới`, thân email 1 dòng rồi vẫn có khối báo cáo và link sheet như trên.

## Email dọn dẹp (Task 3, thứ Hai)

**Tiêu đề:** `[Job Radar] Dọn dẹp tuần — DONE — 15/12`

> Kết quả: **DONE**
> Đã archive: 213 dòng
> jobs_detail: 1.847 → 1.634 dòng
> seen_urls: 2.310 → 2.041 URL
> archive: tổng 213 dòng
> Dòng cũ nhất còn lại trong jobs_detail: 2026-09-16
>
> 📋 Link tới Google Sheet: [Job Radar Tracker](#)

Năm trạng thái có thể gặp:

| Tiêu đề  | Khi nào                                                                                                         | Bạn cần làm gì                                                                   |
| ----------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `SKIPPED` | `jobs_detail` < 50 dòng, hoặc không dòng nào quá 90 ngày. **~3 tháng đầu sẽ toàn thế này** | Không                                                                               |
| `DONE`    | Dọn thành công, không còn dòng nào chờ                                                                   | Lần đầu thấy DONE thì mở sheet liếc qua cho chắc                             |
| `PARTIAL` | Số dòng cần dọn vượt trần 30% mỗi lần → dọn 30% cũ nhất, phần còn lại để tuần sau             | Không. Vài tuần nữa nó tự về`DONE`                                          |
| `BLOCKED` | Số dòng cần dọn > 80% tổng — bất thường, thường là`first_sent_at` hỏng định dạng               | Mở sheet kiểm tra cột`first_sent_at` (Phần 5)                                  |
| `FAILED`  | Append vào`archive` xong nhưng đếm không khớp → dừng trước khi xoá                                  | Xoá dòng vừa append trong`archive`, chạy lại (Phần 5). Không mất dữ liệu |

Email dọn dẹp **luôn gửi**, kể cả `SKIPPED`, để bạn biết task còn sống.

---

# Phần 4 — Phương án 10 task

Chọn phương án này nếu muốn mỗi title một email riêng để lọc và lưu trữ độc lập.

**Đổi lại:** Spark phải mở TopCV / ITviec / LinkedIn 5 lần cho mỗi thành phố thay vì 1 lần — cùng một trang, quét lặp 5 lượt. Tốn quota khoảng gấp 5, và 20 email mỗi ngày. Ngoài ra bạn chỉ được chạy tối đa 15 task đồng thời, và một schedule sẽ không chạy nếu đã có 15 task đang chạy — nên phải đặt giờ so le.

**Lưu ý về giờ so le dưới đây:** mỗi task trong 10 task này vẫn quét đủ 16 URL career
page (chỉ lọc theo nhóm vị trí ở bước sau, không giảm số career page phải mở), nên
thời lượng một task riêng lẻ không nhẹ hơn task gộp bao nhiêu. Khoảng cách 5 phút ở
bảng dưới **rất có thể không đủ** — cân nhắc giãn ra 15–20 phút mỗi task, hoặc chấp
nhận vài task chạy chồng nhau (vẫn an toàn vì cùng ghi sheet theo đúng thứ tự
`jobs_detail` trước `seen_urls`, chỉ là tốn thời gian hơn dự kiến).

**Template** — thay 2 chỗ trong `{}`:

```
Dùng skill /vn-data-job-radar.

Tham số:
- city: {THÀNH PHỐ}
- mode: full
- Email gửi về: [ĐIỀN EMAIL CỦA BẠN]

Chỉ theo dõi nhóm vị trí {NHÓM VỊ TRÍ} — bỏ qua hoàn toàn 4 nhóm còn lại.
Tiêu đề email: [Job Radar] {NHÓM VỊ TRÍ} — {THÀNH PHỐ} — {N} tin mới — {dd/MM HH:mm}
Vì chỉ một nhóm vị trí, thay cấu trúc 5 section bằng một bảng duy nhất,
sắp xếp theo mức lương giảm dần (tin "Thoả thuận" và "không rõ" xếp cuối).
```

Dọn dẹp vẫn dùng **Task 3** riêng như Phần 2, lịch thứ Hai 07:00. Không nhét dọn dẹp vào 10 task quét.

**Giờ chạy so le:**

| #  | Thành phố | Nhóm vị trí        | Sáng | Chiều |
| -- | ----------- | --------------------- | ----- | ------ |
| 1  | Hà Nội    | Data Analyst          | 08:00 | 17:00  |
| 2  | Hà Nội    | Analytics Engineer    | 08:05 | 17:05  |
| 3  | Hà Nội    | Data Engineer         | 08:10 | 17:10  |
| 4  | Hà Nội    | Business Intelligence | 08:15 | 17:15  |
| 5  | Hà Nội    | Business Analyst      | 08:20 | 17:20  |
| 6  | TP.HCM      | Data Analyst          | 08:25 | 17:25  |
| 7  | TP.HCM      | Analytics Engineer    | 08:30 | 17:30  |
| 8  | TP.HCM      | Data Engineer         | 08:35 | 17:35  |
| 9  | TP.HCM      | Business Intelligence | 08:40 | 17:40  |
| 10 | TP.HCM      | Business Analyst      | 08:45 | 17:45  |

Cả 10 task dùng **chung một** Google Sheet `Job Radar Tracker`. Không tạo 10 sheet.

---

# Phần 5 — Xử lý sự cố

## Spark hỏi xác nhận mỗi lần gửi email

Đây là hành vi mặc định không tắt được — Gemini luôn yêu cầu bạn review và xác nhận trước khi gửi thông tin liên lạc. Ba cách xử lý:

| Cách                                   | Ưu                    | Nhược                      |
| --------------------------------------- | ---------------------- | ---------------------------- |
| Cứ bấm confirm                        | Đúng như thiết kế | Phải mở app 2 lần/ngày   |
| Đổi sang**tạo Gmail draft**    | Ít ma sát hơn       | Vẫn phải mở Gmail         |
| Đổi sang**ghi vào Google Doc** | Hoàn toàn tự động | Không có thông báo đẩy |

Chọn cách 3 thì sửa **Bước 8** trong `SKILL.md`: thay "gửi email" bằng

```
Chèn một section mới lên ĐẦU Google Doc tên "Job Radar Log",
tiêu đề section là "{Thành phố} — {dd/MM HH:mm}".
```

Rồi ghim Doc đó lên màn hình chính điện thoại.

## TopCV luôn "không truy cập được"

**Đây là bình thường** với remote browser — TopCV chặn bot bằng Cloudflare, và Xóm Jobs không aggregate TopCV. Skill được thiết kế để chỉ thử TopCV một lần rồi đi tiếp; tin TopCV đến từ Gmail label `TopCV`.

Kiểm tra: Gmail → label `TopCV` → có email trong 3 ngày gần nhất không?

- **Có** → không cần làm gì, dòng "không truy cập được" chỉ là thông tin.
- **Không** → alert chưa tạo hoặc filter chưa gắn label. Làm lại Bước 3b.

**Nếu vẫn muốn quét web TopCV đầy đủ:** dùng Spark trên Chrome desktop với auto browse, đăng nhập sẵn TopCV. Đánh đổi: máy phải bật và Chrome phải chạy vào giờ schedule; tắt máy giữa chừng thì Spark rơi về remote browser. Với đa số người dùng, Gmail alert là đủ và đỡ phiền hơn.

## LinkedIn "không truy cập được"

**Không còn là bình thường** từ bản 3.3. Skill không mở `linkedin.com/jobs/search` (tường đăng nhập) mà mở endpoint guest:

```
https://www.linkedin.com/jobs-guest/jobs/api/seeMoreJobPostings/search?keywords=Data%20Analyst&location=Hanoi%2C%20Vietnam&f_TPR=r259200&start=0
```

Endpoint này là thứ chính trang LinkedIn gọi ngầm để tải thêm tin cho khách chưa đăng nhập, nên nó không có tường đăng nhập và không cần JavaScript. Nếu nó vẫn báo "không truy cập được":

1. **Dán URL trên vào trình duyệt của bạn.** Thấy danh sách tin → endpoint sống, vấn đề nằm ở IP của remote browser bị LinkedIn chặn. Không thấy gì / bị chuyển sang trang đăng nhập → LinkedIn đã đổi endpoint, cần cập nhật skill.
2. Nếu là IP bị chặn: thường tự hết sau vài giờ vì remote browser đổi IP. Trong lúc đó Gmail `LinkedIn Job Alerts` vẫn bù được — kiểm tra label có email đều không.
3. Nếu bị chặn liên tục nhiều ngày: chạy Spark trên Chrome desktop (IP nhà) hoặc tăng khoảng cách giữa hai lần chạy.

**Lưu ý:** cùng cơ chế này, JD của tin LinkedIn được đọc qua `linkedin.com/jobs-guest/jobs/api/jobPosting/{id}`. Nếu email có nhiều tin LinkedIn mà Lương/YOE/Stack toàn `không rõ`, nhắn vào thread: *"Với tin LinkedIn, mở JD qua endpoint jobs-guest/jobs/api/jobPosting/{id} theo đúng skill, không mở jobs/view."*

## Email không về

Kiểm theo thứ tự:

1. **Spark → Tasks** — task có đang ở trạng thái chờ bạn confirm không?
2. **Work panel → Schedules** — lịch có đang bị Pause không?
3. Có đang chạy quá 15 task đồng thời không? Nếu có, schedule sẽ bị bỏ qua.
4. Spark có đang bị tắt trong Settings không? Khi tắt, mọi schedule ngừng chạy.

## Nhận lại tin đã gửi

Ba nguyên nhân, kiểm theo thứ tự:

**1. Bước đọc `seen_urls` không chạy.** Dấu hiệu: `SEEN_COUNT` cuối email = 0. Nhắn vào thread:

```
Bạn vừa gửi lại tin đã gửi trước đó. Trước khi làm bất cứ việc gì ở lần chạy tới,
mở Google Sheet "Job Radar Tracker", tab "seen_urls", đọc toàn bộ cột A vào
danh sách SEEN, và loại bỏ mọi job có URL nằm trong SEEN.
Báo lại cho tôi số URL đọc được.
```

**2. URL chưa được chuẩn hoá.** Dấu hiệu: `SEEN_COUNT` > 0 nhưng vẫn trùng; cột A của `seen_urls` có URL chứa `?utm`, `&ref=`, `trackingId`, hoặc cùng job LinkedIn xuất hiện 2 dòng khác slug. Nhắn:

```
Áp dụng đúng Bước 6 của skill: bỏ toàn bộ query string, bỏ www., bỏ dấu / cuối.
Với LinkedIn, chuẩn hoá về https://linkedin.com/jobs/view/{id}.
```

**3. Cùng tin nhưng khác URL thật** (đăng trên nhiều trang, hoặc headhunter đăng lại). Đây không phải lỗi chống trùng — sheet không thể biết. Skill xử lý bằng khoá phụ `company|title` ở Bước 5, nhưng chỉ trong cùng một lần chạy. Nếu thấy nhiều, nhắn:

```
Loại hết tin từ công ty outsourcing và headhunt, chỉ giữ product company,
ngân hàng, fintech.
```

## Ngày đăng không đáng tin

Nhiều nhà tuyển dụng VN "làm mới" tin cũ để đẩy lên đầu, nên tin hiện "đăng hôm nay" có thể đã tồn tại từ tháng trước.

Sheet chống trùng xử lý được phần lớn: một khi URL đã vào `seen_urls` thì dù tin được làm mới bao nhiêu lần cũng không gửi lại. Đây là lý do nên để cửa sổ dọn dẹp tới 90 ngày chứ không phải 7 ngày.

## Email dọn dẹp báo BLOCKED hoặc FAILED

**BLOCKED** — guard 80%: số dòng cần archive vượt 80% tổng `jobs_detail`. Đây là mức bất thường, gần như luôn do dữ liệu hỏng chứ không phải do tồn đọng: mở sheet kiểm tra cột `first_sent_at` xem có dòng nào sai định dạng hoặc sai năm không, sửa về ISO 8601 rồi chạy lại task bằng tay.

**PARTIAL** — không phải lỗi, không cần làm gì. Mỗi lần dọn task chỉ archive tối đa 30% số dòng hiện có, để một lần chạy không xoá quá nhiều. Lần đầu có dữ liệu đủ 90 ngày thường rơi vào trạng thái này vài tuần liền — số "còn chờ tuần sau" trong email sẽ giảm dần rồi về 0.

**FAILED** — đã append vào `archive` nhưng số dòng không khớp, task dừng trước khi xoá. Sheet đang ở trạng thái: `archive` có thể có dòng trùng, `jobs_detail` và `seen_urls` **chưa mất gì**. Mở `archive`, xoá các dòng vừa append (nhìn theo `first_sent_at`), rồi chạy lại task bằng tay.

Cả hai trường hợp đều **không mất dữ liệu** — thiết kế là "append trước, xoá sau".

## Giới hạn dung lượng Sheet

Một spreadsheet chứa tối đa 10 triệu ô tính gộp trên tất cả các tab (Google đang nâng lên 20 triệu qua chương trình beta).

Với 9 cột và giả sử 100 job mới mỗi ngày, bạn dùng khoảng 329.000 ô mỗi năm — chạm trần sau khoảng 30 năm. Hạn mức này không phải thứ đáng lo.

Thứ hỏng trước là **khả năng đọc của Spark**: sau 6 tháng không dọn dẹp, nó phải nhét ~9.000 dòng vào context hai lần mỗi ngày, sẽ chậm rồi bắt đầu cắt bớt. Bản thân Sheets cũng chậm rõ từ khoảng 10.000 dòng.

Cấu trúc 3 tab cộng với dọn dẹp 90 ngày giải quyết việc này: tab đọc chỉ có 1 cột thay vì 8, và số dòng đứng yên ở mức vài nghìn vĩnh viễn.

## Câu lệnh tinh chỉnh thường dùng

Nhắn thẳng vào task thread, không cần tạo task mới:

```
Loại hết tin từ công ty outsourcing và headhunt, chỉ giữ product company,
ngân hàng, fintech.
```

```
Chỉ giữ tin yêu cầu dưới 3 năm kinh nghiệm.
```

```
Thêm một cột "Độ phù hợp" chấm 1–5 sao, dựa trên CV tôi đã upload.
```

```
Bỏ nhóm Business Analyst, quá nhiều nhiễu.
```

```
Thêm nguồn: quét thêm các group Facebook về tuyển dụng data ở Việt Nam.
```

```
Từ giờ gộp email Hà Nội và TP.HCM thành một, chia section theo thành phố.
```

Tinh chỉnh áp dụng lâu dài thì nên sửa thẳng vào `SKILL.md` và dán lại vào Spark, thay vì nhắn vào thread — thread có thể quên sau nhiều lần chạy.

## Lưu ý an toàn

Spark là tính năng thử nghiệm, giai đoạn đầu. Vài điều nên biết:

- **Không gõ thông tin nhạy cảm vào task thread** — mật khẩu, thông tin thanh toán. Nếu cần đăng nhập, dùng "Take control" rồi nhập trực tiếp trên trang web.
- **Prompt injection là rủi ro thật.** Một trang tuyển dụng hoặc một email alert có thể chứa chỉ dẫn ẩn mà bạn không thấy nhưng agent đọc được. Vì task này chỉ đọc web/email và ghi vào một sheet riêng nên rủi ro thấp, nhưng vẫn nên liếc qua email trước khi bấm link.
- **Nếu schedule chạy lúc bạn offline**, bạn không kịp dừng nếu nó làm gì đó ngoài ý muốn. Task dọn dẹp (xoá dòng trong sheet) đã có guard và email báo cáo riêng, nhưng vẫn nên liếc qua sheet vào thứ Hai đầu tiên nó báo `DONE`.
- Xoá dữ liệu remote browser định kỳ: **Settings → Gemini Spark Settings → Delete remote browser data**.
