# Biên bản đàm phán hợp đồng API

- Cặp đàm phán: Product B
- Product: B1 IoT Ingestion / B7 Notification
- Provider: B1 IoT Ingestion
- Consumer: B7 Notification
- Phiên: v1.0
- Ngày: 2026-05-17

---

## Issue #1

- Raised by: Consumer
- Endpoint: POST /alerts
- Concern: B7 Notification cần nhận alert đã tạo đầy đủ thông tin để gửi thông báo đa kênh mà không cần gọi thêm endpoint thứ hai.
- Proposal: B1 IoT Ingestion trả về 201 Created với `Location` header và payload theo schema `Alert`.
- Resolution: Accepted
- Rationale: Giảm roundtrip, giúp Notification có đủ dữ liệu để gửi Telegram, email và app message ngay.
- Impact: Rút ngắn thời gian xử lý, giảm tải cho hệ thống, đồng thời đảm bảo consumer được cấp dữ liệu chi tiết.

---

## Issue #2

- Raised by: Provider
- Endpoint: POST /alerts
- Concern: `relatedEventId` không rõ có bắt buộc hay không khi alert phát sinh từ dữ liệu IoT mà chưa liên quan trực tiếp đến event cụ thể.
- Proposal: Giữ `relatedEventId` là optional; chỉ gửi khi có event id, còn không thì omit hoặc gửi `null`.
- Resolution: Accepted
- Rationale: Một số alert từ IoT ingestion không có event ràng buộc rõ, nên contract cần linh hoạt.
- Impact: Tránh lỗi schema với request không có `relatedEventId` và giữ hợp đồng dễ sử dụng cho cả hai bên.

---

## Issue #3

- Raised by: Consumer
- Endpoint: GET /alerts/recent
- Concern: Balance giữa bảo mật và khả năng truy vấn cảnh báo gần đây; nếu thiếu xác thực, Notification có thể nhận dữ liệu nhạy cảm.
- Proposal: Duy trì `bearerAuth` với 401 Unauthorized khi thiếu token và 403 Forbidden khi token hợp lệ nhưng không đủ quyền.
- Resolution: Accepted
- Rationale: Bảo mật endpoint cảnh báo là cần thiết để tránh lộ thông tin sự kiện campus.
- Impact: Giảm rủi ro truy cập trái phép và đảm bảo tính tuân thủ bảo mật của contract.

---

## Issue #4

- Raised by: Provider
- Endpoint: POST /events
- Concern: Schema polymorphism `CampusEvent` với `oneOf` và discriminator `eventType` có thể tạo ra khó khăn cho codegen và validation payload.
- Proposal: Duy trì `eventType` bắt buộc và rõ ràng, đồng thời ghi chú trong docs rằng event payload phải tuân theo subtype tương ứng.
- Resolution: Accepted
- Rationale: Cấu trúc này hỗ trợ mở rộng event trong tương lai đồng thời vẫn xác định được loại event ngay ở request.
- Impact: Giúp consumer và provider thống nhất cách gửi event IoT và cảnh báo, giảm lỗi mapping payload.

---

## Issue #5

- Raised by: Provider
- Endpoint: GET /alerts
- Concern: `cursor` parameter hiện là `type: [string, 'null']`, gây nhầm lẫn SDK/validator khi parse OpenAPI.
- Proposal: Giữ `cursor` là string và chỉ ghi chú rằng giá trị có thể là `null` khi hết dữ liệu.
- Resolution: Modified
- Rationale: Cursor nên được xác định là chuỗi để tối ưu cho tooling và code generation.
- Impact: Tăng khả năng tương thích với codegen và giảm lỗi parse OpenAPI trên client.

---

## Issue #6

- Raised by: Provider
- Endpoint: Global error responses (`components/schemas/Problem` và response examples)
- Concern: Spectral cảnh báo `instance` không hợp lệ với format uri khi dùng giá trị `/alerts` hoặc `/alerts/recent`.
- Proposal: Điều chỉnh example `instance` thành URI đầy đủ như `https://api.campus.local/alerts` hoặc `https://api.campus.local/alerts/recent`.
- Resolution: Accepted
- Rationale: Đảm bảo contract tuân RFC 7807 và giúp Spectral lint sạch.
- Impact: Loại bỏ warning, cải thiện khả năng dùng tooling và tăng tính chuyên nghiệp của error contract.

---

# Chốt hợp đồng v1.0

Provider sign-off: B1 IoT Ingestion Team  
Consumer sign-off: B7 Notification Team  
Witness (GV/TA): FIT4110 TA  
Date: 2026-05-17

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| `instance` format trong example response problem | Hiện contract đang tập trung vào nghiệp vụ alert và notification; chấp nhận tạm để hoàn thành đàm phán | Cập nhật example `instance` thành URI hợp lệ và re-run Spectral ngay sau khi sửa |
