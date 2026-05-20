# Phân tích yêu cầu — vai Provider

- Cặp đàm phán: Cá nhân tự thực hiện (Giả lập cặp Camera - AI Vision)
- Product: A (Camera Stream)
- Provider service: Camera Stream Service
- Consumer service: AI Vision Processing Service
- Người viết: Bùi Đình Phúc (Nhóm A2)
- Ngày: 20-05-2026

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `Camera` | Đại diện cho một thiết bị Camera IP trong hệ thống Smart Campus. | `id` (UUID), `name` (string), `status` (string: `active`, `inactive`, `error`), `rtspUrl` (string) | `description` (string or null), `location` (string), `createdAt` (string-time) |
| `StreamSession` | Phiên kết nối và truyền luồng livestream từ Camera sang định dạng Web. | `sessionId` (UUID), `cameraId` (UUID), `streamUrl` (string), `protocol` (string: `HLS`, `WebRTC`) | `expiresAt` (string-time), `bitrateKbps` (integer) |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/api/v1/cameras` | Lấy danh sách toàn bộ Camera đang hoạt động trong hệ thống. | Khi ứng dụng AI khởi động để đồng bộ danh sách thiết bị cần quét. |
| GET | `/api/v1/cameras/{cameraId}` | Lấy chi tiết thông tin và cấu hình của một Camera cụ thể. | Khi hệ thống cần kiểm tra trạng thái hoặc thông số kỹ thuật của một mắt cam. |
| POST | `/api/v1/cameras/{cameraId}/stream` | Yêu cầu khởi tạo một phiên Livestream (HLS/WebRTC) cho Camera. | Khi module AI bắt đầu vào tiến trình phân tích hình ảnh/khuôn mặt thời gian thực. |
| DELETE | `/api/v1/cameras/{cameraId}/stream` | Hủy/Tắt phiên truyền luồng Stream đang chạy của Camera. | Khi module AI dừng tiến trình xử lý để giải phóng băng thông và tài nguyên hệ thống. |

---

## 3. Error case

Tối thiểu 5 case (Tuân thủ cấu trúc lỗi chuẩn RFC 7807 Problem Details).

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload truyền lên sai định dạng JSON hoặc thiếu trường bắt buộc khi tạo stream. | `{"type": "https://api.smartcampus.com/errors/bad-request", "title": "Bad Request", "status": 400, "detail": "Missing required field 'protocol' in body.", "instance": "/api/v1/cameras/123/stream"}` |
| 401 | Thiếu Bearer token hoặc token không hợp lệ trong Header `Authorization`. | `{"type": "https://api.smartcampus.com/errors/unauthorized", "title": "Unauthorized", "status": 401, "detail": "Full authentication is required to access this resource.", "instance": "/api/v1/cameras"}` |
| 403 | Token hợp lệ nhưng tài khoản không có quyền quản trị/kết nối Camera. | `{"type": "https://api.smartcampus.com/errors/forbidden", "title": "Forbidden", "status": 403, "detail": "You do not have permission to control camera streams.", "instance": "/api/v1/cameras/123/stream"}` |
| 404 | Không tìm thấy `cameraId` tương ứng trong hệ thống cơ sở dữ liệu. | `{"type": "https://api.smartcampus.com/errors/not-found", "title": "Resource Not Found", "status": 404, "detail": "Camera with ID 8a9b1c-23d4-45e6 không tồn tại.", "instance": "/api/v1/cameras/8a9b1c-23d4-45e6"}` |
| 409 | Thiết bị đang bận hoặc phiên Stream cho Camera này đã tồn tại và đang chạy. | `{"type": "https://api.smartcampus.com/errors/conflict", "title": "Conflict", "status": 409, "detail": "A stream session for this camera is already active.", "instance": "/api/v1/cameras/123/stream"}` |
| 422 | Dữ liệu đúng cấu trúc JSON nhưng giá trị truyền vào không hợp lệ (ví dụ: `protocol` truyền vào là `MP4` thay vì `HLS`). | `{"type": "https://api.smartcampus.com/errors/unprocessable", "title": "Unprocessable Entity", "status": 422, "detail": "Unsupported protocol value. Allowed protocols: ['HLS', 'WebRTC'].", "instance": "/api/v1/cameras/123/stream"}` |

---

## 4. Giả định bổ sung

Ghi rõ những điểm user story chưa nói nhưng Provider cần giả định.

- Giả định 1: Định dạng dữ liệu thời gian trong toàn bộ hệ thống (như `createdAt`, `expiresAt`) bắt buộc phải sử dụng chuẩn ISO 8601 (Dạng định dạng: `YYYY-MM-DDTHH:mm:ssZ`).
- Giả định 2: Hệ thống Camera Stream không lưu trữ dữ liệu video quá khứ, chỉ chịu trách nhiệm chuyển đổi giao thức phần cứng sang luồng live trực tiếp (Real-time). Việc lưu trữ dữ liệu ghi hình (nếu có) thuộc trách nhiệm của bên Storage Service khác.
- Giả định 3: Mỗi phiên khởi tạo Stream Session (`POST`) sẽ tự động hết hạn và tự đóng sau 2 giờ liên tục nếu Consumer không thực hiện cơ chế gửi gói tin giữ kết nối (Keep-Alive) để tránh lãng phí tài nguyên băng thông mạng Campus.

---

## 5. Câu hỏi cho Consumer

1. Bên Consumer (AI Vision) muốn nhận luồng stream trả về dưới dạng cắt nhỏ phân đoạn (HLS - HTTP Live Streaming) hay luồng kết nối trực tiếp độ trễ thấp (WebRTC)?
2. Tần suất gọi API cập nhật trạng thái hoặc quét danh sách Camera của Consumer dự kiến là bao nhiêu lần một phút để hệ thống tính toán ngưỡng giới hạn (Rate Limiting)?
3. Phía Consumer có yêu cầu định dạng độ phân giải cố định (ví dụ: bắt buộc 1080p) hay hệ thống có thể tự động điều chỉnh luồng (Adaptive Bitrate) theo băng thông hiện tại?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field không thống nhất | Consumer parse lỗi | Chốt naming camelCase đồng bộ trong `openapi.yaml`. |
| Payload lớn hoặc rớt gói tin luồng | Gây hiện tượng giật lag, timeout khi mock server / stream thật. | Thống nhất content-type định dạng và giới hạn size tối đa của header. Sử dụng CDN trung gian nếu cần mở rộng luồng. |
| Trễ xử lý khi mở luồng đồng thời | Phản hồi của hàm POST tạo stream bị chậm (hơn 5 giây). | Thực hiện xử lý bất đồng bộ (Async) ở phía hạ tầng phần cứng và trả về trạng thái `202 Accepted` kèm URL kiểm tra tiến trình nếu hệ thống quá tải. |