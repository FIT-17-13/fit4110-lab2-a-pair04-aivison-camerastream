# Biên bản đàm phán hợp đồng API

- **Cặp đàm phán:** Pair 01 (Camera Stream ↔ AI Vision)
- **Product:** A
- **Provider:** A4 AI Vision (Nhóm 2 - WL)
- **Consumer:** A2 Camera Stream (Nhóm 3)
- **Phiên:** v1.1
- **Ngày:** 2026-05-19

---

## Issue #1: Định dạng truyền tải dữ liệu hình ảnh

- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera Stream cần gửi frame ảnh sang AI Vision để nhận diện đối tượng theo thời gian thực.
- **Vấn đề:** Hai bên cần thống nhất định dạng payload để vừa truyền được ảnh, vừa đính kèm metadata như `cameraId`, `timestamp`, đồng thời dễ validate bằng OpenAPI Schema.
- **Đề xuất:** Sử dụng `application/json`, trong đó ảnh được mã hóa thành chuỗi `Base64` (mặc định) hoặc truyền bằng `URL` nếu xử lý bất đồng bộ.
- **Quyết định:** Chấp thuận (Accepted). Thống nhất dùng schema `ImageInput`, mặc định `BASE64`, hỗ trợ thêm `URL` cho các luồng non-realtime.
- **Rationale:** Cả Consumer và Provider đều đã giả định dùng JSON + Base64 trong phần phân tích, nên đây là phương án thống nhất ngay từ đầu.
- **Tác động đến service:** Consumer phải encode ảnh sang Base64 trước khi gửi; Provider parse toàn bộ payload từ một JSON duy nhất.

---

## Issue #2: Giới hạn dung lượng tải trọng (Payload Size)

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera có thể sinh ra ảnh độ phân giải cao, dễ vượt khả năng xử lý của GPU AI server.
- **Vấn đề:** Nếu ảnh quá lớn sẽ làm tăng latency hoặc gây lỗi Out Of Memory (OOM).
- **Đề xuất:** Quy định giới hạn kích thước tối đa cho mỗi request.
- **Quyết định:** Chấp thuận (Accepted). Chốt giới hạn **5MB/request**. Nếu vượt quá sẽ trả về **`413 Payload Too Large`**.
- **Rationale:** Phù hợp với câu hỏi Consumer đã đặt ra cho Provider và giúp ổn định hiệu năng dưới 1 giây/frame.
- **Tác động đến service:** Consumer bắt buộc resize hoặc nén ảnh trước khi gửi.

---

## Issue #3: Chuẩn hóa định dạng Bounding Box

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera Stream cần hiển thị bounding box lên màn hình giám sát.
- **Vấn đề:** Nếu không thống nhất định dạng tọa độ sẽ dễ vẽ sai vị trí.
- **Đề xuất:** Chuẩn hóa một format duy nhất.
- **Quyết định:** Chấp thuận (Accepted). Dùng chuẩn **`[xMin, yMin, xMax, yMax]`** với giá trị **normalized từ `0.0` đến `1.0`**.
- **Rationale:** Trùng khớp với giả định của Provider và yêu cầu của Consumer.
- **Tác động đến service:** Cập nhật schema `BoundingBox` với `minimum: 0.0`, `maximum: 1.0`.

---

## Issue #4: Giá trị `riskLevel` khi không có rủi ro

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** `riskLevel` là trường tùy chọn trong `DetectionResult`.
- **Vấn đề:** Nếu phát hiện đối tượng nhưng không có rủi ro thì nên trả gì.
- **Đề xuất:** Không dùng chuỗi rỗng để tránh nhập nhằng dữ liệu.
- **Quyết định:** Chỉnh sửa đề xuất (Modified). Dùng **`type: [string, "null"]`**, trả về `null` nếu không có rủi ro.
- **Rationale:** Đúng chuẩn JSON Schema/OpenAPI 3.1 hơn dùng `""`.
- **Tác động đến service:** Consumer phải kiểm tra null trước khi parse/hiển thị.

---

## Issue #5: Cơ chế chống quá tải Burst Traffic

- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera chỉ gửi ảnh khi có motion nhưng vẫn có thể tạo burst traffic.
- **Vấn đề:** Lượng request tăng đột biến có thể làm AI service quá tải.
- **Đề xuất:** Áp dụng Rate Limit ở phía Provider.
- **Quyết định:** Chấp thuận (Accepted). Provider trả về **`429 Too Many Requests`**; Consumer xử lý bằng **Queue + Exponential Backoff** hoặc **Drop Frame**.
- **Rationale:** Khớp với phân tích của cả hai bên.
- **Tác động đến service:** Bổ sung response `429` trong OpenAPI; Consumer bổ sung logic retry/backoff.

---

## Issue #6: Lỗi dữ liệu Base64 bị hỏng

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** JSON hợp lệ nhưng chuỗi Base64 giải mã thất bại hoặc file ảnh bị corrupt.
- **Vấn đề:** Chọn status code phù hợp.
- **Đề xuất:** Phân biệt với lỗi JSON cú pháp (`400`).
- **Quyết định:** Chỉnh sửa đề xuất (Modified). Thống nhất dùng **`422 Unprocessable Entity`** kèm `Problem Details`.
- **Rationale:** Đúng với semantic error, phù hợp cả hai bản phân tích.
- **Tác động đến service:** Consumer dễ log đúng lỗi mã hóa thay vì nhầm lỗi mạng.

---

# Chốt hợp đồng v1.1

- **Provider sign-off:** Nguyễn Văn Vinh (Đại diện Nhóm 2 - WL - AI Vision)
- **Consumer sign-off:** Vũ Bích Hợp (Đại diện Nhóm 3 - Camera Stream)
- **Witness (GV/TA):** FIT4110 TA
- **Date:** 2026-05-19