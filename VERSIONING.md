# Quy định quản lý phiên bản API - Nhóm A2

Hệ thống Camera Stream áp dụng chặt chẽ chuẩn Semantic Versioning (SemVer) `MAJOR.MINOR.PATCH` để quản lý hợp đồng API:

- **Phiên bản hiện tại:** `v1.0.0`
- **Tăng MAJOR (Ví dụ: v2.0.0):** Khi có các thay đổi lớn làm thay đổi cấu trúc cũ, không tương thích ngược (Breaking Changes) như xóa path, đổi tên trường bắt buộc.
- **Tăng MINOR (Ví dụ: v1.1.0):** Khi bổ sung thêm các tính năng, thêm path mới hoặc thêm các trường tùy chọn (Optional) vào schema mà vẫn đảm bảo tương thích ngược.
- **Tăng PATCH (Ví dụ: v1.0.1):** Khi thực hiện sửa đổi nhỏ không ảnh hưởng đến logic như cập nhật mô tả (description), sửa lỗi chính tả trong tài liệu OpenAPI.