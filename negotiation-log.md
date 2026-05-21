# Biên bản đàm phán hợp đồng API

- **Cặp đàm phán:** Pair 01 (Camera Stream ↔ AI Vision)
- **Product:** A
- **Provider:** A4 AI Vision (Nhóm 2 - WL)
- **Consumer:** A2 Camera Stream (Nhóm 3)
- **Phiên:** v1.0
- **Ngày:** 2026-05-19

---

## Issue #1: Định dạng truyền tải dữ liệu hình ảnh

- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
- **Bối cảnh:** Hệ thống Camera Stream cần gửi liên tục dữ liệu hình ảnh (frame) lên AI Vision qua API để nhận diện đối tượng.
- **Vấn đề:** Consumer muốn gửi ảnh thô qua định dạng `multipart/form-data`, nhưng Provider lo ngại việc này làm khó quá trình đính kèm các dữ liệu metadata (như `cameraId`, `timestamp`) vào cùng một luồng payload, đồng thời khó validate bằng JSON Schema.
- **Đề xuất:** Provider đề xuất sử dụng `application/json` và cấu trúc đa hình (`oneOf`), trong đó hình ảnh được mã hóa sang chuỗi `Base64` hoặc truyền qua `URL`.
- **Quyết định:** Chấp thuận (Accepted). Thống nhất dùng JSON payload với schema `ImageInput`. Mặc định dùng `BASE64` cho xử lý realtime, và `URL` cho các luồng xử lý độ trễ thấp.
- **Rationale:** Giao thức JSON giúp Consumer đóng gói siêu dữ liệu dễ dàng hơn. OpenAPI 3.1.0 hỗ trợ validate JSON Schema chặt chẽ hơn so với form-data.
- **Tác động đến service:** Consumer phải code thêm logic encode ảnh sang Base64 trước khi gọi API. Provider có thể parse toàn bộ dữ liệu chỉ trong một khối JSON duy nhất.

---

## Issue #2: Giới hạn dung lượng tải trọng (Payload Size)

- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera Stream thường ghi hình ở độ phân giải cao để đảm bảo độ sắc nét khi giám sát an ninh khuôn viên.
- **Vấn đề:** Nếu Consumer đẩy ảnh độ phân giải cao (4K) chưa nén, AI server sẽ bị quá tải bộ nhớ VRAM, dẫn đến sập ứng dụng (`OOM`) và làm tăng độ trễ inference.
- **Đề xuất:** Cần quy định giới hạn dung lượng tải trọng (payload size) tối đa cho mỗi lượt gửi Base64.
- **Quyết định:** Chấp thuận (Accepted). Chốt giới hạn kích thước tối đa là 5MB/request. Trả về mã lỗi `413 Payload Too Large` nếu ảnh vượt quá dung lượng.
- **Rationale:** Bảo vệ tính toàn vẹn và hiệu năng của phần cứng (GPU), giữ thời gian xử lý (latency) ổn định dưới 1 giây.
- **Tác động đến service:** Consumer bắt buộc phải thiết lập cơ chế nén ảnh hoặc giảm độ phân giải (resize) xuống (VD: 1080p) tại thiết bị Edge trước khi gọi API.

---

## Issue #3: Chuẩn hóa định dạng Bounding Box

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** AI Vision trả về kết quả phân tích chứa tọa độ khung bao (Bounding Box) để các hệ thống khác vẽ khung nhận diện lên màn hình giám sát.
- **Vấn đề:** Định dạng của tọa độ khung bao trả về chưa rõ ràng (là pixel tuyệt đối hay tỷ lệ tương đối), dẫn đến rủi ro Consumer vẽ sai vị trí đối tượng trên các màn hình có độ phân giải khác nhau.
- **Đề xuất:** Consumer yêu cầu chuẩn hóa định dạng tọa độ thống nhất.
- **Quyết định:** Chấp thuận (Accepted). Thống nhất sử dụng tọa độ tương đối theo cấu trúc `[xMin, yMin, xMax, yMax]`, được chuẩn hóa trong khoảng từ `0.0` đến `1.0`.
- **Rationale:** Tọa độ tương đối (normalized coordinates) giúp Frontend/Consumer scale khung hình chính xác và độc lập với kích thước ảnh gốc.
- **Tác động đến service:** Schema `BoundingBox` được cập nhật ràng buộc `minimum: 0.0` và `maximum: 1.0`. Cả hai bên đều phải ánh xạ công thức tính toán tọa độ theo chuẩn này.

---

## Issue #4: Xử lý giá trị khi không có rủi ro

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** AI Vision phân tích frame ảnh và đánh giá mức độ rủi ro sơ bộ (`riskLevel`) của đối tượng/hành vi.
- **Vấn đề:** Thuộc tính `riskLevel` sẽ mang giá trị gì nếu AI nhận diện được đối tượng (VD: sinh viên đi bộ) nhưng hành vi hoàn toàn bình thường, không cấu thành rủi ro?
- **Đề xuất:** Consumer đề xuất trả về chuỗi rỗng `""` hoặc không trả về trường này trong chuỗi JSON.
- **Quyết định:** Chỉnh sửa đề xuất (Modified). Thống nhất sử dụng Union type của OpenAPI 3.1.0: `type: [string, "null"]`. Hệ thống sẽ trả về `null` thay vì chuỗi rỗng nếu không có rủi ro.
- **Rationale:** Sử dụng `null` biểu thị đúng ngữ nghĩa cấu trúc dữ liệu theo chuẩn JSON Schema 2020-12 thay vì cách lách luật bằng chuỗi rỗng.
- **Tác động đến service:** Consumer phải cập nhật mã nguồn để kiểm tra giá trị `null` khi đọc kết quả `riskLevel`, tránh lỗi `NullPointerException` lúc parsing dữ liệu.

---

## Issue #5: Cơ chế chống quá tải Burst Traffic

- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera Stream được cấu hình để gửi ảnh lên AI Vision liên tục mỗi khi có chuyển động (motion detection).
- **Vấn đề:** Nếu có sự kiện bất thường (ví dụ sinh viên ùa ra giờ tan tầm), Consumer có thể liên tục đẩy hàng nghìn frame/giây vào API khiến AI bị quá tải và sập toàn bộ dịch vụ (Thundering Herd).
- **Đề xuất:** Provider sẽ áp dụng HTTP Rate Limit để bảo vệ API.
- **Quyết định:** Chấp thuận (Accepted). Provider sẽ trả về lỗi `429 Too Many Requests` khi chạm ngưỡng. Consumer cam kết áp dụng thuật toán ngắt quãng (`Exponential Backoff`) để thử lại, hoặc chủ động bỏ qua khung hình (Drop frame).
- **Rationale:** Đảm bảo độ sẵn sàng (liveness) và ổn định của AI service trong tình trạng lượng truy cập đột biến.
- **Tác động đến service:** OpenAPI được cập nhật thêm response `429`. Consumer tốn thêm effort để viết code xử lý hàng đợi (Queue) nội bộ và cơ chế drop frame.

---

## Issue #6: Lỗi dữ liệu Base64 bị hỏng

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** Consumer mã hóa ảnh thành chuỗi Base64 và nhúng vào JSON payload để gửi lên Server.
- **Vấn đề:** Nếu chuỗi Base64 truyền lên đúng cấu trúc JSON (không bị lỗi HTTP 400) nhưng nội dung file ảnh bên trong bị hỏng (corrupted), mô hình AI không thể giải mã thì sẽ dùng status code nào cho hợp lý?
- **Đề xuất:** Dùng mã lỗi `400 Bad Request`.
- **Quyết định:** Chỉnh sửa đề xuất (Modified). Thống nhất sử dụng mã `422 Unprocessable Entity` kèm cấu trúc chuẩn `Problem Details`.
- **Rationale:** Lỗi `422` phản ánh chính xác nhất tình huống này: Server hiểu Content-Type, cú pháp JSON đúng, nhưng không thể xử lý nội dung ngữ nghĩa bên trong khối dữ liệu ảnh. Giúp phân biệt rõ ràng với lỗi sai định dạng JSON thông thường.
- **Tác động đến service:** Bổ sung schema `UnprocessableEntity` vào `openapi.yaml`. Consumer có cơ sở rõ ràng để log lỗi hệ thống mã hóa từ đầu ghi camera thay vì đánh giá sai là do lỗi đường truyền mạng.

---

# Chốt hợp đồng v1.0

- **Provider sign-off:** Nguyễn Văn Vinh (Đại diện Nhóm 2 - WL - AI Vision)
- **Consumer sign-off:** Vũ Bích Hợp (Đại diện nhóm 3 - CameraStream)
- **Witness (GV/TA):** FIT4110 TA
- **Date:** 2026-05-19