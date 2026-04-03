# SOLID Principles - Nguyên lý Thiết kế Phần mềm

Tài liệu tóm tắt 5 nguyên lý thiết kế hướng đối tượng giúp mã nguồn dễ bảo trì, mở rộng và kiểm thử hơn.

---

## 1. S - Single Responsibility Principle (Nguyên lý Đơn trách nhiệm)

<details>
<summary><b>Chi tiết: Một Class chỉ nên giữ một trách nhiệm duy nhất</b></summary>

### Cốt lõi
Một Class (hoặc method) chỉ nên giữ một trách nhiệm duy nhất, hay nói cách khác là chỉ có "một lý do để thay đổi".

### Vấn đề thực tế (Bad Practice)
Bạn có một class `OrderService`. Trong class này, bạn viết code để: Tính toán tổng tiền, lưu data xuống database, VÀ gửi email xác nhận cho khách hàng.
* **Hậu quả:** Khi team Marketing muốn đổi template email, bạn cũng phải chui vào `OrderService` để sửa đổi. Khi team Sale muốn đổi công thức giảm giá, bạn lại chui vào `OrderService`. Class này trở thành một "đống rác" (God object) và cực kỳ khó viết Unit Test.

### Cách giải quyết (Good Practice)
Tách ra làm 3 phần:
1. **OrderLogicService:** Chỉ xử lý tính toán tiền.
2. **OrderRepository:** Chỉ chịu trách nhiệm lưu xuống DB.
3. **EmailNotificationService:** Chỉ lo việc gửi mail.

> **💡 Điểm ăn tiền:** "Việc áp dụng SRP giúp dự án của em chia nhỏ được các module. Khi có bug ở phần gửi mail, em biết chắc chắn phải vào class nào để sửa mà không sợ làm chết phần tính tiền của hệ thống."
</details>

---

## 2. O - Open/Closed Principle (Nguyên lý Đóng/Mở)

<details>
<summary><b>Chi tiết: Mở để mở rộng, Đóng để sửa đổi</b></summary>

### Cốt lõi
Mở cho việc mở rộng (thêm tính năng mới), nhưng Đóng cho việc sửa đổi (không sửa trực tiếp vào code cũ đang chạy tốt).

### Vấn đề thực tế (Bad Practice)
Bạn viết một hàm tính phí ship trong `ShippingService` dùng hàng loạt vòng lặp `if-else` hoặc `switch-case` cho "Giao tiêu chuẩn", "Giao hỏa tốc". Ngày mai, sếp yêu cầu thêm "Giao quốc tế". Bạn lại phải mở class `ShippingService` ra, chèn thêm 1 cái `else if` nữa.
* **Hậu quả:** Việc chạm vào code cũ đang chạy ổn định có nguy cơ sinh ra bug mới (regression bug) và làm vỡ các test case cũ.

### Cách giải quyết (Good Practice)
Sử dụng **Interface** (thường kết hợp với Design Pattern là **Strategy**).
1. Tạo một interface `IShippingStrategy` có hàm `calculateFee()`.
2. Tạo các class `StandardShipping`, `ExpressShipping` implements interface đó.
3. Khi cần thêm "Giao quốc tế", bạn chỉ việc tạo thêm class `InternationalShipping` implement `IShippingStrategy`. Bạn không hề chạm một dòng code nào vào các class cũ.

</details>

---

## 3. L - Liskov Substitution Principle (Nguyên lý Thay thế Liskov)

<details>
<summary><b>Chi tiết: Class con phải thay thế được hoàn toàn class cha</b></summary>

### Cốt lõi
Class con phải có khả năng thay thế hoàn toàn class cha mà không làm hỏng tính đúng đắn của chương trình. Đừng ép class con làm những việc nó không thể làm.

### Vấn đề thực tế (Bad Practice)
Bạn có class cha `Bird` (Chim) với method `fly()` (bay). Bạn tạo class con `Ostrich` (Đà điểu) kế thừa `Bird`. Nhưng đà điểu không biết bay, nên trong hàm `fly()` của đà điểu, bạn ném ra một Exception: `throw new Exception("Không bay được")`.
* **Hậu quả:** Hệ thống có một vòng lặp gọi tất cả các `Bird` ra để gọi hàm `fly()`. Khi chạy đến con Đà điểu, chương trình bị văng lỗi (Crash). Tính đa hình bị phá vỡ.

### Cách giải quyết (Good Practice)
Cấu trúc lại. Tạo class `Bird` chung. Tạo interface `IFlyable`.
* Con chim sẻ (`Sparrow`) thì kế thừa `Bird` và implements `IFlyable`.
* Con đà điểu (`Ostrich`) chỉ kế thừa `Bird` thôi, không implements `IFlyable`.

> **💡 Điểm ăn tiền:** "Nguyên lý này nhắc nhở em cẩn thận khi dùng Kế thừa (Inheritance). Nếu thấy class con phải override method của cha rồi throw Exception hoặc bỏ trống, tức là hệ thống đang thiết kế sai cấu trúc."
</details>

---

## 4. I - Interface Segregation Principle (Nguyên lý Phân tách Interface)

<details>
<summary><b>Chi tiết: Chia nhỏ Interface thay vì dùng một Interface khổng lồ</b></summary>

### Cốt lõi
Thay vì dùng một Interface khổng lồ (fat interface), hãy chia nhỏ thành nhiều Interface cụ thể. Đừng ép Client (class sử dụng) phải implements những method mà nó không cần đến.

### Vấn đề thực tế (Bad Practice)
Bạn tạo một interface `IMachine` cho các máy móc trong văn phòng gồm 3 hàm: `print()`, `scan()`, `fax()`. Bạn tạo class `BasicPrinter` (Máy in cơ bản). Máy này chỉ in được, nhưng vì nó bắt buộc implements `IMachine`, bạn đành phải để trống 2 hàm `scan()` và `fax()`.

### Cách giải quyết (Good Practice)
Tách ra làm 3 interface: `IPrinter`, `IScanner`, `IFax`.
* `BasicPrinter` chỉ implements `IPrinter`.
* `AdvancedMachine` (Máy đa năng) sẽ implements cả 3.

</details>

---

## 5. D - Dependency Inversion Principle (Nguyên lý Đảo ngược Phụ thuộc)

<details>
<summary><b>Chi tiết: Phụ thuộc vào Abstraction, không phụ thuộc vào Implementation</b></summary>

### Cốt lõi
Các module cấp cao KHÔNG nên phụ thuộc vào các module cấp thấp. Cả hai nên phụ thuộc vào **Abstraction** (Interface/Abstract Class). (Đây chính là tư duy nền tảng sinh ra Spring Framework).

### Vấn đề thực tế (Bad Practice)
Class `UserController` (cấp cao) khởi tạo trực tiếp `EmailService` (cấp thấp) bằng từ khóa `new EmailService()` ở bên trong nó.
* **Hậu quả:** Hai class bị dính chặt vào nhau (Tightly Coupled). Nếu muốn đổi gửi thông báo bằng SMS thay vì Email, bạn phải lôi `UserController` ra viết lại code.

### Cách giải quyết (Good Practice)
1. Tạo interface `INotificationService`. Class `EmailService` và `SmsService` sẽ implement nó.
2. Trong `UserController`, khai báo phụ thuộc vào `INotificationService` thay vì class cụ thể. Ta sẽ truyền implementation cụ thể vào qua Constructor (**Constructor Injection**).

> **💡 Điểm ăn tiền:** "Nguyên lý DIP là tiền đề để hiểu về IoC (Inversion of Control) và DI (Dependency Injection) trong Spring. Nhờ nó mà em không bao giờ phải đi new object thủ công nữa, hệ thống cực kỳ dễ test vì có thể dễ dàng mock (làm giả) các interface."
</details>