# Kế hoạch triển khai hệ thống điểm danh (Visual Studio)

Tài liệu này mô tả chi tiết các bước xây dựng ứng dụng điểm danh bằng nhận diện khuôn mặt trong Visual Studio dựa trên kiến trúc đã trình bày ở `system-architecture.md`.

## 1. Thiết lập môi trường

1. Cài đặt **Visual Studio 2022** với workload `.NET desktop development`.
2. Cài thêm các gói mở rộng:
   - `Microsoft.ML` & `Microsoft.ML.OnnxRuntime`
   - `OpenCvSharp4` hoặc `Emgu.CV` cho xử lý ảnh thời gian thực.
   - `CommunityToolkit.Mvvm` để triển khai MVVM trong WPF.
3. Chuẩn bị thư viện `dlib`/`face_recognition` dạng DLL hoặc sử dụng model ONNX (`ArcFace`, `SFace`).

## 2. Cấu trúc dự án & lớp chính

| Project | Mục đích | Thành phần chính |
|---------|----------|------------------|
| `FaceAttendance.Domain` | Khai báo entity & interface | `Student`, `Classroom`, `AttendanceRecord`, `IFaceRecognizer`, `IAttendanceRepository` |
| `FaceAttendance.Infrastructure` | EF Core, truy cập dữ liệu, lưu trữ model | `AppDbContext`, `AttendanceRepository`, `StudentRepository`, `FaceTemplateRepository`, `OnnxFaceRecognizer`, `CameraProvider` |
| `FaceAttendance.Application` | Nghiệp vụ & service | `AttendanceService`, `RecognitionService`, `ReportService`, `ClassService`, DTO/Validators |
| `FaceAttendance.App` | Tầng UI (WPF) | View + ViewModel cho Dashboard, CameraPage, ClassManager, ReportView |
| `FaceAttendance.Tests` | Kiểm thử tự động | xUnit/NUnit test cho service & repository |

## 3. Quy trình điểm danh tự động

1. **Khởi tạo camera**: `CameraProvider` mở kết nối webcam, phát event `FrameCaptured` (24 fps).
2. **Tiền xử lý**: chuyển `Mat` sang `BitmapSource`, chuẩn hóa kích thước.
3. **Nhận diện**:
   - `RecognitionService` gọi `IFaceDetector` (Mediapipe) để lấy bounding box.
   - Cắt & cân bằng sáng, truyền vào `IFaceRecognizer.GetEmbedding()`.
   - So khớp với embeddings lưu trong bảng `FaceTemplates` (SVD, cosine similarity).
4. **Lưu điểm danh**:
   - `AttendanceService.MarkAttendance()` kiểm tra xem đã có bản ghi trong buổi học chưa.
   - Tạo `AttendanceRecord` (CheckInTime, Session, IsLate) và lưu qua `IAttendanceRepository`.
   - Phát event `AttendanceUpdated` để UI hiển thị danh sách mới nhất.
5. **Xử lý ngoại lệ**: nếu confidence < ngưỡng, hiển thị pop-up yêu cầu giáo viên xác nhận.

## 4. Báo cáo & thống kê

- `ReportService.CalculateMonthlySummary(classId, month)` trả về đối tượng `MonthlyAttendanceSummary` gồm:
  - `TotalSessions`
  - `PresentCount`
  - `AbsentCount`
  - `LateCount`
  - `AttendanceRate`
- Tạo biểu đồ cột & heatmap bằng `LiveCharts2`.
- Xuất báo cáo PDF sử dụng `QuestPDF` hoặc `Syncfusion.Pdf`.

## 5. Tính năng quản lý lớp học & học sinh

### Tạo mới lớp học
- Form nhập `ClassName`, `SchoolYear`, `HomeroomTeacher`.
- `ClassService.CreateClass()` gọi repository và phát sự kiện cập nhật danh sách lớp.

### Quản lý học sinh
- Lưới dữ liệu (DataGrid) hiển thị danh sách theo lớp.
- Hỗ trợ import từ Excel (`ClosedXML`): parse file, map cột → entity `Student`.
- Mở modal chụp ảnh mẫu trực tiếp: `CameraProvider.CapturePhoto()` lưu xuống `wwwroot/images/<studentId>.jpg` và tạo embedding mới.

## 6. Đồng bộ dữ liệu

- Module `SyncService` chạy nền (HostedService) gửi bản ghi mới lên server (REST API `POST /api/attendance/sync`).
- Sử dụng `Polly` để retry khi mất mạng.
- Cấu hình thời gian đồng bộ trong `appsettings.json`.

## 7. Bảo mật & phân quyền

- Tích hợp `Microsoft Identity` cho đăng nhập: bảng `AspNetUsers`, `AspNetRoles`.
- Phân quyền `Admin`, `Teacher`, `Inspector`:
  - `Admin`: CRUD lớp, học sinh, cấu hình hệ thống.
  - `Teacher`: điểm danh, xem báo cáo lớp phụ trách.
  - `Inspector`: xem thống kê toàn trường.
- Lưu log thao tác bằng `Serilog`.

## 8. Kiểm thử & triển khai

1. Unit test cho `RecognitionService` (mock embeddings), `AttendanceService` (logic không trùng bản ghi).
2. UI test với `WinAppDriver` (tùy chọn) cho kịch bản quét điểm danh.
3. Tạo pipeline CI (GitHub Actions) build solution + chạy test.
4. Đóng gói bằng `ClickOnce` hoặc `MSIX` để triển khai cho giáo viên.

## 9. Lộ trình phát triển

| Giai đoạn | Thời gian | Hạng mục |
|-----------|-----------|----------|
| Sprint 1 | Tuần 1–2 | Thiết kế DB, tạo solution, hoàn thiện quản lý lớp & học sinh |
| Sprint 2 | Tuần 3–4 | Tích hợp camera, module nhận diện, lưu điểm danh |
| Sprint 3 | Tuần 5 | Báo cáo tháng, biểu đồ, export file |
| Sprint 4 | Tuần 6 | Đồng bộ dữ liệu, phân quyền, tối ưu UX |
| Sprint 5 | Tuần 7 | Kiểm thử, đóng gói, viết tài liệu hướng dẫn |

## 10. Phụ lục: Các thư viện đề xuất

- `OpenCvSharp4`, `Mediapipe.NET`: phát hiện khuôn mặt real-time.
- `Microsoft.ML.OnnxRuntime`: suy luận embedding.
- `Dapper` hoặc `Entity Framework Core`: truy cập SQLite/SQL Server.
- `CommunityToolkit.Mvvm`: hỗ trợ MVVM, RelayCommand.
- `LiveCharts2`, `OxyPlot`: biểu đồ chuyên cần.
- `QuestPDF`, `ClosedXML`: xuất PDF, Excel.
- `Serilog`, `Polly`: logging & retry.

