# Kiến trúc Backend: Quản trị phiên bản Database với Flyway

## 1. Lý thuyết cốt lõi (Core Concepts)

### 1.1. Vấn đề của cách làm cũ (Hibernate DDL-Auto)
Khi mới học Spring Boot, chúng ta thường dùng `spring.jpa.hibernate.ddl-auto: update`. Cơ chế này cho phép Hibernate tự động quét các Entity (như `@Entity User`) và tự sinh ra các câu lệnh `CREATE`, `ALTER` dưới database. 
* **Rủi ro trên Production:** Hibernate có thể đoán sai ý đồ của Dev. Ví dụ: khi bạn đổi tên biến từ `firstName` thành `givenName`, Hibernate có thể `DROP` cột `firstName` cũ (gây mất dữ liệu) và tạo cột mới `givenName`. Nó cũng có thể lock (khóa) các bảng hàng triệu dòng trong lúc update.

### 1.2. Giải pháp: Database Migration
**Database Migration** là khái niệm "quản lý phiên bản" (Version Control) dành riêng cho Database, giống hệt như cách Git quản lý code.
* **Flyway** là công cụ migration phổ biến nhất. 
* **Quy tắc:** Bất kỳ thay đổi nào dưới Database (tạo bảng, thêm cột, chèn data) đều phải được viết bằng một file script SQL (`.sql`) và đánh số thứ tự (V1, V2, V3...).

### 1.3. Cách Flyway hoạt động (Under the hood)
1. Khi App khởi động, Flyway sẽ tạo một bảng nội bộ tên là `flyway_schema_history` trong DB.
2. Nó đọc các file SQL trong thư mục `db/migration`, tính toán mã băm (Checksum) của từng file.
3. Chạy file theo thứ tự (V1 -> V2). Chạy xong sẽ lưu lịch sử vào bảng `flyway_schema_history`.
4. Lần khởi động sau, nó sẽ bỏ qua V1, V2 và chỉ chạy file mới (V3).
5. **Cơ chế bảo vệ:** Nếu một Dev lén sửa nội dung file V1 (file đã chạy trước đó), mã Checksum sẽ bị lệch, Flyway sẽ báo lỗi và ngăn App khởi động.

---

## 2. Hướng dẫn Implement

**Bước 1: Cài đặt Dependency**
Trong `pom.xml`, thêm thư viện lõi và driver cho PostgreSQL:
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```
Bước 2: Cấu hình application.yaml / application-prod.yaml
Tắt quyền tự update DB của Hibernate và bật Flyway:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate # CHỈ kiểm tra, KHÔNG tự sửa DB
  flyway:
    enabled: true
    baseline-on-migrate: true # Bắt buộc có nếu DB đã có sẵn dữ liệu từ trước
    locations: classpath:db/migration
```
Bước 3: Viết script SQL
Tạo thư mục src/main/resources/db/migration, thêm file V1__init_database.sql (Lưu ý có 2 dấu gạch dưới). Viết script CREATE TABLE bằng ngôn ngữ SQL chuẩn của PostgreSQL.

3. Best Practices & Troubleshooting (Kinh nghiệm thực chiến)
Quy tắc "Bút sa gà chết": Không bao giờ sửa nội dung file SQL cũ đã được merge vào Git. Nếu muốn sửa DB, tạo file V2__xxx.sql.

Lỗi Checksum Mismatch: Thường gặp ở môi trường Dev do dev vô tình sửa file cũ. Cách fix nhanh ở local là Drop bảng flyway_schema_history hoặc Drop toàn bộ DB chạy lại.

Lỗi ON UPDATE CURRENT_TIMESTAMP: Đây là cú pháp riêng của MySQL. Khi dùng PostgreSQL, cú pháp này sẽ báo lỗi Syntax Error. Trong PostgreSQL, hãy chỉ dùng DEFAULT CURRENT_TIMESTAMP.

