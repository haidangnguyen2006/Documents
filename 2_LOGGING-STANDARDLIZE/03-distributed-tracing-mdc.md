# Kiến trúc Backend: Truy vết lỗi với MDC Logging (Distributed Tracing)

## 1. Lý thuyết cốt lõi (Core Concepts)

### 1.1. Vấn đề "Biển Log" (Log Spaghetty) trên Production
Spring Boot sử dụng cơ chế xử lý đa luồng (Multi-threading). Khi có 1 user gọi request, 1 Thread (ví dụ: `Thread-1`) sẽ được phân công xử lý.
* Ở máy Local: Chỉ có 1 mình bạn test, log in ra theo thứ tự rất dễ đọc.
* Trên Production: Hàng trăm user gọi cùng lúc, `Thread-1`, `Thread-2`, `Thread-3` cùng nhau in log xen kẽ vào 1 màn hình console. Khi có 1 dòng báo Exception, bạn **hoàn toàn không biết** dòng báo lỗi đó thuộc về request của user nào để mà gỡ lỗi (debug).

### 1.2. Giải pháp: Mapped Diagnostic Context (MDC)
**MDC** là một kỹ thuật đính kèm dữ liệu ngữ cảnh vào Thread hiện tại.
* Ngay khi request vừa chạm vào Server, ta tạo ra một mã ID ngẫu nhiên (gọi là **Trace ID**, ví dụ `[uuid-123]`) và ném nó vào MDC.
* Bộ in log (Logback) được cấu hình để: Cứ in ra bất kỳ câu log nào, hãy tự động lấy cái mã trong MDC dán vào phía trước dòng log.
* Kết quả: Ta có thể `Ctrl + F` tìm cái mã `[uuid-123]` để trích xuất ra toàn bộ vòng đời của đúng request đó, bất chấp hàng ngàn request khác đang chạy song song.

---

## 2. Hướng dẫn Implement

**Bước 1: Chặn Request và gắn thẻ bằng Filter (`MdcLoggingFilter.java`)**
Filter này phải đứng ngoài cùng (chạy đầu tiên):

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class MdcLoggingFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) {
        try {
            String traceId = UUID.randomUUID().toString();
            MDC.put("traceId", traceId); // Gắn mã vào Thread hiện tại
            response.setHeader("X-Trace-Id", traceId); // Trả cho Frontend để tiện báo lỗi
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove("traceId"); // BẮT BUỘC PHẢI XÓA MÃ SAU KHI XONG
        }
    }
}
```
Bước 2: Cấu hình File Log (logback-spring.xml)
Định nghĩa lại biến pattern của Spring Boot, nhúng %X{traceId} vào để nó in ra Console:

```xml
<property name="CONSOLE_LOG_PATTERN" 
          value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%15.15t] %-5level %logger{39} [%X{traceId}] - %m%n"/>
```
3. Best Practices & Troubleshooting
Nạn rò rỉ bộ nhớ (Memory Leak) / Sai dữ liệu: Trong Java, các Thread xử lý web thường được lấy từ một ThreadPool (tái sử dụng). Nếu xử lý xong request mà bạn quên gọi MDC.remove("traceId"), thì khi Thread đó được lấy ra phục vụ request của người tiếp theo, nó vẫn sẽ in ra cái mã Trace ID cũ.

Log lỗi ngầm: File GlobalExceptionHandler.java bắt buộc phải có lệnh log.error("Lỗi: ", exception);. Nếu chỉ return về JSON mà không ghi log, khi server sập (Lỗi 500), Backend Dev sẽ "mù mắt" vì không có dấu vết nào để điều tra.

