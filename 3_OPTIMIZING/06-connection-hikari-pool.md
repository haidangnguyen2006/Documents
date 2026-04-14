# Kiến trúc Backend: Tối ưu Connection Pool với HikariCP

## 1. Mục tiêu tài liệu

Tài liệu này đi sâu vào kiến trúc, nguyên lý hoạt động, các tham số tinh chỉnh của HikariCP và phương pháp đo/chẩn đoán để vận hành Connection Pool an toàn trên môi trường Production. Mục tiêu là giúp bạn: tối ưu throughput, tránh connection leak, cấu hình đúng kích thước pool phù hợp với Database và workload thực tế.

## 2. Bản chất cốt lõi (Góc nhìn Architect)

### 2.1. Vấn đề "Mở kết nối" (Connection Overhead)
Mỗi connection tới DB là một chi phí (TCP handshake, auth, memory/cursor allocation...). Mở/đóng kết nối cho mỗi request sẽ làm tăng latency và giảm throughput. Connection Pool là pattern để tái sử dụng các kết nối sẵn có.

### 2.2. HikariCP là gì?
HikariCP là connection pool mặc định của Spring Boot (kể từ các phiên bản gần đây) và được biết đến nhờ hiệu năng cao, tiêu tốn tài nguyên thấp và latency ổn định. Nó tập trung vào 3 điểm:
- Đơn giản, ít overhead
- Thuộc tính cấu hình rõ ràng, mặc định an toàn
- Tích hợp tốt với Micrometer để export metrics

### 2.3. Vòng đời một kết nối trong pool
1. Pool mở một số kết nối khi khởi động (theo minimum-idle).
2. Khi ứng dụng cần DB, nó mượn (borrow) một connection từ pool.
3. Sau khi dùng xong, connection được trả lại (return) vào pool chứ không bị đóng.
4. Hikari quản lý việc validate, tái tạo kết nối lỗi thời (theo max-lifetime) và dọn các connection nhàn rỗi (idle-timeout).

## 3. Các cấu hình quan trọng và ý nghĩa

Đoạn cấu hình mẫu (put vào `application-prod.yaml` hoặc file cấu hình tương ứng):

```yaml
spring:
  datasource:
    url: "${DB_URL:jdbc:postgresql://localhost:5432/identity_service}"
    driver-class-name: org.postgresql.Driver
    username: "${DB_USERNAME:postgres}"
    password: "${DB_PASSWORD:123}"
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 30000     # ms
      idle-timeout: 600000          # ms
      max-lifetime: 1800000        # ms
      pool-name: IdentifyService-DB-Pool
      leak-detection-threshold: 2000 # ms (0 = disabled). Bật cand giúp phát hiện leak.
      initialization-fail-timeout: 1 # ms (0 = never fail; -1 = wait indefinitely). Choose appropriate.
```

Giải thích chi tiết:
- maximum-pool-size: số kết nối tối đa pool cho phép mở cùng lúc. Không nên đặt quá lớn so với sức chịu của DB.
- minimum-idle: số kết nối nhàn rỗi tối thiểu pool cố gắng giữ. Giúp giảm latency khi surge.
- connection-timeout: thời gian tối đa client chờ để mượn connection trước khi ném lỗi Timeout.
- idle-timeout: nếu connection nhàn rỗi quá lâu sẽ bị thu hồi (trừ khi số connection đang nhỏ hơn minimum-idle).
- max-lifetime: vòng đời tối đa của 1 connection. Dùng để tái tạo connection tránh tài nguyên rò rỉ hay socket stale.
- leak-detection-threshold: nếu connection bị mượn quá ngưỡng này mà không trả, Hikari sẽ log stacktrace giúp bạn tìm connection leak.
- initialization-fail-timeout: thời gian Hikari chờ khởi tạo pool. Thông thường để >0 để fail fast nếu DB unreachable.

## 4. Công thức sizing & chiến lược đặt pool size

Không có con số "chuẩn" cho maximum-pool-size — nó phụ thuộc vào: số core CPU của DB server, I/O, tốc độ query, và concurrency từ các app instances.

Một cách tiếp cận an toàn:
1. Tính tổng kết nối DB cần từ tất cả app instances: total_connections = instances * maximum-pool-size
2. Giới hạn này phải nhỏ hơn giới hạn tối đa DB chấp nhận (postgresql.max_connections) trừ đi các connection khác (replication, admin)

Gợi ý từ Postgres docs (simple):
connections_per_server ≈ (core_count * 2) + effective_spindle_count

Ví dụ thực tế:
- DB server: 8 cores, SSD => target per-server ≈ 17-25 connections
- Nếu bạn có 3 app instances, target cho mỗi instance là khoảng 6-8 kết nối.

Nếu cần xử lý hàng nghìn QPS đồng thời, thay vì tăng pool size lên quá cao, cân nhắc:
- Thêm caching (Redis)
- Tối ưu query / thêm chỉ mục
- Scale bằng cách mở thêm application instances hoặc read replicas

## 5. Monitoring & Alerting (quan trọng)

Theo dõi các metric sau (Hikari exposes via Micrometer):
- hikari.connections: tổng số connection hiện tại
- hikari.connections.active: số connection đang dùng
- hikari.connections.idle: connection nhàn rỗi
- hikari.connections.pending: request đang chờ
- hikari.usage: tỷ lệ sử dụng

Alert rules (gợi ý):
- Nếu `connections.pending` > 0 liên tục hơn 30s -> có khả năng pool size quá nhỏ
- Nếu `connections.active` gần bằng `maximum-pool-size` trong dài hạn -> điều chỉnh/scale
- Nếu leak-detection logs xuất hiện -> điều tra ngay

Ví dụ tích hợp Prometheus + Grafana:
1. Bật Micrometer/Prometheus exporter trong Spring Boot
2. Export Hikari metrics (Micrometer auto binds HikariCP)
3. Tạo dashboard hiển thị active/idle/pending và alert nếu pending > 0 trong 1 phút

## 6. Debugging connection leak & slow queries

6.1. Phát hiện leak
- Bật `leak-detection-threshold` (ví dụ 2000 ms). Khi connection bị giữ lâu hơn ngưỡng, Hikari log stacktrace nơi connection được mượn.
- Tìm các chỗ không dùng try-with-resources hoặc quên `close()` trên ResultSet/PreparedStatement/Connection.

6.2. Xác định slow queries
- Bật slow-query log ở PostgreSQL (ví dụ log_min_duration_statement = 200ms) và correlate với thời điểm `connections.active` cao.
- Sử dụng pg_stat_activity để xem những session đang chạy.

6.3. Example: tìm leak trong code
```java
// KHÔNG NÊN
Connection conn = dataSource.getConnection();
// nếu quên conn.close() -> leak

// NÊN DÙNG
try (Connection conn = dataSource.getConnection()) {
    try (PreparedStatement ps = conn.prepareStatement(sql)) {
        // xử lý
    }
}
```

## 7. Các tuỳ chọn nâng cao

- validationTimeout: thời gian tối đa để validate connection
- connectionInitSql: câu lệnh chạy ngay khi mượn connection (thận trọng — có thể tăng latency)
- readOnly / autoCommit: set đúng theo use-case (ví dụ read-only connection có thể tối ưu driver/DB)
- preparedStatementCacheSqlLimit / cachePrepStmts (nếu dùng driver hỗ trợ)

Ví dụ cấu hình thêm:

```yaml
      validation-timeout: 5000
      connection-test-query: SELECT 1
      auto-commit: false
```

Lưu ý: với PostgreSQL driver hiện đại, `connection-test-query` thường không cần (driver supports isValid()). Dùng `connection-test-query` chỉ khi bạn thấy flaky TCP idle connections.

## 8. Best Practices & Anti-patterns

- Không set maximum-pool-size quá lớn chỉ để đáp ứng spike ngắn. Thay vào đó, limit concurrency ở tầng app hoặc queue requests.
- Không disable leak detection trong môi trường dev/prod.
- KHÔNG đặt max-lifetime nhỏ hơn than DB/driver's timeout; chọn slightly smaller than any network/DB idle timeout.
- Fail-fast: để `initialization-fail-timeout` > 0 để app không boot mà không thể kết nối DB.

## 9. Nhiệm vụ nghiệm thu (Checklist thực thi)

1. Thêm cấu hình Hikari vào `application-prod.yaml` theo ví dụ.
2. Khởi động ứng dụng, kiểm tra log có thông báo pool Start completed.
3. Dùng Prometheus/Grafana để xem `hikari.connections.active` và `pending` khi chạy load test.
4. Kích hoạt `leak-detection-threshold` trong môi trường staging. Nếu log leak xuất hiện, fix code (close/try-with-resources).
5. Chạy load test tăng dần (siege/hey/locust) để xác định sizing hợp lý; ghi lại các thông số: QPS, latency p95, DB cpu, connections used.

## 10. Tóm tắt nhanh

HikariCP là lớp phủ kết nối nhẹ, hiệu năng cao. Việc tối ưu pool size phải cân bằng giữa sức chịu của DB và concurrency của app. Luôn monitor metrics, bật leak detection và test bằng load test thực tế trước khi áp cấu hình cho Production.

---

