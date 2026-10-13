# Lab 25B — Report

**Họ tên / nhóm:** Phạm Thị Thùy Linh — MSV 2A202602909 **Commit:** 0679431a1edf1603f74c2e561582e3c02176973f **Điểm tự động (`make grade`):** 84 / 88

> Số liệu lấy từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json` (seed 25).

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) |      PASS?      |
| ---- | ------ | -----------: | ------: | ------------: | ----------: | --------: | -----------: | :-------------: |
| G1   | naive  |       0,7194 |   7,157 |         7,738 |       6,738 |     2,163 |          4,0 |        ✗        |
| G1   | yours  |          1,0 |   4,496 |         1,061 |       0,001 |     1,954 |          0,0 |        ✓        |
| G2   | naive  |        0,456 |  15,034 |         9,904 |       8,904 |     4,627 |        155,0 |        ✗        |
| G2   | yours  |          1,0 |  10,406 |         1,038 |       0,004 |     2,015 |          0,0 |        ✓        |
| G3   | naive  |          1,0 |   4,768 |         1,164 |       0,164 |     2,328 |          0,0 | ✗ (retry_ratio) |
| G3   | yours  |          1,0 |   4,601 |         1,057 |       0,030 |     2,088 |          0,0 |        ✓        |
| G4   | naive  |          1,0 |  13,764 |         1,004 |       0,004 |     2,008 |          0,0 |     ✗ (p95)     |
| G4   | yours  |          1,0 |   5,321 |         1,014 |       0,002 |     1,967 |          0,0 |        ✓        |
| G5   | naive  |       0,3699 |   5,359 |         17,03 |       16,03 |     2,124 |            — |        ✗        |
| G5   | yours  |       0,8179 |   4,088 |         1,002 |       0,002 |     1,851 |            — |        ✓        |
| G6   | naive  |        0,517 |   4,836 |        12,344 |      11,344 |     2,009 |          0,0 |        ✗        |
| G6   | yours  |       0,9916 |   4,314 |         1,026 |       0,005 |     1,801 |          0,0 |        ✓        |

Naive 0/6, của tôi 6/6 trên cả 5 seed (25–29).

## 2. Cấu hình client và lý do

Giữ nguyên khối `client:` của starter vì đã đạt 6/6 trên 5 seed; tôi thử `attempt_timeout_s` 4/5/6/8 và breaker `window 10/min_calls 5` — availability các băng gần như không đổi (G6: 0,98–0,995), nên chọn giá trị bảo thủ, ít phản ứng thái quá.

| Tham số                         |    Giá trị | Vì sao (dựa trên băng nào, số liệu nào)                                                                                                                                                                                   |
| ------------------------------- | ---------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| attempt_timeout_s               |        8,0 | Deadline user là 30 s và một lượt có 2 call; 8 s cho phép ≥ 3 attempt/call. G4 (alpha chậm +5 s ⇒ ~6,2 s): 8 s vẫn để call hoàn tất, p95 5,3 s ≤ 10 s; breaker slow-call lo việc chuyển hướng. Thử 30 s xem thí nghiệm 2. |
| max_attempts_per_provider       |          2 | 1 retry rồi failover: G3 (500 ngẫu nhiên 30%) retry cùng provider thường thành công, nhưng không dồn thêm tải vào provider hỏng như G2 (amplification 1,04 vs naive 9,9).                                                 |
| backoff_base_s / cap_s          | 0,25 / 4,0 | Full jitter, delay ≤ 0,25·2^(n−1). Đủ nhỏ để retry vẫn nằm trong deadline, đủ ngẫu nhiên để tránh đồng bộ (naive: delay cố định 0,2 s).                                                                                   |
| retry_budget_ratio              |        0,1 | Retry ≤ ~10% request: G3 retry_ratio của tôi 0,03 (≤ 0,10) so với naive 0,164.                                                                                                                                            |
| breaker.window_size / min_calls |    20 / 10 | Cửa sổ 20 kết quả ≈ vài giây ở 8 call/s; min_calls=10 tránh mở breaker vì 2–3 lỗi lẻ.                                                                                                                                     |
| breaker.failure_rate_threshold  |        0,5 | G3 lỗi 30% **không** nên mở breaker (retry/failover xử lý), G2/G6 lỗi ≫ 50% thì mở. 0,5 tách hai trường hợp.                                                                                                              |
| breaker.slow_call_s             |        5,0 | G4 không có lỗi nào, chỉ chậm +5 s: call p50 bình thường ~1,2 s nên ≥ 5 s là bất thường ⇒ breaker mở theo `slow_call_rate`, traffic sang alpha-eu, p95 còn 5,3 s (naive 13,8 s).                                          |
| breaker.open_duration_s         |         10 | Đủ ngắn để quay lại alpha sớm sau khi hồi phục (G2 recovery 0 s, `primary_share_after_restore` ≥ 0,8), đủ dài để không bắn thăm dò liên tục.                                                                              |
| breaker.half_open_max_probes    |          2 | Chỉ 2 request thăm dò trong HALF-OPEN, tránh dồn cả đàn vào provider vừa hồi phục.                                                                                                                                        |

Hai quyết định ngoài YAML, rút ra từ G6:

- Chờ tối đa 3 lần (có jitter) cho cooldown rate-limit ngắn (≤ 5 s) thay vì bỏ cuộc ngay.
- Call `chat`, khi họ model chính không dùng được, ưu tiên `gamma-mini` (rẻ, không cần tool) trước `beta`, để dành rate limit hạ xuống 9 rps của `beta` cho call `tool`. Trước thay đổi: G6 availability 0,89 với 20+ `no_provider_available`/phút (toàn bộ là call tool); sau thay đổi: 0,99.

Sơ đồ quyết định `on_error`:

```
loại lỗi          breaker      retry cùng provider?        failover?   cooldown
CLIENT_ERROR      release      không                       không       GiveUp("client_error")
QUOTA_EXHAUSTED   release      không                       có          quota_cooldown_s = 300 s
RATE_LIMITED      release      không                       có          retry_after (mặc định 1 s)
OVERLOADED/5xx/   on_failure   có nếu tries < 2 và còn     có (khi hết không
TIMEOUT                         retry budget, delay jitter  lượt/budget)
```

## 3. Ba thí nghiệm

1. **Retry budget là lớp phòng thủ cuối** (breaker cố ý hỏng: `min_calls = 1e9`, `max_attempts_per_provider = 5`).

   | Băng | Cấu hình   | availability | amplification | retry_ratio | recovery (s) |
   | ---- | ---------- | -----------: | ------------: | ----------: | -----------: |
   | G2   | có budget  |       0,9996 |         1,383 |       0,057 |            0 |
   | G2   | tắt budget |       0,9742 |         1,600 |       0,397 |           25 |
   | G3   | có budget  |          1,0 |         1,136 |       0,068 |            0 |
   | G3   | tắt budget |          1,0 |         1,162 |       0,158 |            0 |

   Khi breaker không làm việc, budget là thứ duy nhất chặn retry ở ~6% (so với 40% khi tắt): nó giới hạn tải tương đối với traffic bất kể provider hỏng kiểu gì. Khi breaker đúng, nó cắt provider hỏng hoàn toàn nhưng cần vài kết quả xấu mới mở và có thể cấu hình sai/chậm; budget không phụ thuộc vào việc phân loại "provider hỏng" nên bắt được những gì breaker bỏ sót (lỗi 30% ở G3 không mở breaker). Hai lớp bổ trợ nhau: breaker chuyển hướng, budget chặn khuếch đại.

2. **`attempt_timeout_s: 30`.**

   | Băng | timeout | availability | p95 (s) | give_up_reasons                          |
   | ---- | ------: | -----------: | ------: | ---------------------------------------- |
   | G6   |       8 |       0,9916 |    4,31 | no_provider_available: 20                |
   | G6   |      30 |       0,9447 |    4,06 | deadline: 120, no_provider_available: 12 |
   | G4   |       8 |          1,0 |    5,32 | —                                        |
   | G4   |      30 |          1,0 |    5,51 | —                                        |

   Ở G6, `alpha-eu` "treo": với timeout 30 s một call chờ hết deadline của user mà không kịp failover (120 lượt hết hạn). p95 lại _giảm_ (4,31 → 4,06) dù availability tụt 4,7 điểm — survivorship bias vì p95 chỉ tính lượt thành công. G4 không đổi vì alpha vẫn trả lời (~6 s) nên timeout dài hay ngắn không bị chạm tới. Timeout nên dựa vào thời gian cần để kịp failover trong deadline (≈ deadline / số bước × số lần thử), không phải p99 bình thường.

3. **Bỏ lọc tool khỏi `ladder()`** (G6): `incompatible_routes = 20`, tất cả giảm thành `incompatible_model_output` — tức 20 lượt khách bị trả về một câu trả lời hỏng vì lệnh gọi công cụ (hoàn tiền, tra đơn) được gửi tới model không hỗ trợ tool. Availability đo được gần như không đổi (0,9916), nên chỉ nhìn availability sẽ không phát hiện ra; đó là lý do cần metric riêng.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds |    overpaid | wasted LLM / crash |
| -------- | ---------: | ------: | ----------------: | ---------------: | -----------: | ----------: | -----------------: |
| naive    |        1,0 |      81 |               128 |               81 |            0 | 1.496.700 ¢ |              1,123 |
| yours    |        1,0 |      66 |                 0 |                0 |            0 |           0 |              0,303 |

- Chỉ idempotency key **chưa đủ**: không có checkpoint thì sau crash worker mới chạy lại từ đầu, gọi lại LLM (tốn tiền) và không biết refund_id cũ để các bước sau dùng; kết quả mỗi lần phải gọi lại API để lấy.
- Chỉ checkpoint **chưa đủ**: khi `ServiceTimeout` (tiền đã đi nhưng mất response) hoặc crash giữa lúc gọi payment, chưa có checkpoint ⇒ lần chạy lại gọi payment lần nữa. Không có key ổn định thì đó là một lần hoàn tiền thứ hai (naive: 128 lần trùng, 1,5 triệu ¢ trả dư).
- Ghi checkpoint **trước** khi gọi payment: nếu worker chết giữa chừng thì lần chạy sau thấy "đã xong" và bỏ qua ⇒ hoàn tiền bị mất. Vì vậy thứ tự phải là: chạy step (kèm key) → ghi checkpoint.

## 5. Alerting & error budget

| Sự cố            | Mong đợi | Naive alerter   | Burn-rate alerter |     Độ trễ phát hiện |
| ---------------- | -------- | --------------- | ----------------- | -------------------: |
| checkout_bug     | page     | page            | page              |    27 phút (naive 1) |
| provider_outage  | page     | page            | page              |     4 phút (naive 0) |
| deploy_blip      | none     | page (sai)      | không             |                    — |
| slow_leak        | ticket   | không phát hiện | ticket            |            1337 phút |
| degraded_evening | page     | page            | page              | 291 phút (naive 214) |
| night_spike      | none     | page (sai)      | không             |                    — |
| small_bump       | none     | page (sai)      | không             |                    — |

Burn-rate: 3 page, 1 ticket, **0 false page** (naive: 3 false page, bỏ sót slow_leak).

Error budget 30 ngày: 25.901.755 request, được phép lỗi 129.508,8; thực tế 55.618 lỗi ⇒ availability 0,997853, đã tiêu **42,95%**, còn **57,05%** ⇒ **không** đóng băng release. `small_bump` (20 phút ở 12% lỗi) đốt budget ở tốc độ 24× nhưng chỉ ~0,5% budget tháng; đánh thức người lúc 11 giờ trưa cho việc tự hết là báo động giả làm mòn sự tin cậy vào alert — một ticket/dashboard là đủ.

## 6. Postmortem — Băng G2 với client naive (blameless)

_Hình: `reports/gameday/G2_overload_storm.png`; CSV: `reports/gameday/naive_G2_overload_storm_timeseries.csv`._

**Tóm tắt:** Lúc t=120 `alpha` mất 75% GPU, một sự cố vốn kéo dài 3 phút, nhưng lớp retry 3 tầng của client v1 (app × gateway × SDK = tối đa 36 attempt/call, delay cố định, không breaker) nhân tải lên khoảng 12–20 lần so với bình thường, khiến hàng đợi của `alpha` không bao giờ cạn. Hệ thống không tự hồi phục khi GPU trở lại (t=300) mà chỉ thoát khoảng t≈450, tức 150 s sau khi nguyên nhân gốc đã hết.

**Ảnh hưởng:** 1.077/2.362 lượt thành công (availability 0,456) ⇒ **1.285 lượt thất bại**, thời gian suy giảm ~330 s (t=120→~450), trong đó 150 s xảy ra _sau_ khi provider đã hồi phục. Nếu tháng chỉ có sự cố này với tải ~4 lượt/s (~10,4 triệu lượt/tháng, budget 0,5% ≈ 51,8k), sự cố tiêu ~2,5% budget; tính trong cửa sổ ngắn thì burn-rate ≈ 54% lỗi / 0,5% ≈ **109×** — vượt xa ngưỡng page 14,4×.

**Timeline** (giây ảo — lấy từ CSV, attempts/s cộng mọi provider):

|       t | Sự kiện                                                                  | Bằng chứng (số)                                                                        |
| ------: | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
|   0–120 | Bình thường                                                              | ~7,5 attempts/s, 895/895 call thành công (100%)                                        |
|     120 | alpha mất 75% năng lực                                                   | attempts/s vọt lên ~91 trong 30 s đầu; chỉ 51/98 call hoàn tất thành công (52%)        |
| 150–300 | Retry storm                                                              | ~146,5 attempts/s (≈ 20× bình thường), 0/631 call thành công (0%)                      |
|     300 | alpha hồi phục hoàn toàn                                                 | attempts/s vẫn 115; success 0 — **chưa hồi phục**                                      |
| 300–360 | Hàng đợi vẫn nghẹt                                                       | 19,6 attempts/s, 0/169 thành công                                                      |
| 360–420 | Đợt retry thứ hai (deadline 30 s hết hạn, call mới của user nhân 36 lần) | ~95,5 attempts/s, 0/252 thành công                                                     |
| 420–450 | Tải giảm dần                                                             | ~21,5 attempts/s, 0/108 thành công                                                     |
|    ~450 | Hồi phục thực sự                                                         | 450–480: 299/332 thành công (90%), 11,5 attempts/s; 480+: 986/986 (100%), 8 attempts/s |

**Trigger vs root cause:** _Trigger_ là mất 75% GPU của alpha (t=120–300) — một sự kiện bên ngoài, ngắn, đã được dự đoán trước là có thể xảy ra. _Root cause_ là thiết kế retry của client v1: ba tầng retry chồng nhau (tối đa 36 attempt/call) với delay cố định, không jitter, không retry budget, không circuit breaker và không fallback — nó chuyển một sự cố 3 phút thành 5+ phút sập. Vòng lặp metastable tự duy trì: provider chậm ⇒ client timeout ⇒ retry nhân tải ⇒ hàng đợi dài hơn ⇒ càng nhiều timeout. Sau t=300 vòng lặp vẫn chạy: mỗi call mới sinh tới 36 attempt, tải (~115–146 attempts/s) lớn hơn nhiều năng lực của alpha (18 slot / 1,2 s ≈ 15 req/s).

**Vì sao không tự hồi phục sau t=300:** provider là hàng đợi FIFO và _tiếp tục xử lý request mà client đã bỏ_ (timeout), nên GPU hồi phục vẫn phải tiêu hóa hàng đợi đầy request "ma"; trong lúc đó retry mới tiếp tục đổ vào, giữ tải > năng lực. Request chờ quá 10 s bị 529 và lại sinh retry. Chỉ khi call đã hết deadline 30 s và ngừng sinh attempt mới (t≈450) thì hàng đợi mới cạn.

**Điều gì giúp / điều gì làm tệ hơn:** Giúp: quan sát (CSV theo giây) cho thấy ngay attempts/s ≫ calls/s. Làm tệ: retry 3 tầng không ai sở hữu toàn bộ (mỗi tầng "hợp lý" riêng nhưng nhân với nhau); không tôn trọng `retry-after` (28.381 vi phạm); không có đường fallback sang `alpha-eu`/`beta` dù đang rảnh; không có alert theo burn-rate nên không ai nhìn thấy "đang tự làm mình sập".

**Action items** (cụ thể, có owner, đo được):

| #   | Hành động                                                                                           | Loại     | Owner                                | Cách kiểm chứng                                                                                       |
| --- | --------------------------------------------------------------------------------------------------- | -------- | ------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| 1   | Chỉ một tầng retry (ở client AI), tắt retry ở gateway và SDK; thêm full jitter + retry budget 10%   | prevent  | Nhóm platform / chủ sở hữu AI client | CI chạy băng G2: `amplification ≤ 1,2` (hiện 1,04) và `retry_ratio ≤ 0,10`                            |
| 2   | Circuit breaker theo tỷ lệ lỗi + slow-call, fallback sang `alpha-eu`/`beta` (lọc theo tool calling) | mitigate | Nhóm platform                        | CI băng G2: `recovery_s ≤ 30`, `primary_share_after_restore ≥ 0,8`                                    |
| 3   | Alert burn-rate đa cửa sổ (14,4× 1h/5m; 6× 6h/30m) cộng alert riêng cho `attempts/calls > 2`        | detect   | SRE on-call                          | Diễn tập game day: page tới on-call trong ≤ 5 phút kể từ lúc bắt đầu, 0 false page trên `deploy_blip` |
| 4   | Giới hạn thời gian chờ ở server (hủy request khi client đã bỏ) / load shedding theo deadline        | mitigate | Nhóm provider gateway                | Thử nghiệm tải: sau khi capacity hồi phục, hàng đợi cạn ≤ 30 s                                        |

Tinh thần blameless: không ai "viết sai" retry; mỗi tầng đều là mặc định hợp lý của thư viện. Thiếu một nơi nào đó nhìn toàn bộ đường đi của request và giới hạn tổng số attempt.

## 7. Một điểm yếu còn lại

G5 chỉ đạt availability 0,8179 (ngưỡng 0,78) và đó là mức tối đa có thể đạt được vì 60% request bị malformed trong 2 phút (lỗi 400 không cứu được); dư địa của tôi chỉ ~4 điểm nên băng ẩn đẩy tỷ lệ malformed cao hơn có thể làm tụt. Điểm yếu thứ hai: khi `beta` (rate limit 9 rps ở G6) gánh toàn bộ call tool, vẫn còn ~20 lượt `no_provider_available`. Cách sửa: nới cooldown theo cửa sổ rate-limit của `beta` (admission control/queue có hạn thay vì 429 rồi chờ), và thêm một deployment dự phòng có tool calling.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`, tape C1, seed 25).

| Client                  | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
| ----------------------- | -----------: | ---------------: | --------------: | -----------: | ------------: | ---------------: | --------: | :---: |
| resilient (không cache) |       0,8373 |           0,8424 |             0,0 |            0 |             0 |                0 |     1,386 |   ✗   |
| cached (của bạn)        |       0,9124 |           0,9645 |           0,652 |          157 |             0 |                3 |     0,915 |   ✓   |

| Tham số `cache:`     | Giá trị | Vì sao (số liệu nào)                                                                                                                                                                                                                                              |
| -------------------- | ------: | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| similarity_threshold |    0,92 | Paraphrase cùng intent ~0,94–0,98, các intent "gần mà khác" ~0,80–0,86. Ở 0,80 có **175 `wrong_answers` (`other_question`)**; từ 0,92 là 0. 0,97 vẫn 0 sai nhưng hit ratio giảm 0,652 → 0,579 và cost/turn tăng 0,915 → 0,965 nên 0,92 là điểm nằm giữa hai nhóm. |
| ttl_s                |      60 | Câu trả lời của tài liệu đổi không báo trước (`policy/shipping` ở t=480) chỉ bị giới hạn bởi TTL: TTL 300 ⇒ `outdated_answers` = 95 (> 15), 60 ⇒ 3, 30 ⇒ 2. 60 đạt yêu cầu mà vẫn giữ hit ratio 0,652 (30 s chỉ 0,592).                                           |
| max_stale_s          |     300 | Mọi provider sập 120 s (t=240–360); entry phải còn trong cửa sổ stale suốt thời gian đó. `max_stale_s: 0` ⇒ faq_availability 0,8733; 300 ⇒ 0,9645 (đạt ≥ 0,93).                                                                                                   |

Thí nghiệm:

1. **Ngưỡng**: 0,80 / 0,92 / 0,97 ⇒ hit 0,659 / 0,652 / 0,579; wrong_answers 175 / 0 / 0; cost/turn 0,978 / 0,915 / 0,965. Ở 0,80 toàn bộ lỗi là `other_question` — các cặp "gần mà khác" như `refund_policy` ↔ `refund_time` (hỏi chính sách bị trả lời thời gian hoàn tiền) hay `shipping_fee_city` ↔ `shipping_fee_province`.
2. **TTL**: 30 / 60 / 300 ⇒ outdated 2 / 3 / 95; hit 0,592 / 0,652 / 0,731. Webhook chỉ bắt được thay đổi có báo trước (`policy/refund` ở t=150: bị invalidate, không còn trong `wrong_answers`). Thay đổi im lặng (`policy/shipping`) không có tín hiệu nào cho cache, nên chỉ tuổi entry (TTL) giới hạn được thời gian trả sai.
3. **Stale-if-error**: `max_stale_s: 0` ⇒ faq_availability 0,8733 (so với 0,9645), `stale_served` 0 (so với 157), và băng FAIL do `faq_availability_min` — các câu FAQ hết TTL trong lúc mọi provider sập không còn cứu được.

Trả lời:

- **Vì sao không cache câu hỏi cá nhân, kể cả cache theo từng user?** Câu hỏi như "Đơn #48213 của tôi đang ở đâu?" chỉ khác nhau ở số đơn nên embedding của hai user khác nhau ~0,99 giống nhau; cache theo similarity sẽ trả đơn của người khác (rò rỉ dữ liệu). Cache theo user cũng không an toàn vì dữ liệu cá nhân đổi liên tục (trạng thái đơn) và không có tài liệu nguồn để invalidate; câu trả lời cũ sai còn tệ hơn không trả lời. Do đó personal luôn đi thẳng tới provider.
- **Alert khi cache khiến dashboard xanh?** Không alert trên availability phía user. Alert trên: tỷ lệ `stale_served` / (số lượt FAQ) (vượt vài % kéo dài), error rate/availability của _provider phía sau cache_ (`GiveUp` trước khi `_or_stale`), `cache_hit_ratio` đột ngột tăng cùng lúc `provider_calls` giảm, và tuổi lớn nhất của câu trả lời đang được phục vụ stale (tiến gần `ttl_s + max_stale_s`).
