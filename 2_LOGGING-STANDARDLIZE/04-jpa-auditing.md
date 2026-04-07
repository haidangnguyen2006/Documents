# Kiến trúc Backend: Tự động ghi nhận dữ liệu với JPA Auditing

## 1. Lý thuyết cốt lõi (Core Concepts)

### 1.1. Vấn đề "Boilerplate code" (Code lặp lại)
Mọi bảng dữ liệu chuẩn Enterprise đều cần 2 trường: `created_at` (Ngày tạo) và `updated_at` (Ngày chỉnh sửa cuối).
Nếu lập trình viên tự gán thủ công bằng tay (gọi `entity.setCreatedAt(LocalDateTime.now())` ở mọi hàm tạo mới), sẽ dẫn đến việc:
1. Code lặp đi lặp lại rất nhàm chán.
2. Dễ dính rủi ro "con người" (Quên gọi hàm set khi update dữ liệu, dẫn đến lịch sử bị sai lệch).

### 1.2. Giải pháp: JPA Auditing (Khía cạnh AOP)
Spring Data JPA cung cấp cơ chế **Auditing**. Nó dựa trên các trình lắng nghe sự kiện (`EntityListeners`) của Hibernate.
* Thay vì Dev tự set tay, Hibernate sẽ "canh chừng" ở tầng dưới cùng. 
* Ngay trước thời điểm một lệnh SQL `INSERT` hoặc `UPDATE` được đẩy xuống Database, Hibernate sẽ tự động chặn lại, lấy thời gian thực của máy chủ và bơm vào các thuộc tính thời gian của Entity.

---

## 2. Hướng dẫn Implement

**Bước 1: Bật tính năng Auditing**
Mở file khởi chạy Spring Boot (file chứa hàm `main`), gắn `@EnableJpaAuditing`:

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application { ... }
```
Bước 2: Sử dụng OOP tạo Base Class (BaseEntity.java)
Tạo một class cha để chứa các trường chung, tránh việc class nào cũng phải lặp lại khai báo:

```java
@Getter
@Setter
@MappedSuperclass // Báo cho Hibernate biết đây là lớp cha, hãy mang thuộc tính của nó gán cho lớp con
@EntityListeners(AuditingEntityListener.class) // Đăng ký bộ lắng nghe
public abstract class BaseEntity {

    @CreatedDate
    @Column(name = "created_at", updatable = false) // Không ai được phép sửa ngày tạo
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
}
```
Bước 3: Áp dụng vào thực tế
Các Entity chỉ việc kế thừa (extends) lại class cha này:

```java
@Entity
public class User extends BaseEntity { ... }
```
3. Best Practices
Database Alignment: Đừng quên rằng JPA Auditing chỉ giúp xử lý trên tầng Java. Bạn vẫn phải viết file SQL Flyway thêm 2 cột created_at và updated_at vào các bảng Database (Nên set default là CURRENT_TIMESTAMP ở tầng DB để nhân đôi sự an toàn).

Phân quyền nâng cao: Ngoài @CreatedDate, JPA Auditing còn có @CreatedBy và @LastModifiedBy. Cùng với SecurityContext, tính năng này có thể tự động ghi nhận luôn ID của người thao tác (Ai là người sửa record này) mà Dev không cần phải tốn công viết code truyền UserID vào Service.

