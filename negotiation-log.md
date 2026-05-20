# Nhật ký đàm phán API (Negotiation Log)

- Cặp đàm phán: Cá nhân tự thực hiện (Giả lập đàm phán Camera Stream - AI Vision)
- Người tham gia: Bùi Đình Phúc (Nhóm A2 - Cân cả 2 vai Provider & Consumer)
- Trạng thái chốt hợp đồng (Sign-off): Đã hoàn thành

---

## Issue 1: Lựa chọn giao thức truyền luồng video trực tuyến (Streaming Protocol)

* **Context:** Consumer (AI Vision) cần nhận luồng hình ảnh trực tiếp từ Provider (Camera) để đưa vào mô hình nhận diện hành vi thời gian thực.
* **Issue:** Hạ tầng Camera gốc hỗ trợ giao thức RTSP (Real-Time Streaming Protocol). Tuy nhiên, Consumer muốn một giao thức chạy mượt mà trực tiếp trên nền tảng Web thông thường và dễ quản lý qua HTTP mà không cần cài thêm plugin nặng dòng lệnh.
* **Proposal:** * *Phương án 1:* Provider giữ nguyên luồng thô RTSP, Consumer tự dựng một Media Server trung gian để convert.
    * *Phương án 2:* Provider tích hợp module đóng gói, tự convert sang luồng **HLS (HTTP Live Streaming)** phân đoạn để trả về URL dạng `.m3u8`.
* **Decision:** Thống nhất chọn phương án 2: Sử dụng giao thức **HLS**.
* **Rationale:** Giúp Consumer giảm tải hạ tầng xử lý convert video, tận dụng được cơ chế cache phân đoạn của HTTP. Độ trễ của HLS cấu hình thấp (Low-Latency HLS) hoàn toàn đáp ứng được nghiệp vụ Campus.
* **Impact:** Provider phải bổ sung thuộc tính `protocol: "HLS"` vào schema của StreamSession.

---

## Issue 2: Định dạng và cấu trúc dữ liệu của các mã định danh (ID)

* **Context:** Hệ thống cần định danh duy nhất cho từng mắt Camera và từng phiên kết nối dữ liệu hình ảnh.
* **Issue:** Ban đầu bên làm Camera muốn dùng ID dạng số nguyên tự tăng (`integer: 1, 2, 3...`) cho đơn giản cơ sở dữ liệu. Nhưng bên làm AI lo ngại khi scale hệ thống lên quy mô lớn toàn trường hoặc khi gộp các chi nhánh, ID số nguyên sẽ bị trùng lặp và dễ bị dò quét bảo mật (Insecure Direct Object References).
* **Proposal:** Đổi toàn bộ các trường ID liên quan sang định dạng chuỗi chuỗi mã hóa **UUIDv4**.
* **Decision:** Đồng ý sử dụng định dạng chuỗi tuân theo chuẩn **UUID** cho tất cả các trường định danh (`cameraId`, `sessionId`).
* **Rationale:** Bảo mật tốt hơn, đảm bảo tính duy nhất trên toàn hệ thống phân tán mà không cần một server tập trung sinh ID.
* **Impact:** Cập nhật kiểu dữ liệu trong file OpenAPI thành `type: string` kèm `format: uuid`.

---

## Issue 3: Cấu trúc dữ liệu phản hồi khi xảy ra lỗi hệ thống (Error Responses)

* **Context:** Khi gọi các API điều khiển camera hoặc bật/tắt stream, hệ thống chắc chắn sẽ có lúc gặp lỗi phần cứng hoặc sai quyền truy cập.
* **Issue:** Consumer muốn cấu trúc lỗi trả về đơn giản chỉ gồm `{ "code": 404, "message": "Not Found" }`. Tuy nhiên, quy chuẩn thiết kế API của nhà trường yêu cầu hệ thống phải cung cấp thông tin chi tiết, có ngữ cảnh và dễ mở rộng khi có nhiều vi phạm nghiệp vụ cùng lúc.
* **Proposal:** Ép buộc sử dụng cấu trúc lỗi tuân thủ chặt chẽ theo đặc tả chuẩn **RFC 7807 (Problem Details)**.
* **Decision:** Đồng ý áp dụng chuẩn **RFC 7807** cho toàn bộ các mã phản hồi lỗi 4xx và 5xx.
* **Rationale:** Đảm bảo tính nhất quán trên toàn bộ Smart Campus, giúp Consumer dễ dàng viết một bộ decode lỗi dùng chung (Global Error Handler) cho tất cả các dịch vụ gọi sang.
* **Impact:** Toàn bộ các mã lỗi 400, 401, 403, 404, 409, 422 trong `openapi.yaml` sẽ trả về chung một `components/schemas/ProblemDetails`.

---

## Issue 4: Xử lý thuộc tính tùy chọn có thể bị trống (Nullable Fields)

* **Context:** Resource `Camera` có trường mô tả chi tiết (`description`) để ghi chú vị trí hoặc thông tin kỹ thuật của mắt cam.
* **Issue:** Ở phiên bản OpenAPI cũ, khi không có mô tả, hệ thống thường bỏ qua trường đó hoặc trả về chuỗi rỗng `""`. Consumer phản ánh rằng việc thiếu trường làm code frontend/backend bị dính lỗi `undefined` hoặc crash ứng dụng nếu không check kỹ.
* **Proposal:** Sử dụng tính năng **Union Type với null** của OpenAPI 3.1 để định nghĩa rõ ràng trường này có thể nhận giá trị null.
* **Decision:** Định nghĩa trường `description` có kiểu dữ liệu là `type: [string, "null"]`.
* **Rationale:** Thừa nhận một cách tường minh trong hợp đồng API rằng dữ liệu này có thể null, bắt buộc Consumer khi viết code sinh ra dữ liệu phải handle trường hợp null này, tránh lỗi sập hệ thống (NullPointerException).
* **Impact:** Khai báo chính xác trong cấu trúc Schema của Camera.

---

## Issue 5: Giới hạn thời gian sống của một phiên Livestream (Stream Session Expiration)

* **Context:** Module AI kết nối vào Camera để quét hình ảnh liên tục.
* **Issue:** Nếu module AI bị crash đột ngột hoặc lập trình viên quên gọi API `DELETE` để tắt luồng, Server Camera sẽ phải giữ kết nối và encode video mãi mãi, gây hiện tượng nghẽn mạng và tốn tài nguyên phần cứng.
* **Proposal:** * *Đề xuất của Camera:* Giới hạn mỗi phiên stream chỉ sống tối đa 30 phút.
    * *Đề xuất của AI:* Như vậy quá ngắn, làm gián đoạn tiến trình giám sát liên tục của họ.
* **Decision:** Thống nhất phiên stream có thời hạn mặc định là **2 tiếng (120 phút)**. Trả về thêm trường `expiresAt` trong dữ liệu phản hồi thành công.
* **Rationale:** Vừa đảm bảo an toàn cho tài nguyên server của Provider, vừa cho phép Consumer chủ động lập lịch tự động tạo mới phiên (Refresh Session) mượt mà trước khi hết hạn.
* **Impact:** Thêm trường `expiresAt` (định dạng `date-time`) vào object `StreamSession`.

---

## Issue 6: Quy chuẩn định dạng dữ liệu ngày tháng thời gian (Date-Time Format)

* **Context:** Các trường liên quan đến thời gian tạo thiết bị `createdAt` hay thời gian hết hạn phiên `expiresAt`.
* **Issue:** Hai bên sử dụng các múi giờ khác nhau khi cấu hình server hệ thống (Ví dụ: Server Camera dùng múi giờ local Việt Nam GMT+7, Server AI chạy Docker dùng múi giờ chuẩn UTC GMT+0) dẫn đến việc lệch giờ khi so sánh thời gian hết hạn của phiên stream.
* **Proposal:** Bắt buộc áp dụng định dạng chuỗi chuẩn quốc tế **ISO 8601**.
* **Decision:** Đồng ý tất cả các trường dữ liệu thời gian truyền qua lại API phải có định dạng đầy đủ dạng: `YYYY-MM-DDTHH:mm:ssZ`.
* **Rationale:** Triệt tiêu hoàn toàn việc hiểu sai múi giờ giữa các hệ thống độc lập kết nối với nhau.
* **Impact:** Thiết lập thuộc tính `format: date-time` đi kèm kiểu string trong OpenAPI.

---

## Xác nhận hợp đồng (Sign-off)

Do bài tập thực hiện dưới hình thức cá nhân tự quản lý cả hai vai trò để tối ưu tiến độ, tôi xin đại diện ký xác nhận thông qua toàn bộ các điều khoản hợp đồng API trên.

* **Đại diện Provider (Camera Stream Service):** Bùi Đình Phúc
* **Đại diện Consumer (AI Vision Service):** Bùi Đình Phúc
* **Ngày chốt hợp đồng:** 20-05-2026