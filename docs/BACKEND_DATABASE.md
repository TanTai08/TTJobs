# Cấu hình PostgreSQL và Flyway cho TTJobs

## Chạy backend trên Windows

Backend dùng Java 17. Tạo một database PostgreSQL trống và cấp cho tài khoản backend quyền tạo, cập nhật schema cùng quyền đọc ghi dữ liệu. Với backend chạy trực tiếp trên Windows, dùng cổng PostgreSQL công bố trên máy. Bản Docker Compose dùng địa chỉ nội bộ `postgres:5432`.

Trong PowerShell tại thư mục gốc của repo, điền giá trị của database rồi chạy:

```powershell
$env:SPRING_DATASOURCE_URL = 'jdbc:postgresql://127.0.0.1:5432/ttjobs'
$env:SPRING_DATASOURCE_USERNAME = 'ttjobs'
$env:SPRING_DATASOURCE_PASSWORD = '<mật khẩu PostgreSQL>'
$env:JWT_SECRET = '<chuỗi ngẫu nhiên ít nhất 32 ký tự ASCII>'
Set-Location backend
.\mvnw.cmd spring-boot:run
```

Đổi cổng `5432` thành `55432` nếu dùng container local `ttjobs-dev-postgres`. Dùng tài khoản PostgreSQL thực tế của container, chẳng hạn `postgres` trong bản local đó.

## Kiểm tra database

- `GET /actuator/health` trả `200` và `{"status":"UP"}` khi các dịch vụ đã bật đều hoạt động. Health check cho SMTP và Redis đi theo `TTJOBS_EMAIL_ENABLED` và `TTJOBS_REALTIME_REDIS_ENABLED`.
- Flyway tự chạy các migration trong `backend/src/main/resources/db/migration`. Hibernate xác thực schema bằng `ddl-auto=validate`. Truy vấn sau cho biết các migration gần nhất đã áp dụng.

```sql
SELECT installed_rank, version, description, success
FROM flyway_schema_history
ORDER BY installed_rank DESC
LIMIT 5;
```

- Database đã có `flyway_schema_history` sẽ tiếp tục từ phiên bản đã ghi nhận. Với schema có dữ liệu nhưng thiếu lịch sử Flyway, sao lưu và xác minh phiên bản schema trước khi baseline. Chỉ dùng `SPRING_FLYWAY_BASELINE_ON_MIGRATE=true` cho đợt tiếp nhận database cũ nếu đã xác nhận `SPRING_FLYWAY_BASELINE_VERSION` khớp với schema hiện có. Giá trị mặc định của phiên bản baseline là `1`.
- Các công ty do migration tạo để minh họa có trạng thái `PENDING`. Tìm việc công khai theo mặc định chỉ hiển thị tin đang mở, còn hạn và thuộc công ty `VERIFIED`. Dùng công ty đã xác minh hoặc fixture trên database thử nghiệm để kiểm tra kết quả tìm kiếm có dữ liệu.

## Kiểm thử API

| Luồng | Kết quả mong đợi |
| --- | --- |
| `POST /api/auth/login` | `200` kèm JWT khi thông tin hợp lệ, `401` khi sai thông tin và `400` khi dữ liệu gửi lên không hợp lệ. |
| `GET /api/jobs` và `GET /api/jobs/search` | `200` cho truy vấn, lọc địa điểm và phân trang hợp lệ, danh sách rỗng khi không có kết quả, `400` với kích thước trang ngoài giới hạn. |
| `GET /api/saved-jobs` | Chỉ trả danh sách của ứng viên đang đăng nhập. |
| `POST /api/saved-jobs/{jobId}` | `200` khi lưu thành công, `404` khi không tìm thấy việc và `409` khi lưu trùng, kể cả các yêu cầu đồng thời. |
| `DELETE /api/saved-jobs/{jobId}` | `200` khi bỏ lưu thành công và `404` khi việc chưa được lưu bởi ứng viên đó. |

Dùng JWT từ API đăng nhập trong header `Authorization: Bearer <token>` khi gọi API lưu việc. Trước khi chạy `mvnw.cmd test`, trỏ các biến datasource sang database thử nghiệm riêng vì các bài kiểm thử tích hợp ghi dữ liệu.

Đã kiểm tra các luồng trên bằng HTTP với Java 17 và PostgreSQL 17.12. Cả 28 kiểm tra đều đạt. Trong bài thử gửi 8 yêu cầu đồng thời lưu cùng một việc, một yêu cầu trả `200`, bảy yêu cầu trả `409` và chỉ có một bản ghi được lưu. Flyway áp dụng đủ 42 migration trên database trống, xác thực lại khi khởi động lại và dừng khi gặp schema có bảng nhưng thiếu lịch sử Flyway.
