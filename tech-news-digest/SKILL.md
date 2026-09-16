---
name: tech-news-digest
description: Quét các nguồn tin công nghệ (Hacker News, Reddit, Dev.to, Medium) hàng ngày, phân loại theo chủ đề (DE, AI, BE, CD, CR, DA), chấm điểm lọc bài hay, gửi email digest. Không cần setup Sheet hay Gmail label — chỉ cần tạo skill và chạy.
---
# Tech News Digest

**Phiên bản skill: 1.0.** Luôn ghi số này vào dòng `Skill:` trong khối báo cáo cuối email.

## Mục tiêu

Quét các nguồn tin công nghệ, lọc bài hay **trong 24 giờ gần nhất**, phân loại theo
chủ đề, rồi gửi một email digest tổng hợp.

Skill này **không lưu trạng thái** giữa các lần chạy — không cần Google Sheet, không
cần Gmail label. Mỗi lần chạy là độc lập, quét từ đầu. Có thể trùng bài giữa các
ngày nếu bài vẫn nằm trên trang — chấp nhận được vì digest hàng ngày, user tự nhận ra.

## Tham số đầu vào

| Tham số | Giá trị hợp lệ | Mặc định | Ghi chú |
| ------- | --------------- | -------- | ------- |
| Không có tham số | — | — | Skill chạy với cấu hình cố định |

Không hỏi lại user — skill chạy tự động, không có ai trả lời.

## Quy tắc tự chủ — KHÔNG BAO GIỜ dừng chờ user

Skill này chạy theo lịch lúc user offline. Vì vậy, trong suốt task:

- **Không hỏi bất kỳ câu nào** — không hỏi xác nhận, không hỏi "có tiếp tục không"
- **Không yêu cầu "take control"** / không đề nghị user đăng nhập hộ. Gặp trang đăng
  nhập, captcha → ghi nguồn đó là "không truy cập được" và đi tiếp ngay
- **Không tự đoán hay tự tìm domain mới.** Chỉ mở đúng những domain đã liệt kê sẵn:
  `hn.algolia.com`, `reddit.com`, `dev.to`, `medium.com`, và `google.com` (gửi email).
  Danh sách cố định — không phát sinh domain lạ
- **Không dùng Chrome local** — chỉ remote browser
- Gặp lỗi ở một nguồn → ghi nhận, đi tiếp nguồn sau

Những thứ platform bắt buộc xác nhận (gửi email, duyệt danh sách site) nằm ngoài
tầm skill — cứ làm đúng bước và để hệ thống hỏi.

## Bước 1 — Quét Hacker News (Algolia API)

Mở URL sau trong remote browser, đọc JSON response:

```
https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=50
```

- Trả JSON, không cần auth
- Mỗi hit có: `title`, `url`, `points`, `num_comments`, `created_at`, `objectID`
- **Chỉ giữ bài có `points ≥ 50`**
- URL bài: lấy trường `url`. Nếu `url` rỗng (Ask HN, Show HN), dùng
  `https://news.ycombinator.com/item?id={objectID}`

**Trần:** 1 lần mở, tối đa 50 bài.

## Bước 2 — Quét Reddit (RSS feeds)

Mở lần lượt các URL sau, đọc RSS XML:

```
https://www.reddit.com/r/dataengineering/top/.rss?t=day&limit=15
https://www.reddit.com/r/programming/top/.rss?t=day&limit=15
https://www.reddit.com/r/devops/top/.rss?t=day&limit=10
https://www.reddit.com/r/MachineLearning/top/.rss?t=day&limit=10
https://www.reddit.com/r/ExperiencedDevs/top/.rss?t=day&limit=10
```

- Mỗi `<entry>` có: `<title>`, `<link href="...">`, `<updated>`, `<content>`
- **Chờ 3 giây giữa mỗi feed** để tránh rate-limit

**Trần:** 5 lần mở, tối đa 60 bài. Feed nào lỗi → ghi nhận, đi tiếp.

## Bước 3 — Quét Dev.to (Public API)

Mở lần lượt các URL sau, đọc JSON:

```
https://dev.to/api/articles?top=1&tag=dataengineering&per_page=10
https://dev.to/api/articles?top=1&tag=database&per_page=10
https://dev.to/api/articles?top=1&tag=devops&per_page=10
https://dev.to/api/articles?top=1&tag=machinelearning&per_page=10
https://dev.to/api/articles?top=1&tag=career&per_page=10
https://dev.to/api/articles?top=1&tag=python&per_page=10
```

- Mỗi article có: `title`, `url`, `positive_reactions_count`, `published_at`, `tag_list`
- **Chờ 2 giây giữa mỗi request**

**Trần:** 6 lần mở, tối đa 60 bài.

## Bước 4 — Quét Medium (remote browser)

Mở lần lượt các URL sau bằng remote browser:

```
https://medium.com/tag/data-engineering/recommended
https://medium.com/tag/data-science/recommended
https://medium.com/tag/system-design/recommended
https://medium.com/tag/devops/recommended
```

Mỗi trang:
1. Chờ trang tải xong (JavaScript render)
2. Đọc danh sách bài: tiêu đề, author, link bài
3. **Không** cuộn thêm hoặc bấm "Load more"

**Trần:** 4 lần mở, tối đa 2 phút mỗi trang. Captcha/paywall/trang trống → ghi
"không truy cập được", đi tiếp.

### Trần chung cho mỗi nguồn

Chạm bất kỳ ngưỡng nào thì dừng nguồn đó, đi tiếp:
- Tối đa số lần mở URL đã ghi ở từng mục
- Hoặc tối đa **3 phút** cho một nguồn

## Bước 5 — Phân loại chủ đề & Chấm điểm

### 5.1 Phân loại chủ đề

Với mỗi bài, so khớp tiêu đề (case-insensitive) với bảng từ khoá:

**Data Engineering (`DE`)**
`dbt`, `airflow`, `spark`, `kafka`, `data pipeline`, `ETL`, `ELT`, `data warehouse`,
`lakehouse`, `iceberg`, `delta lake`, `flink`, `dagster`, `prefect`, `bigquery`,
`snowflake`, `redshift`, `clickhouse`, `data modeling`, `data lake`, `data mesh`,
`data platform`, `data infrastructure`, `streaming`, `batch processing`, `fivetran`,
`airbyte`, `meltano`, `trino`, `presto`, `parquet`, `avro`

**AI / ML / LLM (`AI`)**
`AI`, `ML`, `LLM`, `GPT`, `Claude`, `Gemini`, `transformer`, `fine-tuning`, `fine tuning`,
`RAG`, `vector database`, `embedding`, `neural`, `deep learning`, `machine learning`,
`prompt engineering`, `agent`, `diffusion`, `RLHF`, `LoRA`, `inference`, `model training`,
`computer vision`, `NLP`, `natural language`, `generative AI`, `gen AI`, `OpenAI`,
`Anthropic`, `Mistral`, `Llama`, `chatbot`, `copilot`

**Backend / System Design (`BE`)**
`system design`, `distributed system`, `microservice`, `API design`, `database`,
`PostgreSQL`, `Postgres`, `MySQL`, `Redis`, `message queue`, `gRPC`, `REST`,
`architecture`, `scalability`, `concurrency`, `load balancing`, `caching`, `CAP theorem`,
`event driven`, `event-driven`, `CQRS`, `GraphQL`, `websocket`, `rate limiting`,
`circuit breaker`, `consensus`, `Raft`, `Paxos`

**Cloud / DevOps (`CD`)**
`AWS`, `GCP`, `Azure`, `kubernetes`, `k8s`, `docker`, `terraform`, `CI/CD`, `CICD`,
`monitoring`, `observability`, `SRE`, `infrastructure`, `deployment`, `container`,
`serverless`, `lambda`, `cloud function`, `helm`, `ArgoCD`, `Prometheus`, `Grafana`,
`Datadog`, `EKS`, `GKE`, `IaC`, `platform engineering`

**Career & Leadership (`CR`)**
`career`, `salary`, `interview`, `hiring`, `management`, `leadership`, `promotion`,
`team`, `culture`, `burnout`, `remote work`, `negotiation`, `layoff`, `job market`,
`resume`, `side project`, `senior engineer`, `staff engineer`, `principal engineer`,
`tech lead`, `engineering manager`, `1:1`, `mentoring`

**Data Analytics (`DA`)**
`analytics`, `dashboard`, `visualization`, `metrics`, `KPI`, `A/B testing`,
`experimentation`, `product analytics`, `SQL`, `Tableau`, `Looker`, `Power BI`,
`Metabase`, `Superset`, `reporting`, `business intelligence`, `BI`, `dbt metrics`,
`semantic layer`

**Khi một bài khớp nhiều chủ đề:** đếm số từ khoá khớp mỗi chủ đề, chủ đề nhiều
nhất thắng. Hoà → ưu tiên: **DE → AI → BE → CD → CR → DA**.

**Không khớp chủ đề nào:** gán `topic = "BE"` (mặc định cho bài kỹ thuật chung).

### 5.2 Chấm điểm

```
score = base_score + bonus
```

**Base score:**

| Nguồn  | Điều kiện            | Base |
| ------ | -------------------- | ---- |
| HN     | points ≥ 200         | 5    |
| HN     | points 100–199       | 4    |
| HN     | points 50–99         | 3    |
| Reddit | bài top (đã lọc sẵn) | 3    |
| Dev.to | reactions ≥ 50       | 3    |
| Dev.to | reactions < 50       | 2    |
| Medium | trên recommended     | 3    |

**Bonus:**

| Bonus | Điều kiện                                        |
| ----- | ------------------------------------------------ |
| +1    | Bài xuất hiện ở ≥ 2 nguồn khác nhau             |
| +1    | Chủ đề `DE` hoặc `AI` (ưu tiên cá nhân của user)|

**Phân loại:**
- **Top Picks**: score ≥ 4 → hiện đầu email đầy đủ thông tin
- **Bài thường**: score < 4 → hiện trong section theo chủ đề
- **Bài khác**: không khớp rõ chủ đề hoặc score ≤ 1 → section cuối, chỉ link

### 5.3 Gộp bài trùng

Nếu 2 bài có cùng URL (sau khi bỏ fragment, query tracking, lowercase host) → giữ
bài có score cao hơn, cộng bonus +1.

## Bước 6 — Gửi email digest

Gửi tới email của user. Dùng HTML, không dùng markdown thô.

**Tiêu đề:** `[Tech Digest] {dd/MM} — {N} bài đáng đọc`

**Thân email:**

```html
<p>{Mở đầu 2–3 câu: tổng số bài, chủ đề nổi bật nhất hôm nay,
1 highlight đáng chú ý — bài có điểm cao nhất, chủ đề trending.}</p>

<h3>🔥 Top Picks ({n} bài)</h3>
<table border="1" cellpadding="6" cellspacing="0" style="border-collapse:collapse">
  <tr><th>Title</th><th>Source</th><th>Topic</th><th>Link</th></tr>
  <tr><td>How Uber Migrated to Apache Iceberg</td><td>HN (245 pts)</td><td>DE</td><td><a href="{url}">Read</a></td></tr>
</table>

<h3>⚙️ Data Engineering ({n})</h3>
<table border="1" cellpadding="6" cellspacing="0" style="border-collapse:collapse">
  <tr><th>Title</th><th>Source</th><th>Link</th></tr>
  <tr><td>Dagster vs Airflow in 2026</td><td>Dev.to</td><td><a href="{url}">Read</a></td></tr>
</table>

<h3>🤖 AI / ML / LLM ({n})</h3>
<!-- bảng cùng cấu trúc -->

<h3>💻 Backend / System Design ({n})</h3>
<!-- bảng cùng cấu trúc -->

<h3>☁️ Cloud / DevOps ({n})</h3>
<!-- bảng cùng cấu trúc -->

<h3>📈 Career & Leadership ({n})</h3>
<!-- bảng cùng cấu trúc -->

<h3>📊 Data Analytics ({n})</h3>
<!-- bảng cùng cấu trúc -->

<h3>📎 Bài khác ({n})</h3>
<ul>
  <li><a href="{url}">Title</a> — Source</li>
</ul>

<hr>
<p><b>Digest report</b></p>
<ul>
  <li>Sources scanned: {HN, Reddit (5 subs), Dev.to (6 tags), Medium (4 topics)}</li>
  <li>Sources failed: {Medium (captcha), Reddit/devops (rate-limit) | không có}</li>
  <li>Total articles found: {n}</li>
  <li>After dedup: {n}</li>
  <li>Top Picks: {n}, By topic: {n}, Other: {m}</li>
  <li>Skill: v1.0</li>
</ul>
```

**Quy tắc dựng:**

- Dùng đúng các thẻ và thứ tự trong khung. **Không** thêm CSS, màu nền, font, ảnh,
  `<div>`/`<span>` — Gmail bỏ phần lớn CSS
- Thứ tự section cố định: Top Picks → DE → AI → BE → CD → CR → DA → Bài khác.
  **Section không có bài thì bỏ hẳn**
- Trong Top Picks: sắp xếp theo `score` giảm dần
- Trong section chủ đề: sắp xếp theo `score` giảm dần
- **Source (hiển thị):**
  - HN: `HN ({points} pts)`
  - Reddit: `r/{subreddit}`
  - Dev.to: `Dev.to`
  - Medium: `Medium`
- **Link:** anchor text "Read", không dán URL trần
- **Khối báo cáo:** đủ 6 dòng, đúng thứ tự, luôn có mặt

**Nếu không có bài nào:** vẫn gửi email, tiêu đề
`[Tech Digest] {dd/MM} — không có bài mới`. Thân email: một dòng
`<p>Không có bài mới trong 24 giờ qua.</p>`, rồi `<hr>`, khối báo cáo.

## Ràng buộc chung

- Không bịa. Không tìm thấy thông tin thì bỏ qua, không suy đoán.
- Không tái sử dụng kết quả lần chạy trước. Mỗi lần quét lại từ đầu.
- Output tiếng Anh (giữ nguyên tiêu đề bài viết), khối báo cáo tiếng Việt.
- Múi giờ tham chiếu: Asia/Ho_Chi_Minh (GMT+7).
