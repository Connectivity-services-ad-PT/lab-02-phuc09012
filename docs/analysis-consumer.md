# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán: Cá nhân tự thực hiện (Giả lập cặp Camera - AI Vision)
- Product: A (Camera Stream)
- Provider service: Camera Stream Service
- Consumer service: AI Vision Processing Service
- Người viết: Bùi Đình Phúc (Nhóm A2)
- Ngày: 20-05-2026

---

## 1. Nhu cầu dữ liệu (Data Requirements)

Để phục vụ tính năng nhận diện khuôn mặt, phát hiện đám đông và cảnh báo đột nhập thời gian thực trên Campus, Consumer cần những thông tin sau từ Provider:
- Danh sách tất cả các Camera kèm theo trạng thái hoạt động để biết mắt cam nào sẵn sàng kết nối.
- URL luồng stream động (HLS hoặc WebRTC) có độ trễ thấp và băng thông ổn định.
- Mã định danh duy nhất (`cameraId`) dạng UUID để đồng bộ dữ liệu sự kiện AI với vị trí vật lý của Camera.

---

## 2. Các Action/API mong muốn từ Provider

| Method | Path | Tần suất gọi dự kiến | Mục đích & Bối cảnh |
|---|---|---|---|
| GET | `/api/v1/cameras` | 5 phút / lần hoặc khi hệ thống AI khởi động lại. | Đồng bộ danh sách thiết bị và kiểm tra xem có Camera nào mới được lắp đặt không. |
| GET | `/api/v1/cameras/{cameraId}` | Chỉ gọi khi phát hiện Camera mất kết nối đột ngột. | Kiểm tra chi tiết trạng thái hệ thống và thông số phần cứng của mắt cam bị lỗi. |
| POST | `/api/v1/cameras/{cameraId}/stream` | Gọi ngay khi kích hoạt một luồng phân tích AI mới (On-demand). | Yêu cầu Server Camera bắt đầu encode luồng stream và trả về endpoint kết nối (URL). |
| DELETE | `/api/v1/cameras/{cameraId}/stream` | Gọi khi tắt tiến trình phân tích AI hoặc khi khung giờ giám sát kết thúc. | Báo cho Server Camera tắt luồng, giải phóng băng thông mạng để tránh nghẽn mạch. |

---

## 3. Các ràng buộc phi chức năng (Non-functional Constraints)

- **Độ trễ luồng truyền hình ảnh (Latency):** Phải dưới 2 giây để đảm bảo tính năng nhận diện và cảnh báo đột nhập không bị chậm so với thực tế.
- **Bảo mật kết nối:** Toàn bộ API phải được bảo vệ qua giao thức HTTPS. Token truy cập (`Bearer Token`) phải được Consumer truyền qua Header của mọi request.
- **Tính sẵn sàng (Availability):** API lấy danh sách Camera phải phản hồi nhanh (dưới 500ms) để không làm nghẽn tiến trình khởi động của hệ thống AI.

---

## 4. Giả định bổ sung từ góc nhìn Consumer

- Giả định 1: Provider đã xử lý việc phân quyền ở mức hạ tầng. Nếu một Camera thuộc khu vực bảo mật cao, Provider sẽ chủ động trả về lỗi `403 Forbidden` khi Consumer cố tình tạo luồng stream mà không có quyền.
- Giả định 2: Khi gọi API xóa phiên stream (`DELETE`), nếu phiên đó thực tế đã tắt hoặc không tồn tại, Provider nên trả về code thành công `200 OK` hoặc `204 No Content` thay vì báo lỗi nhằm tối ưu logic xử lý phía Consumer.

---

## 5. Câu hỏi cho Provider

1. Cơ chế phân trang (Pagination) của API `GET /api/v1/cameras` hoạt động ra sao nếu số lượng Camera trên toàn bộ Campus tăng lên hàng nghìn mắt?
2. Định dạng lỗi trả về khi luồng Stream phần cứng bị hỏng đột ngột (nhưng API vẫn sống) sẽ được cấu trúc như thế nào? Phía Provider có chủ động gửi Event cảnh báo không?

---

## 6. Rủi ro tích hợp & Biện pháp giảm thiểu

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| URL luồng livestream bị hết hạn (Expired) giữa chừng. | Tiến trình phân tích AI bị ngắt quãng, mất dữ liệu giám sát. | Thống nhất cấu trúc trả về có trường `expiresAt`. Consumer sẽ chủ động lập lịch tạo lại phiên mới trước khi hết hạn 5 phút. |
| Định dạng dữ liệu lỗi không đồng nhất. | Hệ thống AI bị crash (lỗi crash app) khi parse dữ liệu lỗi từ Server. | Ép buộc áp dụng chặt chẽ chuẩn **RFC 7807 (Problem Details)** cho toàn bộ mã lỗi 4xx/5xx trong file `openapi.yaml`. |
| Server Camera bị quá tải khi bật nhiều luồng stream cùng lúc. | Gây treo hệ thống, HTTP trả về mã 500 hoặc 503. | Áp dụng cơ chế giới hạn tần suất gọi (Rate Limiting) và trả về lỗi `429 Too Many Requests` rõ ràng để Consumer biết và thực hiện cơ chế thử lại sau (Retry with Backoff). |