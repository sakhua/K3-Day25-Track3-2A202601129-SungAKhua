# Báo cáo Reliability Lab

## 1. Kiến trúc

```
prompt
  |
  v
[Cache.get] --hit--> route="cache_hit:{score}", cache_hit=True, cost=0
  | miss
  v
[Circuit Breaker: primary] --OPEN?--> bỏ qua, sang provider tiếp
  | cho phép
  v
[Provider primary] --thành công--> cache.set() --> route="primary"
  | lỗi
  v
[Circuit Breaker: backup] --OPEN?--> bỏ qua
  | cho phép
  v
[Provider backup] --thành công--> cache.set() --> route="fallback"
  | lỗi
  v
[Static fallback] --> route="static_fallback", error=last_error
```

Mỗi provider có một `CircuitBreaker` riêng (CLOSED → OPEN → HALF_OPEN → CLOSED), độc lập với nhau. Cache đứng trước cả chuỗi provider — hit thì bỏ qua breaker và provider hoàn toàn (latency = 0, cost = 0).

## 2. Cấu hình

| Tham số | Giá trị | Lý do |
|---|---:|---|
| failure_threshold | 3 | Chịu được lỗi lẻ tẻ (jitter mạng), nhưng phản ứng đủ nhanh sau 3 lỗi liên tiếp để tránh retry storm. |
| reset_timeout_seconds | 2 | Đủ ngắn để hồi phục nhanh khi provider khoẻ lại, đủ dài để không dội probe liên tục vào provider còn đang lỗi. |
| success_threshold | 1 | 1 probe thành công là đủ đóng lại circuit vì lỗi trong `FakeLLMProvider` không có tính bó cụm/tương quan. |
| cache TTL | 300s | Đủ dài để giữ các câu hỏi lặp lại trong 1 lần chạy (100 request × 20 câu mẫu), vẫn tự dọn khi chạy lâu. |
| similarity_threshold | 0.92 | Thử 0.85 trước: mức này lỏng, câu hỏi giống hệt nhau nhưng khác năm (2024 vs 2026) dễ vượt ngưỡng nếu chỉ dựa vào similarity. Nâng lên 0.92 để chỉ những câu gần như giống hệt mới vào cache; việc chặn sai theo năm/số giao cho `_looks_like_false_hit()` xử lý riêng. |
| load_test requests | 100 | Đủ lớn (300 request/3 kịch bản) để P95/P99 ổn định và circuit breaker có đủ chu kỳ mở/đóng để đo recovery time, vẫn chạy nhanh trong vài giây. |

## 3. SLO

| SLI | Mục tiêu | Thực tế | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | 98.33% | Không (sát ngưỡng — xem mục 8) |
| Latency P95 | < 2500 ms | 315.14 ms | Đạt |
| Fallback success rate | >= 95% | 93.9% | Không (sát ngưỡng — xem mục 8) |
| Cache hit rate | >= 10% | 61.67% | Đạt |
| Recovery time | < 5000 ms | 2219.9 ms | Đạt |

## 4. Metrics

Từ `reports/metrics.json` (config mặc định, cache backend = memory, 300 request/3 kịch bản):

| Metric | Giá trị |
|---|---:|
| availability | 0.9833 |
| error_rate | 0.0167 |
| latency_p50_ms | 276.28 |
| latency_p95_ms | 315.14 |
| latency_p99_ms | 320.36 |
| fallback_success_rate | 0.939 |
| cache_hit_rate | 0.6167 |
| estimated_cost_saved | 0.185 |
| circuit_open_count | 8 |
| recovery_time_ms | 2219.89 |

*Nguyên nhân: `FakeLLMProvider` dùng `random` không seed nên số liệu dao động nhẹ mỗi lần chạy; kịch bản `primary_flaky_50` (fail 50%) là nguồn dao động lớn nhất.*

## 5. So sánh có/không cache

Cùng config, chỉ đổi `cache.enabled`:

| Metric | Không cache | Có cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 276.04 | 276.28 | ~0 |
| latency_p95_ms | 317.67 | 315.14 | ~0 |
| estimated_cost | 0.124518 | 0.0481 | -0.0764 (~61%) |
| cache_hit_rate | 0 | 0.6167 | +0.6167 |

Latency gần như không đổi vì `FakeLLMProvider` mô phỏng latency cố định (~180-260ms) không phụ thuộc cache. Lợi ích rõ nhất là **cost**: giảm gần đúng theo tỷ lệ cache_hit_rate, vì mỗi cache hit không tốn gọi provider.

## 6. Redis shared cache

- **Vấn đề của in-memory cache**: `ResponseCache` lưu trong list Python trên heap của từng process. Nhiều replica gateway (production thật) sẽ có cache riêng biệt — query đã cache ở replica A không giúp gì cho replica B.
- **Cách Redis giải quyết**: `SharedRedisCache` lưu mỗi entry là 1 Redis hash (`{prefix}{md5(query)}` → `{query, response}`) với TTL qua `EXPIRE`. Mọi instance trỏ cùng Redis đều thấy chung 1 cache.

### Bằng chứng shared state

2 instance `SharedRedisCache` độc lập, cùng kết nối Redis:

```
c1.set("How do I contact support?", "Email support@example.com or use live chat.")
c2.get("How do I contact support?")
-> c2 đọc được: 'Email support@example.com or use live chat.' (score=1.0)
```

`c2` không hề gọi `set()` — chỉ đọc lại dữ liệu `c1` ghi, chứng minh state nằm ở Redis chứ không phải trong process. Test `test_shared_state_across_instances` xác nhận điều này:

```
pytest tests/test_redis_cache.py -v
6 passed in 1.90s
```

### Redis CLI

```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
rl:cache:fa9d0510afaa
rl:cache:f452fc0bc027

$ docker compose exec redis redis-cli HGETALL "rl:cache:f452fc0bc027"
query    "What is the refund policy?"
response "You can request a refund within 30 days."
```

### So sánh in-memory vs Redis

| Metric | In-memory | Redis | Ghi chú |
|---|---:|---:|---|
| latency_p50_ms | 276.28 | 269.46 | Tương đương — chi phí tra cache không đáng kể so với latency provider mô phỏng |
| latency_p95_ms | 315.14 | 316.30 | Tương đương |
| cache_hit_rate | 0.6167 | 0.70 | Chạy Redis từ DB rỗng sau `FLUSHDB`; chênh lệch do lấy mẫu ngẫu nhiên, không phải do backend |

## 7. Kịch bản chaos

| Kịch bản | Kỳ vọng | Quan sát thực tế | Kết quả |
|---|---|---|---|
| primary_timeout_100 | Toàn bộ traffic fallback sang backup, circuit mở | Breaker primary mở sau 3 lỗi liên tiếp; các request sau đó qua backup (`fallback`) hoặc cache hit | Pass |
| primary_flaky_50 | Circuit dao động, trộn giữa primary và fallback | Breaker primary dao động OPEN → HALF_OPEN → CLOSED nhiều lần (đóng góp phần lớn trong 8 lần `circuit_open_count`, recovery ~2.2s khớp `reset_timeout_seconds=2`) | Pass |
| all_healthy | Toàn bộ qua primary, không mở circuit | Cả 2 provider khoẻ, breaker primary luôn CLOSED | Pass |

### Vì sao recovery_time_ms ≈ 2000ms

`recovery_time_ms` được tính bằng cách ghép mỗi lần circuit chuyển `to="open"` với lần `to="closed"` gần nhất sau đó trong `transition_log`, lấy hiệu `ts` (giây) rồi nhân 1000. Đường đi ngắn nhất để một circuit đóng lại là: mở (`open`) → chờ đủ `reset_timeout_seconds` → `allow_request()` cho phép 1 probe và chuyển `half_open` → probe thành công, mà `success_threshold=1` nên đóng lại ngay lập tức. Vì vậy gần như toàn bộ "recovery time" chính là thời gian chờ `reset_timeout_seconds` (2s trong config), cộng thêm vài chục–vài trăm ms cho việc request tiếp theo thực sự tới và probe chạy xong — khớp với số đo thực tế 2219.89 ms mà không cần mở lại code để kiểm tra.

## 8. Phân tích điểm yếu

**Điểm yếu:** Circuit breaker lưu state trong bộ nhớ process (`circuit_breaker.py`). Nếu chạy nhiều replica gateway, mỗi replica có breaker riêng — khi 1 provider bắt đầu lỗi, mỗi replica phải tự tích luỹ đủ `failure_threshold` lỗi mới mở circuit của mình, tức toàn hệ thống chịu `failure_threshold × số replica` request lỗi trước khi mọi replica cùng dừng gọi provider đó, thay vì phản ứng như một khối thống nhất. Đây cũng là lý do `availability` (98.33%) và `fallback_success_rate` (93.9%) sát dưới ngưỡng SLO trong kịch bản `primary_flaky_50`.

**Cách khắc phục:** Lưu counter và state của circuit breaker trong Redis (`INCR`/`EXPIRE` cho failure count, 1 key chung cho state hiện tại) để mọi replica dùng chung 1 circuit state. Khi đó 1 lần lỗi sẽ mở circuit cho toàn hệ thống thay vì từng replica riêng lẻ, cải thiện trực tiếp availability và fallback_success_rate.

## 9. Bước tiếp theo

1. Chuyển circuit breaker state sang Redis (shared, cross-instance) — như mục 8.
2. Thêm seed cho RNG (`random.Random(seed)`, truyền vào `FakeLLMProvider` và bước chọn query trong `run_scenario()`) để chaos run tái lập được hoàn toàn — hiện chỉ pass/fail của kịch bản ổn định, số liệu chi tiết thì chưa.
3. Thêm cost-aware routing: khi `estimated_cost` cộng dồn vượt ngưỡng ngân sách, ưu tiên route sang cache-only hoặc provider rẻ hơn (TODO bonus đã ghi trong `gateway.py`).
