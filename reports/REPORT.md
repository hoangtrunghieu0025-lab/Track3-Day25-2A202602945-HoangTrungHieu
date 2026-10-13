# Lab 25B — Report

**Họ tên / nhóm:** Hoàng Trung Hiếu (MSV: 2A202602945)  **Commit:** `ef4b253`  **Điểm tự động (`make grade`):** 84.0 / 88 (96.0 / 100 kể cả bộ dữ liệu ẩn)

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.786 | 1.064 | 0.001 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 9.789 | 1.032 | 0.003 | 2.100 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.714 | 1.036 | 0.016 | 2.110 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.536 | 1.016 | 0.003 | 2.091 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8108 | 5.578 | 1.003 | 0.002 | 2.521 | None | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 6.489 | 1.056 | 0.019 | 2.196 | 0.0 | **PASS** |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Dựa trên G6 và G2. User turn deadline là 30s với 2 calls (tool + chat). Đặt 8.0s giúp mỗi attempt đủ thời gian hoàn thành (service median ~1.2–1.8s) nhưng không chiếm trọn deadline, cho phép thử primary và failover sang secondary/tertiary (8s + 8s = 16s < 30s). Ở thí nghiệm 2, đặt 30s khiến availability G6 tụt từ 100% xuống 94.88% vì kẹt ở alpha-eu. |
| max_attempts_per_provider | 2 | Dựa trên G2, G3. Cho phép tối đa 1 lần retry trên cùng một provider (tổng 2 attempts) đối với lỗi transient. Ở G3 (30% 500 error), 1 lần retry giúp đạt availability 1.0 mà retry_ratio chỉ 0.016 (dưới trần 0.10). Ngược lại, nếu cho phép 5 attempts thì retry bùng nổ làm sập hàng đợi. |
| min_time_left_s | 1.0 | Ngăn chặn việc gửi attempt mới khi thời gian còn lại đến deadline của user (<1s) không đủ để nhận kết quả (median service time là 1.2s), tránh lãng phí GPU của provider. |
| backoff_base_s / cap_s | 0.25 / 4.0 | Dựa trên G1 và G2. Full jitter với base 0.25s giải phóng đồng loạt các request tránh thundering herd; cap 4.0s đảm bảo thời gian chờ không vượt quá giới hạn deadline còn lại của lượt hội thoại. |
| retry_budget_ratio | 0.1 | Dựa trên G2, G3. Token bucket hạn chế retry tối đa chiếm 10% tổng lưu lượng. Ở G3, retry_ratio là 0.016 <= 0.10. Trong thí nghiệm 1 khi breaker bị vô hiệu hóa, budget này là chốt chặn giữ amplification ở 1.382 thay vì vọt lên 1.672. |
| retry_budget_min_per_s / max_tokens | 0.5 / 20 | Cho phép hồi phục 0.5 token/giây để các request ban đêm/tải thấp vẫn có thể retry, và giới hạn tối đa 20 token để không tích lũy đạn dược gây burst retry storm. |
| quota_cooldown_s | 300.0 | Dựa trên G5. Khi account A hết quota (429 insufficient_quota), cooldown 300s ngăn toàn bộ request sau đó gửi vào A, failover ngay sang B, giữ unneeded_failovers = 0 và availability đạt 81.08% (>= 78%). |
| breaker.window_size / min_calls | 20 / 10 | Dựa trên G2, G4. Cửa sổ trượt 20 kết quả gần nhất phản ánh nhanh chóng độ suy thoái của dịch vụ trong ~2.5 giây (ở tốc độ 8 calls/s), min_calls = 10 ngăn ngừa việc mở nhầm mạch khi chỉ có 1–2 lỗi thoáng qua. |
| breaker.failure_rate_threshold | 0.5 | Dựa trên G2, G6. Khi provider sập (503 hoặc 529 quá tải), tỷ lệ lỗi >50% lập tức trip breaker sang OPEN, chuyển hướng lưu lượng sang alpha-eu/beta, giảm amplification từ 9.904 (naive) xuống 1.032. |
| breaker.slow_call_s / slow_rate_threshold | 5.0 / 0.6 | Dựa trên G4. Khi alpha bị nghẽn mạng chậm thêm 5s (không có lỗi HTTP), cuộc gọi kéo dài ~6.2s > 5.0s. Khi >60% cuộc gọi bị chậm, breaker trip OPEN, kéo p95 của G4 từ 13.764s (naive) xuống 5.536s. |
| breaker.open_duration_s | 10.0 | Dựa trên G2, G6. 10 giây đủ để hàng đợi của provider xả bớt tải, đồng thời sau khi sự cố kết thúc (t=300 ở G2, t=420 ở G6), breaker nhanh chóng thăm dò và đưa hệ thống hồi phục trong recovery_s = 0.0s. |
| breaker.half_open_max_probes / successes | 2 / 2 | Dựa trên G2. Chỉ cho đúng 2 probes đồng thời ở HALF-OPEN để tránh thundering herd tái bùng phát sự cố; cần 2 probes thành công liên tiếp để xác nhận sức khỏe trước khi đóng mạch hoàn toàn. |

Sơ đồ quyết định của `on_error`:

```
                           +--------------------------+
                           |  Nhận ProviderError      |
                           +--------------------------+
                                        |
                                [Phân loại lỗi]
                                        |
        +-------------------------------+-------------------------------+
        |                               |                               |
 [CLIENT_ERROR]           [QUOTA_EXHAUSTED / RATE_LIMITED]     [OVERLOADED / SERVER_ERROR / TIMEOUT]
 (400, 401, 422)                 (429 quota / 429 limit)             (529, 503, 500, 0 timeout)
        |                               |                               |
  Release permit                  Release permit                   Ghi on_failure vào Breaker
        |                               |                               |
  GiveUp("client_error")    Cooldown provider:                  Có thể retry cùng provider?
  (0 retry, 0 failover)     * Quota: now + 300s                 (tries < max_attempts VÀ
                            * Rate limit: now + retry_after      budget.try_acquire_retry() VÀ
                                        |                        now + delay + 1s <= deadline)
                                Failover ngay lập tức                   |
                                sang provider tiếp theo           +-----+-----+
                                                                  |           |
                                                                (CÓ)        (KHÔNG)
                                                                  |           |
                                                           Send(provider,   Failover ngay
                                                           delay=backoff)   sang provider tiếp theo
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.**  
   Giả lập breaker hỏng (`min_calls: 1000000000`) và `max_attempts_per_provider: 5`:
   - Trên **G2**, khi *giữ* retry budget: availability đạt **0.9996**, amplification là **1.382**, retry_ratio là **0.058**, recovery_s = **0.0s**. Khi *tắt* budget: availability tụt xuống **0.9649**, amplification vọt lên **1.672**, retry_ratio tăng gần 10 lần lên **0.491**, recovery_s chậm thành **30.0s**.
   - Trên **G3**, khi giữ budget: retry_ratio là **0.067** (đạt chuẩn <=0.10); khi tắt budget: retry_ratio tăng vọt lên **0.162** (vi phạm tiêu chí).
   - *Kết luận:* Khi breaker hoạt động đúng, nó ngắt tải theo tỷ lệ lỗi; nhưng nếu breaker bị cấu hình sai, phản ứng trễ hoặc trong các pha brownout chưa vượt ngưỡng ngắt, retry budget đóng vai trò là chốt chặn toán học cuối cùng khống chế lượng retry không vượt quá 10% lưu lượng hệ thống. Cần có cả hai để đạt defense-in-depth.

2. **`attempt_timeout_s: 30`.**  
   Chạy G6 (outage toàn bộ Provider A, alpha trả 503, alpha-eu treo cứng) và G4 (alpha chậm 5s):
   - Trên **G6**: Client chuẩn (timeout 8s) đạt availability **1.0000** (100%), p95 = **6.048s**, giveups = `{}`. Khi đổi sang timeout 30s: availability **tụt xuống 0.9488** (vi phạm min 0.95), xuất hiện **108 lượt bỏ cuộc do deadline** (`give_up_reasons: {'deadline': 108, 'no_provider_available': 14}`). Đáng chú ý, p95 lại *giảm ảo* từ 6.048s xuống **5.425s**. Đây là hiện tượng *survivorship bias*: các lượt bị treo 30s ở alpha-eu đã chết do quá deadline nên bị loại khỏi tập tính p95 (chỉ tính trên lượt thành công).
   - Trên **G4**: Cả hai mức timeout đều đạt availability **1.0000** và p95 gần như không đổi (**5.604s** vs **5.683s**). Lý do: G4 được bảo vệ bởi circuit breaker phát hiện cuộc gọi chậm (`slow_call_s: 5.0`), breaker đã ngắt alpha và failover sang alpha-eu từ trước khi chạm mốc timeout.
   - *Kết luận:* Timeout của một attempt không được đặt bằng deadline của người dùng, mà phải dựa trên `p99 bình thường + buffer` đủ nhỏ (6–8s) để chừa thời gian cho ít nhất 2–3 lần failover sang provider dự phòng trong tổng ngân sách 30s của user.

3. **Bỏ lọc tool calling khỏi `ladder()`.**  
   Chạy G6 khi không lọc `supports_tools`:
   - Availability tụt từ **1.0000** xuống **0.8927**, ghi nhận **256 incompatible routes** (`incompatible_routes: 256`), xuất hiện 78 timeout và 67 rate limit do tắc nghẽn.
   - *Giải thích góc độ Product Manager:* Khi Provider A sập, hệ thống định tuyến mù quáng chuyển tiếp yêu cầu xử lý hoàn tiền (bước 1 cần gọi tool hoàn tiền) sang `gamma-mini` — một model giá rẻ không hề có tính năng Tool Calling. Hậu quả là 256 khách hàng bị từ chối phục vụ ngay lập tức, quy trình hoàn tiền bị đứt gãy, gây tổn hại trực tiếp đến trải nghiệm khách hàng và độ tin cậy của sản phẩm chỉ vì tầng hạ tầng không kiểm tra tính tương thích tính năng giữa các dòng model.

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 (100%) | 81 | 128 | 81 | 0 | 1,496,700 ₫ | 1.123 |
| yours | **1.0 (100%)** | 66 | **0** | **0** | **0** | **0 ₫** | **0.303** |

**Vì sao chỉ idempotency key là chưa đủ, và chỉ checkpoint cũng chưa đủ?**
- *Nếu chỉ có Idempotency Key:* Khi worker bị crash giữa chừng, worker mới khởi động lại không hề biết các bước trước đã hoàn thành đến đâu nếu không có checkpoint. Worker mới buộc phải chạy lại từ đầu tất cả các bước (tốn chi phí gọi LLM classify, query database), làm tăng lãng phí tài nguyên (`wasted_llm_per_crash` cao).
- *Nếu chỉ có Checkpoint:* Khi worker đang gọi payment thì gặp sự cố mạng (mất response) hoặc worker chết ngay sau khi tiền đã được trừ nhưng chưa kịp nhận kết quả trả về. Nếu không có idempotency key, khi worker mới retry bước thanh toán, API thanh toán sẽ xem đó là giao dịch mới và trừ tiền lần 2, dẫn đến hoàn tiền trùng lặp (duplicate refund, thất thoát tài chính).
- *Nếu ghi checkpoint TRƯỚC khi gọi payment:* Nếu worker crash ngay sau khi lưu checkpoint nhưng trước khi lệnh trừ tiền được gửi đi, worker mới khởi động lại thấy checkpoint đã tồn tại sẽ tưởng rằng thanh toán đã thành công và bỏ qua bước này để gửi email xác nhận. Hậu quả là khách hàng nhận được email thông báo hoàn tiền nhưng tài khoản thực tế không nhận được tiền (**lost refund** - lỗi nghiêm trọng nhất trong hệ thống tài chính). Do đó, nguyên tắc bất di bất dịch là: **Thực thi side-effect với Idempotency Key trước, lưu Checkpoint sau**.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page (delay 1m) | page | 27 min |
| provider_outage | page | page (delay 0m) | page | 4 min |
| deploy_blip | none | **page (False Alarm)** | none (paged: False) | Không page |
| slow_leak | ticket | **none (Bỏ sót)** | ticket | 1337 min |
| degraded_evening | page | page (delay 214m) | page | 291 min |
| night_spike | none | **page (False Alarm)** | none (paged: False) | Không page |
| small_bump | none | **page (False Alarm)** | none (paged: False) | Không page |

- **Error budget 30 ngày:** Tổng 25,901,755 requests; cho phép lỗi 129,508.8 requests (SLO 99.5%); thực tế lỗi 55,618 requests; độ khả dụng đạt **99.7853%**.
- Ngân sách lỗi đã tiêu: **42.95%** (`budget_consumed: 0.4295`), còn lại: **57.05%** (`budget_remaining: 0.5705`).
- **Quyết định:** **KHÔNG đóng băng release** (`freeze_releases: False`) vì ngân sách lỗi vẫn còn hơn 57%.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**  
  Trong 20 phút với tỷ lệ lỗi 12%, lượng lỗi phát sinh chỉ chiếm khoảng `20 × (12% - 0.5%) × rate` ≈ 2.3% ngân sách lỗi của cả tháng. Sự cố này mang tính tạm thời và tự hồi phục, hoàn toàn không đe dọa đến mục tiêu SLO 30 ngày. Sử dụng Naive Alerter (ngưỡng 5 phút > 5%) sẽ phát sinh cảnh báo giả (false page), làm gián đoạn công việc của kỹ sư trực ca và gây hội chứng chai lì cảnh báo (alert fatigue). Burn-rate Alerter với cửa sổ kép (long window 1h và short window 5m với burn-rate 14.4x) đã lọc bỏ thành công sự cố này mà không phát tín hiệu page.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Hình phân tích: `reports/gameday/G2_overload_storm.png` | Dữ liệu: `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**  
Vào lúc t=120s của kịch bản G2, cụm máy chủ Provider Alpha bị sụt giảm 75% năng lực xử lý trong vòng 3 phút khiến hàng đợi bị nghẽn và phát sinh lỗi quá tải (529/timeout). Bản client NaiveClient với 3 tầng retry cố định không có backoff/breaker đã kích hoạt cơn bão retry storm, đẩy tải gửi lên Alpha tăng đột biến lên tới 254 attempts/s và duy trì trạng thái quá tải kéo dài. Mặc dù Alpha đã hồi phục 100% phần cứng vào lúc t=300s, toàn bộ hệ thống vẫn bị tê liệt thêm 155 giây và chỉ hồi phục hoàn toàn vào t=455s do rơi vào vòng lặp suy thoái giả bền vững (metastable failure).

**Ảnh hưởng:**  
- 1,285 trên tổng số 2,362 lượt khách hàng thất bại hoàn toàn (availability chỉ đạt **45.6%** so với mục tiêu 95%).
- Thời gian gián đoạn kéo dài **335 giây** (từ t=120s đến t=455s).
- Tổng số attempt bị khuếch đại lên 34,475 lượt (amplification **9.904x**). Lượng lỗi tiêu tốn tương đương hơn 150% ngân sách lỗi tháng của dịch vụ.

**Timeline (giây ảo — trích xuất từ `naive_G2_overload_storm_timeseries.csv`):**

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 119 | Trạng thái bình thường trước sự cố | 9 attempts/s gửi tới Alpha, 7/7 calls hoàn thành thành công (success ratio 100%). |
| 120 | Alpha mất 75% năng lực xử lý (GPU capacity factor 0.25) | Hàng đợi Alpha bắt đầu ùn ứ; service time kéo dài vượt ngưỡng max queue wait 10s. |
| 136 | Bắt đầu phát sinh lỗi 529 và timeout, kích hoạt bão retry | attempts_alpha tăng vọt lên 53 attempts/s, success ratio tụt về 0% (calls_ok: 0/0). |
| 143 | Retry storm đạt đỉnh khuếch đại | attempts_alpha đạt đỉnh **254 attempts/s** (>12 lần tải thông thường 20 rps), calls_ok: 0. |
| 200..299 | Alpha ngập chìm trong trạng thái quá tải liên tục | Duy trì 130–180 attempts/s, toàn bộ lượt người dùng thất bại, retry_after_violations = 28,381. |
| 300 | Alpha phục hồi 100% dung lượng phần cứng | attempts_alpha vẫn duy trì ở mức 115 attempts/s; calls_ok vẫn bằng 0 (0% thành công). |
| 305..452 | Vòng lặp metastable failure tự duy trì | Hàng đợi FIFO ngập tràn các request mà client đã từ bỏ; GPU xử lý công việc vô ích; calls_ok duy trì ở mức 0/s. |
| 455 | Hàng đợi xả hết các request tồn đọng, hệ thống thoát suy thoái | attempts_alpha giảm về 13 attempts/s; calls_ok bắt đầu phục hồi: 4/5 (80%). |
| 456 | Hệ thống hồi phục hoàn toàn (sau 155s kể từ khi Alpha sửa xong) | calls_ok đạt 8/8 (100%), attempts_alpha ổn định ở mức 15 attempts/s. |

**Trigger vs Root cause:**  
- *Trigger:* Sự cố phần cứng/hạ tầng bên phía nhà cung cấp Alpha lúc t=120s làm mất 75% GPU trong 3 phút.
- *Root cause:* Thiết kế client v1 (NaiveClient) thiếu các cơ chế bảo vệ cốt lõi: 3 tầng retry xếp chồng (tối đa 36 attempts/call), khoảng cách retry cố định (0.2s) không có full jitter, phớt lờ header `retry-after`, không có circuit breaker để cắt mạch quá tải, và không có retry budget để chặn bão khuếch đại lưu lượng.
- *Vì sao không tự hồi phục sau t=300:* Cơ chế hàng đợi FIFO của Alpha tiếp tục xử lý các request cũ đã xếp hàng từ trước. Khi client đã timeout (sau 8s) hoặc user đã rời đi (sau 30s), máy chủ Alpha vẫn tiếp tục tốn năng lực xử lý các request "rác" này. Trong khi đó, các lượt retry dồn dập đổ vào giữ mức tải luôn vượt quá khả năng tiếp nhận (115–140 req/s > 20 rps limit), khiến các request mới tiếp tục phải xếp hàng >10s và bị lỗi 529. Vòng lặp này tự duy trì cho đến khi lượng request rác được xả hết vào t=455s.

**Điều gì giúp / điều gì làm tệ hơn:**  
- *Làm tệ hơn:* Càng retry nhanh và nhiều thì hàng đợi càng nghẽn; việc không lắng nghe `retry-after` vi phạm rate limit hàng chục nghìn lần; không failover sang `alpha-eu` hay `beta`.
- *Giúp giải quyết (trong ResilientClient):* Circuit breaker lập tức mở mạch sau khi phát hiện lỗi, chuyển toàn bộ lưu lượng sang `alpha-eu` và `beta`, chặn đứng retry storm (amplification giảm từ 9.904 xuống 1.032), và thăm dò nhẹ nhàng với 2 probes ở HALF-OPEN cho phép Alpha hồi phục ngay lập tức tại t=300 (recovery_s = 0.0s).

**Action items:**

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế NaiveClient bằng ResilientClient tích hợp Circuit Breaker, Exponential Backoff với Full Jitter và Retry Budget (ratio 0.1). | Prevent | Đội Core Platform | Chạy CI benchmark kịch bản G2 đạt `amplification <= 1.2` và `recovery_s <= 5s`. |
| 2 | Cấu hình Fallback ladder đa vùng tự động chuyển vùng sang `alpha-eu` và `beta` khi phát hiện quá tải hoặc tỷ lệ lỗi >50%. | Mitigate | Đội SRE / Infra | Diễn tập Game Day định kỳ; kiểm chứng `primary_share_after_restore >= 80%`. |
| 3 | Triển khai hệ thống giám sát SLO Burn-rate Alerting đa cửa sổ thay thế cho alert theo ngưỡng tức thời 5 phút. | Detect | Đội Monitoring | Xác minh trên 30 ngày dữ liệu lịch sử đạt 0 false page và phát hiện outage trong vòng <5 phút. |

## 7. Một điểm yếu còn lại

**Tình huống điểm yếu:**  
Trong kịch bản Provider A gặp sự cố diện rộng trên cả hai vùng (G6: alpha 503 và alpha-eu treo cứng), toàn bộ lưu lượng dồn sang cụm `beta`. Do model B-large có chi phí đắt hơn 40% (1.4/call so với 1.0/call của alpha), chi phí trung bình trên mỗi lượt thành công tăng vọt. Đồng thời, vì mọi client đều dồn sang `beta`, hạn mức rate limit của `beta` bị hạ xuống còn 9 rps (dưới mức tổng nhu cầu), dẫn đến việc client phải chờ đợi cooldown hoặc đẩy một phần chat sang `gamma-mini`. Nếu cả 4 provider cùng suy thoái hiệu năng (chậm đồng loạt), circuit breaker ở tất cả các node đều có nguy cơ bị mở mạch, dẫn đến cạn kiệt provider khả dụng và buộc phải trả về câu trả lời suy giảm (GiveUp).

**Giải pháp khắc phục:**  
1. *Adaptive Rate Limiting & Concurrency Control:* Triển khai thuật toán điều tiết đồng thời động (như TCP Vegas / AIMD hoặc Netflix ConcurrencyLimits) tại client để tự động co giãn số lượng request gửi đến `beta` theo độ trễ thực tế, tránh va vào ngưỡng rate limit 9 rps.
2. *Chiến lược Degraded Mode thông minh:* Khi phát hiện toàn bộ model lớn đang bị quá tải, chủ động chuyển đổi prompt sang chế độ cô đọng (short context) hoặc chỉ định các tác vụ không quan trọng dùng model rẻ hơn để giảm tải áp lực tính toán lên provider.

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | **0.9120** | **0.9645** | **0.635** | **174** | **0** | **6** | **0.927** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.95 | Ngăn chặn việc trả lời nhầm câu hỏi khác (`other_question`). Ở mức starter 0.80, có tới 138 câu hỏi bị trả nhầm (`wrong_answers: 138`). Ở mức 0.95, `wrong_answers` triệt để về 0 trong khi vẫn duy trì được hit ratio cao (63.5%). Nếu đặt quá cao (0.97), hit ratio tụt xuống 55.2%. |
| ttl_s | 45.0 | Khống chế số lượng câu trả lời lỗi thời (`outdated_answers`). Do biểu phí vận chuyển đổi âm thầm tại t=480s không có webhook, TTL là cơ chế duy nhất phát hiện. Với TTL 45s, `outdated_answers` chỉ là 6 (dưới giới hạn 15). Nếu để TTL 300s, `outdated_answers` tăng vọt lên 95 vi phạm rubric. |
| max_stale_s | 300.0 | Cung cấp khả năng phục vụ câu trả lời cũ khi sự cố xảy ra (*stale-if-error*). Khi toàn bộ provider sập từ t=240..360s, `max_stale_s: 300` đã phục vụ thành công 174 câu trả lời stale, kéo `faq_availability` từ 86.34% (khi max_stale=0) lên **96.45%** (vượt chuẩn 93%). |
| max_entries | 500 | Đủ sức chứa toàn bộ các câu hỏi FAQ phổ biến của hệ thống, sử dụng chính sách giải phóng LRU để tối ưu bộ nhớ. |

**Thí nghiệm:**

1. **Ngưỡng tương đồng (`similarity_threshold`):**  
   - Thử nghiệm với các ngưỡng 0.80 / 0.95 / 0.97:
     + Ngưỡng 0.80: `cache_hit_ratio` = 0.647, nhưng phát sinh **138 câu trả lời sai** (`wrong_answers: 138`, lý do `other_question: 138`), cost = 0.971.
     + Ngưỡng 0.95 (tối ưu): `cache_hit_ratio` = **0.635**, **0 câu sai** (`wrong_answers: 0`), cost = **0.927**.
     + Ngưỡng 0.97: `wrong_answers: 0`, nhưng `cache_hit_ratio` sụt giảm mạnh xuống **0.552**, cost tăng lên 0.984 do nhiều câu hỏi đồng nghĩa bị trượt cache.
   - *Câu bị trả nhầm ở 0.80:* Các câu hỏi có vector nhúng tương đối gần nhau về mặt từ vựng nhưng khác biệt hoàn toàn về ý định (intent), ví dụ: *"Thời gian xử lý hoàn tiền mất bao lâu?"* bị trả nhầm bằng câu trả lời của *"Chính sách điều kiện hoàn tiền thế nào?"*.

2. **Thời gian sống (`ttl_s`):**  
   - Thử nghiệm với các mức TTL 30s / 45s / 60s / 300s:
     + TTL 30s: `outdated_answers` = 2, `faq_availability` = 0.9645, hit ratio = 0.592.
     + TTL 45s: `outdated_answers` = 6, `faq_availability` = 0.9645, hit ratio = 0.635.
     + TTL 60s: `outdated_answers` = 3, `faq_availability` = 0.9645, hit ratio = 0.652.
     + TTL 300s: `outdated_answers` bùng nổ lên **95 câu** (vi phạm tiêu chuẩn max 15).
   - *Vì sao webhook không giúp được gì cho `outdated_answers`:* Webhook chỉ hoạt động khi hệ thống nguồn (CMS) chủ động gửi sự kiện thông báo thay đổi (như đổi chính sách hoàn tiền ở t=150s). Đối với các thay đổi âm thầm từ bên thứ ba (như đơn vị vận chuyển đổi biểu phí ở t=480s mà không báo trước), client không nhận được bất kỳ tín hiệu webhook nào để xóa cache. Do đó, **TTL là cơ chế duy nhất** để định kỳ ép cache revalidate với nguồn chân lý.

3. **Stale-if-error (`max_stale_s`):**  
   - Khi tắt stale (`max_stale_s: 0`): `stale_served` = 0, `faq_availability` tụt xuống **86.34%** (không đạt chuẩn 93%).
   - Khi bật stale (`max_stale_s: 300`): đã có **174 câu trả lời cũ được phục vụ an toàn** trong giai đoạn mọi provider sập từ t=240..360s, giúp `faq_availability` đạt **96.45%**.

**Trả lời câu hỏi lý thuyết:**

- *Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") KHÔNG được cache, kể cả khi cache theo từng user?*  
  1. **Tính biến động dữ liệu:** Dữ liệu cá nhân (trạng thái đơn hàng, số dư tài khoản, điểm thưởng) thay đổi theo thời gian thực theo từng sự kiện trong hệ thống logistics/kho vận. Câu trả lời được cache sẽ trở nên lỗi thời chỉ sau vài phút.  
  2. **Rủi ro rò rỉ dữ liệu (Data Leak / Privacy Violation):** Tìm kiếm bằng semantic search dựa trên cosine similarity. Nếu hai người dùng hỏi câu hỏi cá nhân có cấu trúc tương tự nhau (hoặc cùng một người hỏi về hai đơn hàng khác nhau), việc đối sánh ngữ nghĩa có nguy cơ trả về thông tin đơn hàng của người này cho người khác (`other_user`), vi phạm nghiêm trọng luật bảo vệ dữ liệu cá nhân (GDPR/quy định bảo mật).
- *Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?*  
  Không thể chỉ dựa vào HTTP status code hay availability nhìn từ phía người dùng cuối vì cache đã che giấu sự cố. Ta cần thiết lập cảnh báo dựa trên:  
  1. **Tỷ lệ phục vụ câu trả lời cũ (`stale_served_rate` hoặc metric `stale_served > 0`):** Cảnh báo ngay lập tức khi phát hiện hệ thống chuyển sang chế độ suy giảm (stale-if-error).  
  2. **Tỷ lệ lỗi từ Upstream Provider (Upstream Error Rate / Circuit Breaker State):** Giám sát trực tiếp tỷ lệ lỗi 5xx/529 tại tầng kết nối tới provider hoặc cảnh báo khi Circuit Breaker của primary provider chuyển sang trạng thái OPEN. Nhờ đó, đội on-call được đánh thức để khắc phục sự cố provider trước khi hết thời gian `max_stale_s` của cache.
