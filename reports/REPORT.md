# Lab 25B — Report

**Họ tên / nhóm:** Bùi Gia Chính — 2A202602693  **Commit:** (xem `git log -1`)  **Điểm tự động (`make grade`):** 84 / 88

> Mọi nhận định đều có số, lấy từ `reports/gameday/*.json`, `reports/cache/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0 | 4.726 | 1.06 | 0.004 | 2.091 | 0.0 | PASS |
| G2 | naive | 0.456 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0 | 5.224 | 1.034 | 0.007 | 2.114 | 0.0 | PASS |
| G3 | naive | 1.0 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0 | 4.642 | 1.035 | 0.019 | 2.101 | 0.0 | PASS |
| G4 | naive | 1.0 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 0.9995 | 5.419 | 1.036 | 0.007 | 2.124 | 0.0 | PASS |
| G5 | naive | 0.3699 | 5.359 | 17.03 | 16.03 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.484 | 1.008 | 0.007 | 2.518 | None | PASS |
| G6 | naive | 0.517 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0 | 5.957 | 1.058 | 0.019 | 2.211 | 0.0 | PASS |

Client của bạn PASS cả 6 băng; naive FAIL cả 6. Chênh lệch lớn nhất ở G2 (avail 0.456 vs 1.0) và G5 (0.3699 vs 0.8179 — phần còn lại là lượt *malformed thật* không cứu được, không phải lỗi client).

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 4.0 | Chạy 8s → 5s → 4s → 3s trên G2/G6. 8s: G2 p95=10.36, G6 p95=5.96 (2 lần hang = 16s). 4s: G2 p95=5.22, G6 p95=5.96, mọi băng PASS. 3s: G4 avail=0.9991, G5 p95=6.76 — timeout oan làm retry tăng (retry_ratio 0.018–0.048 vs 0.004–0.019). |
| max_attempts_per_provider | 2 | 1 retry cùng provider rồi failover. Đủ hút jitter cho lỗi thoáng (G3 retry_ratio=0.019), không đủ tạo storm. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Full jitter: retry đầu ≤0.25s, tối đa 4s. Giữ amplification ≤1.06 trên mọi băng. |
| retry_budget_ratio | 0.1 | Giới hạn retry ≤10% traffic. Thí nghiệm §3.1: tắt budget → G3 ampl 1.063→1.066, G2 giữ 1.033. Quan trọng khi breaker sai cấu hình. |
| breaker.window_size / min_calls | 20 / 10 | 10 kết quả mới bắt đầu xét tỷ lệ — không trip ở "2 lỗi lúc 3h sáng"; 20 đủ nhanh trip trong sự cố 3 phút. |
| breaker.failure_rate_threshold | 0.5 | Trip khi ≥50% lỗi. G3 (30% lỗi thật) không trip — đúng: brownout vẫn phục vụ được, trip sẽ đẩy hết sang beta. |
| breaker.slow_call_s | 5.0 | G4: alpha chậm +5s, mọi câu trả lời 6–7s ≥5s → slow-rate trip, failover. p95 13.76→5.42. |
| breaker.open_duration_s | 10.0 | Đủ dài để filter sự cố, đủ ngắn để probe lại sớm → G2 recovery=0s. |
| breaker.half_open_max_probes | 2 | Giới hạn 2 probe khi HALF_OPEN — không thundering herd, provider hồi phục thật sự. |

Sơ đồ quyết định của `on_error`:

```
CLIENT_ERROR      -> release permit; GiveUp("client_error"); KHÔNG retry, KHÔNG failover, KHÔNG tính breaker
QUOTA_EXHAUSTED   -> release permit; cooldown provider + quota_cooldown_s (300s); failover ngay
RATE_LIMITED      -> release permit; cooldown provider đến now+retry_after; failover ngay
OVERLOADED /      -> breaker.on_failure; tries += 1
SERVER_ERROR /      nếu tries < max_attempts_per_provider
TIMEOUT             và budget.try_acquire_retry OK
                  và now+delay+min_time_left < deadline
                    -> retry CÙNG provider sau full-jitter delay
                  còn lại -> failover (idx += 1, delay = 0)
```

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối.** Với `min_calls=1000000000` (breaker không bao giờ trip) và `max_attempts_per_provider=5`, tắt budget (`ratio=100, max_tokens=1e9, min_per_s=1e6`) rồi so:
   - G2: ampl 1.033 (off) vs 1.033 (on); retry_ratio 0.007 vs 0.007 — breaker *đúng* đã chặn phần lớn trước khi budget kịp tiêu.
   - G3: ampl 1.066 (off) vs 1.063 (on); retry_ratio 0.055 (off) vs 0.036 (on) — budget cắt ~35% retry.
   Khi breaker hoạt động đúng, budget ít thay đổi vì breaker đã chặn retry trên provider chết. Vẫn cần cả hai vì: (a) breaker sai cấu hình (min_calls quá lớn, threshold quá cao) thì budget là lưới chặn cuối; (b) breaker chặn provider *chết*, budget chặn số lượng retry nói chung kể cả trên provider *sống nhưng chậm*; (c) chúng phòng thủ ở hai lớp khác nhau — không thay thế được nhau.
2. **`attempt_timeout_s: 30`.** G4: p95 5.42→5.61, avail vẫn 1.0 — alpha chậm nhưng vẫn trả lời dưới 30s nên không timeout. G6: avail 1.0→**0.9488 (FAIL)** — alpha-eu "hang" không bao giờ trả lời, mỗi lần treo 30s, hết deadline trước khi failover sang beta. Timeout "đúng" phải dựa vào: distribution độ trễ bình thường của provider (median ~1.2–1.8s) + độ trễ bạn sẵn sàng chờ trước khi coi là chết, KHÔNG phải timeout lớn để "chắc ăn".
3. **Bỏ lọc tool calling khỏi `ladder()`.** G6: `incompatible_routes=269`, avail 1.0→0.8872. Diễn đạt cho PM: "269 câu trả lời của hệ thống bị vứt bỏ vì gửi yêu cầu cần gọi công cụ (kiểm tra đơn hàng) cho một mô hình không biết gọi công cụ. Khách nhận câu trả lời lạc đề hoặc lỗi. Về chi phí, chúng ta vẫn trả tiền cho 269 câu mô hình mà không dùng được."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1.574.200 cents | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 | 0.303 |

Chỉ idempotency key **chưa đủ**: key de-dupe ở payment provider, nhưng worker chết thì không ai *biết* bước đã xong — worker mới không checkpoint thì vẫn phải gọi lại classify/lookup (wasted LLM), và nếu step chưa ghi key vào provider thì key không cứu được. Chỉ checkpoint **chưa đủ**: checkpoint ghi "đã gọi payment" không chặn provider gọi lần hai nếu worker chết *trước khi* ghi checkpoint nhưng *sau khi* tiền đã đi — cần key để provider tự nhận ra "đây là lần gọi lại của cùng một giao dịch". Nếu ghi checkpoint *trước* khi gọi payment: worker chết ngay sau đó → worker mới thấy checkpoint "đã thanh toán" → bỏ qua bước refund → **tiền không bao giờ được hoàn nhưng hệ thống báo done** (lost refund). Đúng thứ tự là: gọi payment (idempotency key) → thành công → ghi checkpoint.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 phút |
| provider_outage | page | page | page | 4 phút |
| deploy_blip | none | **page (false)** | none | — |
| slow_leak | ticket | none (miss) | ticket | 1337 phút |
| degraded_evening | page | page | page | 291 phút |
| night_spike | none | **page (false)** | none | — |
| small_bump | none | **page (false)** | none | — |

Naive: 7 page, 3 false page, bỏ lọt slow_leak. Burn-rate: 3 page, 1 ticket, **0 false page**.

Error budget 30 ngày: đã tiêu **42,95%**, còn **57,05%**. Không đóng băng release (`freeze_releases=False`).

`small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa vì: (1) luật 1h&5m yêu cầu burn ≥14,4× *liên tục* cả hai cửa sổ — blip 20 phút không đủ dài để cửa sổ 1h tích lũy đủ; (2) 20 phút × 12% ≈ 0,24% tổng budget tháng, hệ thống tự phục hồi; (3) page半夜blip gây "alert fatigue" → on-call bỏ qua alert thật.

## 6. Postmortem — Băng G2 với client naive (blameless)

**Tóm tắt (3 câu):** Alpha mất 75% GPU trong 3 phút (t=120–300), gây 529 + queue dài. Client naive retry 3 tầng × không jitter × không breaker nên mỗi lỗi sinh 36 attempt, khuếch đại tải gấp ~10×, biến sự cố 3 phút thành 155 giây không phục hồi được sau khi provider đã hồi phục. Vòng lặp metastable: retry giữ tải > năng lực → hàng đợi đầy → 529/timeout → retry tiếp.

**Ảnh hưởng:** 585 lượt fail vì `deadline` + 655 vì `retries_exhausted` = 1240/2700 lượt (~54%) thất bại trong ~255 giây (t≈140–450). Error budget tháng giả định chỉ có sự cố này: 54% × (255/2.592.000s × 30d) — không, tính theo phút: ~1240 lỗi / tổng request tháng ≈ 0,05% budget; nhưng *availability băng* = 0.456, vượt xa ngưỡng 0,95.

**Timeline** (giây ảo — từ `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 119 | Bình thường trước sự cố | ratio=1.0, 9 attempts/giây, ~7 ok/giây |
| 120 | alpha mất 75% GPU | capacity 0.25 → service time ×4, queue bắt đầu dài |
| 140 | Vượt ngưỡng shed | 216 attempts/giây trên alpha (từ 5), 0 ok — 529 `overloaded_error` |
| 150 | Retry storm toàn hệ thống | 163 attempts/giây, 0/3 ok; naive vẫn chỉ gửi alpha |
| 200 | Đỉnh metastable | 135 attempts/giây, 0/1 ok; 26.298 `rate_limit_exceeded` tích lũy |
| 299 | Vài giây trước khi hồi phục | 136 attempts/giây, 0/1 ok — hệ thống hoàn toàn chết |
| 300 | alpha hồi phục 100% năng lực | 115 attempts/giây VẪN 0/5 ok — hàng đợi cũ đầy, tải đến > năng lực |
| 310 | Bắt đầu tan | 11 attempts/giây, vẫn 0/1 ok |
| 450 | Chưa thực sự phục hồi | 16 attempts/giây, 0/3 ok |
| 500 | Hồi phục thật | 9 attempts/giây, 10/10 ok (ratio=1.0) |

**Trigger vs root cause:** Trigger là alpha mất 75% GPU (sự cố bên ngoài, không kiểm soát được). Root cause là **kiến trúc retry của client**: 3 tầng retry chồng lên nhau (app × gateway × SDK = 36 attempt/logical call), delay cố định 0.2s không jitter (mọi client retry cùng lúc), không có circuit breaker (vẫn bắn vào provider đang hấp hối), không có deadline awareness (retry vô hạn tới `retries_exhausted`).

Vì sao không tự hồi phục sau t=300: mô hình provider mô phỏng hàng đợi FIFO — request mà client đã "bỏ" (timeout) **vẫn nằm trong hàng đợi** và vẫn được xử lý. Khi alpha hồi phục, nó phải xử lý hàng nghìn request cũ mà không ai còn chờ kết quả; retry storm tiếp tục bơm thêm. Năng lực xử lý backlog < tốc độ retry đến → deadlock. Chỉ khi retry giảm (naive hết `retries_exhausted` dần) hàng đợi mới tan → t=500.

**Điều gì giúp / điều gì làm tệ hơn:**
- Giúp: không có (toàn bộ baseline là làm tệ hơn).
- Làm tệ hơn: (1) retry không jitter → đồng bộ hóa; (2) 3 tầng retry nhân lượng; (3) không breaker → không có tín hiệu "dừng lại"; (4) timeout 8s quá dài trong storm → giữ kết nối chết.

**Action items:**

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay retry 3 tầng bằng 1 tầng có full jitter + retry budget (ratio≤10%) | prevent | Platform | `make compare` G2: amplification ≤1,2 trong CI |
| 2 | Thêm sliding-window circuit breaker theo failure rate (threshold 0.5, window 20, min_calls 10) | mitigate | Platform | G2: breaker OPEN trong t=140–310, không có attempt nào vào alpha khi OPEN |
| 3 | Alert amplification > 1,5 trong 5 phút liên tục | detect | SRE | Dashboard: amplification gauge; burn-rate page khi vượt 1,5× |
| 4 | Rút attempt_timeout_s xuống ~4s (dựa trên p50 provider ~1.5s) | prevent | Platform | G6: p95 < 7s, avail ≥ 0,95 |

## 7. Một điểm yếu còn lại

Khi **cả 4 provider cùng chậm** (cùng brownout mạng vùng chung), client không có provider tốt để failover: breaker từng provider sẽ trip, HALF_OPEN probe cũng chậm → `_or_stale`/`GiveUp` vẫn xảy ra. p95 sẽ chạm ceiling ~timeout_s × max_attempts. Cách sửa: thêm **deadline propagation** (truyền deadline thật của user xuống từng attempt thay vì timeout cố định) + **adaptive timeout** ước tính từ latency distribution gần nhất của từng provider (vd. p99 × 1.2), và cân nhắc **hedged request** (gửi song song đến 2 provider, lấy kết quả về nhanh nhất) cho calls quan trọng.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.0 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.912 | 0.9645 | 0.681 | 141 | 0 | 13 | 0.896 | PASS |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.92 | 0.80 gây `wrong_answers=181` (`other_question`: "thời gian hoàn tiền" vs "chính sách hoàn tiền" cosine~0.83). 0.92: hit_ratio=0.681, wrong=0. |
| ttl_s | 90 | 120s → outdated=24 (>15); 90s → outdated=13 (≤15). TTL giới hạn tuổi answer tối đa, webhook không cứu được outdated (webhook chỉ invalidate khi *có* webhook — silent update t=480 không có). |
| max_stale_s | 300 | Outage 120s (t=240–360). 300s đủ để answer cũ sống qua toàn bộ outage → faq_avail=0.9645 ≥0.93. |

Thí nghiệm:

1. **Ngưỡng.** 0.80: hit_ratio=0.742, wrong=181 (`other_question`), outdated=80. 0.92: hit=0.681, wrong=0. 0.97: hit_ratio ~0.45 (chỉ exact paraphrase mạnh), wrong=0, cost/turn tăng vì miss nhiều. Câu bị trả nhầm ở 0.80: các cặp câu về *chính sách* khác nhau nhưng cùng embedding gần (vd. refund time vs refund policy) — cosine ~0.82–0.85 < 0.92.
2. **TTL.** 300s: outdated=24; 90s: outdated=13. Webhook (t=150, `policy/refund`) invalidate ngay khi CMS báo — nhưng *silent update* (t=480, `policy/shipping`) **không có webhook**, nên answer cũ về phí ship vẫn sống đến khi TTL hết. TTL là cơ chế duy nhất giới hạn tuổi answer khi không có webhook.
3. **Stale-if-error.** `max_stale_s: 0`: faq_availability 0.9645→0.8424 (giống không cache trong outage) — mọi FAQ call trong t=240–360 fail vì không provider nào trả lời. Stale window chính là thứ cứu 141 lượt.

Trả lời:

- **Vì sao không cache câu hỏi cá nhân?** "Đơn #48213 của tôi đang ở đâu?" phụ thuộc *dữ liệu cá nhân* của từng user, thay đổi theo thời gian (trạng thái đơn cập nhật theo phút). Cache theo từng user vẫn sai vì: (1) TTL không đủ ngắn để phản ánh thay đổi real-time; (2) nguy cơ data leak nếu cache key không đủ phân biệt user (lỗi `other_user` trong test); (3) personal answers không có "câu trả lời chung" để hit — mỗi user một embedding duy nhất, cache không mang lại hit ratio. Chi phí risk >> benefit.
- **Alert khi đang phục vụ stale:** giám sát **`stale_served` rate** và **`cache_hit_ratio` bất thường** (hit_ratio nhảy lên + stale_served > 0 nghĩa là outage đang được mask). Dashboard nên alert khi `faq_availability < 0.95` HOẶC `stale_served_per_min > 0` — vì `faq_availability` tính cả lượt stale, vẫn có thể xanh trong khi providers chết. Thêm alert `provider_error_rate` (từ errors_by_code) để on-call biết *nguyên nhân* là providers down, không phải cache hoạt động tốt.
