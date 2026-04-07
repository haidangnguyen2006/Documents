# Kiến trúc Backend: API Documentation với OpenAPI (Swagger)

## 1. Lý thuyết cốt lõi (Core Concepts)

### 1.1. Vấn đề "Code một đằng, Docs một nẻo"
Theo cách làm truyền thống, Backend dev thường viết tài liệu API bằng Word, Excel hoặc share Collection Postman cho Frontend team. Vấn đề nảy sinh khi Backend sửa logic (thêm 1 trường bắt buộc) nhưng quên báo lại hoặc quên cập nhật tài liệu. Hệ quả là Frontend gọi API bị lỗi, dẫn đến tranh cãi.

### 1.2. Giải pháp: Code as Documentation
Tư duy của hệ thống Enterprise là **"Mã nguồn chính là Tài liệu"**. Chúng ta sử dụng chuẩn **OpenAPI** (trước đây gọi là Swagger) để tự động hóa:
* Thư viện (SpringDoc) sẽ quét (scan) toàn bộ các Controller và DTO trong code.
* Từ code, nó tự động sinh ra một file JSON mô tả chính xác 100% Request và Response.
* Từ file JSON đó, nó dựng lên một trang Web UI trực quan để Dev và Frontend có thể xem và bấm Test API trực tiếp.

---

## 2. Hướng dẫn Implement

**Bước 1: Cài đặt Dependency**
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.4</version> 
</dependency>
```
Bước 2: Cấu hình Authentication (SecurityConfig.java)
Swagger sinh ra 2 link web (/v3/api-docs và /swagger-ui/**). Cần phải cấu hình Spring Security cho phép tải giao diện này (bằng mọi HttpMethod như GET, OPTIONS):

```java
private final String[] SWAGGER_ENDPOINTS = {"/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html"};

// Trong hàm filterChain:
.requestMatchers(SWAGGER_ENDPOINTS).permitAll()
```
Bước 3: Gắn khóa bảo mật cho Docs (OpenApiConfig.java)
Vì API của dự án yêu cầu JWT Token, ta cần cấu hình để Swagger có chỗ nhập Bearer Token:

```java
@Configuration
@OpenAPIDefinition(
        info = @Info(title = "Identity API", version = "1.0", description = "Tài liệu API"),
        security = { @SecurityRequirement(name = "bearerAuth") }
)
@SecurityScheme(name = "bearerAuth", scheme = "bearer", type = SecuritySchemeType.HTTP, bearerFormat = "JWT", in = SecuritySchemeIn.HEADER)
public class OpenApiConfig { }
```
3. Best Practices & Troubleshooting
Sử dụng Annotation: Gắn @Tag lên Controller và @Operation(summary="...") lên Method để giao diện Swagger hiển thị tiếng Việt có ý nghĩa, thay vì hiện tên hàm khô khan.

Lỗi 401 Unauthorized khi vào Swagger: Do nhầm lẫn trong SecurityConfig, chặn mất method GET của đường dẫn /v3/api-docs.

Lỗi 500 (NoSuchMethodError: ControllerAdviceBean): Đây là lỗi xung đột phiên bản cực kỳ phổ biến. Spring Boot 3.4+ đã xóa một số hàm nội bộ. Nếu dùng SpringDoc bản cũ (< 2.8.x), hệ thống sẽ bị sập khi nó cố quét các hàm bắt lỗi GlobalExceptionHandler. Cách fix là nâng version SpringDoc.

