# Bài làm: Phân tích lỗ hổng trong `RegistrationService`

## 1. Câu 1: Chỉ ra ít nhất 2 lỗ hổng/loi nghiệp vụ

### Lỗ hổng 1: Race condition khi kiểm tra chỗ trống và cập nhật số lượng

Trong đoạn mã sau:

```java
if (section.getEnrolledCount() < section.getCapacity()) {
    section.setEnrolledCount(section.getEnrolledCount() + 1);
    classSectionRepository.save(section);
    ...
    registrationRepository.save(registration);
}
```

Hệ thống đã kiểm tra điều kiện và cập nhật số lượng theo hai bước riêng biệt, nhưng không thực hiện trong cùng một transaction và không khóa dữ liệu. Khi nhiều sinh viên cùng đăng ký một lớp còn ít chỗ, nhiều request có thể cùng đọc được giá trị `enrolledCount` giống nhau.

Ví dụ:
- Lớp còn 1 chỗ, `capacity = 10`, `enrolledCount = 9`.
- Hai request cùng chạy đồng thời.
- Mỗi request đều thấy `9 < 10` và đều tăng lên 10.
- Kết quả: hệ thống có thể cho 2 sinh viên cùng đăng ký dù chỉ còn 1 chỗ.

Hậu quả:
- Vượt quá sức chứa của lớp học.
- Dữ liệu đăng ký không chính xác.
- Có thể gây xung đột, sai lệch lịch học, giảng viên và sinh viên bị ảnh hưởng.

### Lỗ hổng 2: Không kiểm tra sinh viên đã đăng ký lớp này chưa

Đoạn mã không kiểm tra xem `studentId` đã từng đăng ký `classSectionId` trong cùng `semester` chưa. Tức là một sinh viên có thể đăng ký cùng một lớp nhiều lần.

Ví dụ:
- Sinh viên A đăng ký `CS101` lần 1.
- Sinh viên A gửi lại request thứ 2 cùng `studentId` và `classSectionId`.
- Hệ thống vẫn lưu `registrationRepository.save(registration)`.

Hậu quả:
- Một sinh viên có thể đăng ký trùng lặp cùng một lớp.
- Tăng số lượng đăng ký sai.
- Gây khó khăn trong đánh giá, xếp lớp, tính học phí và chấm công.

### Lỗ hổng 3: Tin cậy dữ liệu từ client (`studentId` và `semester`)

Các trường sau được lấy trực tiếp từ request:

```java
registration.setStudentId(request.getStudentId());
registration.setSemester(request.getSemester());
```

Nếu không có kiểm tra quyền và xác thực người dùng hợp lệ, một sinh viên có thể gửi `studentId` của người khác để đăng ký thay cho họ. Nói cách khác, hệ thống đang “tin tưởng” dữ liệu do client gửi.

Hậu quả:
- Sinh viên A đăng ký cho sinh viên B.
- Dữ liệu học phần bị nhầm lẫn, sai quyền truy cập.
- Có thể làm sai lệch báo cáo, kế hoạch học tập và thanh toán.

---

## 2. Câu 2: Đề xuất giải pháp khắc phục cụ thể

### Giải pháp cho lỗ hổng 1: Dùng transaction + locking hoặc atomic update

Nên thực hiện kiểm tra chỗ trống và cập nhật số lượng trong cùng một transaction, ví dụ:

- Dùng `@Transactional` cho phương thức `registerCourse`.
- Sử dụng lock cấp độ hàng/đối tượng (`PESSIMISTIC_WRITE`), hoặc cập nhật theo cách atomic: `UPDATE class_section SET enrolled_count = enrolled_count + 1 WHERE id = ? AND enrolled_count < capacity`.
- Nếu update không thành công thì reject request.

Ví dụ ý tưởng:

```java
@Transactional
public Registration registerCourse(...) {
    // lock row class section
    // kiểm tra capacity + update trong cùng transaction
}
```

Ngoài ra, nên có constraint trong database để đảm bảo không vượt quá `capacity`.

### Giải pháp cho lỗ hổng 2: Kiểm tra trùng đăng ký

Trước khi lưu đăng ký, cần kiểm tra:
- Sinh viên này đã đăng ký lớp này chưa.
- Trong cùng học kỳ này hay không.

Ví dụ:

```java
boolean exists = registrationRepository.existsByStudentIdAndClassSectionIdAndSemester(
    request.getStudentId(), request.getClassSectionId(), request.getSemester()
);
if (exists) {
    throw new RuntimeException("Sinh viên đã đăng ký lớp này rồi");
}
```

Ngoài ra nên thêm unique constraint ở database cho cặp `(student_id, class_section_id, semester)` để tránh trùng lặp ở tầng dữ liệu.

### Giải pháp cho lỗ hổng 3: Xác thực quyền và không tin dữ liệu client

- Không cho client tự do gửi `studentId` của người khác.
- Lấy `studentId` từ thông tin người dùng đang đăng nhập (session/token/JWT).
- Kiểm tra quyền so với `request.getStudentId()`.

Ví dụ:

```java
Long currentStudentId = authService.getCurrentStudentId();
if (!currentStudentId.equals(request.getStudentId())) {
    throw new RuntimeException("Bạn không có quyền đăng ký cho sinh viên khác");
}
```

---

## 3. Câu 3: Vào thời điểm mở cổng đăng ký, nhiều sinh viên cùng đăng ký một lớp còn ít chỗ, lỗ hổng nào dễ bị khai thác nhất?

Lỗ hổng dễ bị khai thác nhất là lỗ hổng 1: race condition / check-then-act.

Lý do:
- Nhiều request cùng lúc truy cập cùng một lớp.
- Mỗi request đều thấy “còn chỗ” trước khi cập nhật xong.
- Sau đó, nhiều đăng ký cùng được lưu và số lượng `enrolledCount` vượt quá `capacity`.

Đây là lỗ hổng rất rõ ràng trong thời điểm mở cổng đăng ký vì số lượng request lớn, đồng thời tăng mạnh, khiến hệ thống dễ bị đăng ký quá số chỗ.


