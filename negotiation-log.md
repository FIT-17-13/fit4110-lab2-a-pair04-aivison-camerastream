# Biên bản đàm phán hợp đồng API

- **Cặp đàm phán:** Pair 01 (Camera Stream ↔ AI Vision)
- **Product:** A
- **Provider:** A4 AI Vision (Nhóm 2 - WL)
- **Consumer:** A2 Camera Stream (Nhóm 3)
<<<<<<< HEAD
- **Phiên:** v1.1
=======
- **Phiên:** v1.0
>>>>>>> fb47a7a3e0e6b4bcda95493aef3268cc795b63a3
- **Ngày:** 2026-05-19

---

## Issue #1: Định dạng truyền tải dữ liệu hình ảnh

- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
<<<<<<< HEAD
- **Bối cảnh:** Camera Stream cần gửi frame ảnh sang AI Vision để nhận diện đối tượng theo thời gian thực.
- **Vấn đề:** Hai bên cần thống nhất định dạng payload để vừa truyền được ảnh, vừa đính kèm metadata như `cameraId`, `timestamp`, đồng thời dễ validate bằng OpenAPI Schema.
- **Đề xuất:** Sử dụng `application/json`, trong đó ảnh được mã hóa thành chuỗi `Base64` (mặc định) hoặc truyền bằng `URL` nếu xử lý bất đồng bộ.
- **Quyết định:** Chấp thuận (Accepted). Thống nhất dùng schema `ImageInput`, mặc định `BASE64`, hỗ trợ thêm `URL` cho các luồng non-realtime.
- **Rationale:** Cả Consumer và Provider đều đã giả định dùng JSON + Base64 trong phần phân tích, nên đây là phương án thống nhất ngay từ đầu.
- **Tác động đến service:** Consumer phải encode ảnh sang Base64 trước khi gửi; Provider parse toàn bộ payload từ một JSON duy nhất.
=======
- **Bối cảnh:** Hệ thống Camera Stream cần gửi liên tục dữ liệu hình ảnh (frame) lên AI Vision qua API để nhận diện đối tượng.
- **Vấn đề:** Consumer muốn gửi ảnh thô qua định dạng `multipart/form-data`, nhưng Provider lo ngại việc này làm khó quá trình đính kèm các dữ liệu metadata (như `cameraId`, `timestamp`) vào cùng một luồng payload, đồng thời khó validate bằng JSON Schema.
- **Đề xuất:** Provider đề xuất sử dụng `application/json` và cấu trúc đa hình (`oneOf`), trong đó hình ảnh được mã hóa sang chuỗi `Base64` hoặc truyền qua `URL`.
- **Quyết định:** Chấp thuận (Accepted). Thống nhất dùng JSON payload với schema `ImageInput`. Mặc định dùng `BASE64` cho xử lý realtime, và `URL` cho các luồng xử lý độ trễ thấp.
- **Rationale:** Giao thức JSON giúp Consumer đóng gói siêu dữ liệu dễ dàng hơn. OpenAPI 3.1.0 hỗ trợ validate JSON Schema chặt chẽ hơn so với form-data.
- **Tác động đến service:** Consumer phải code thêm logic encode ảnh sang Base64 trước khi gọi API. Provider có thể parse toàn bộ dữ liệu chỉ trong một khối JSON duy nhất.
>>>>>>> fb47a7a3e0e6b4bcda95493aef3268cc795b63a3

---

## Issue #2: Giới hạn dung lượng tải trọng (Payload Size)

<<<<<<< HEAD
- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera có thể sinh ra ảnh độ phân giải cao, dễ vượt khả năng xử lý của GPU AI server.
- **Vấn đề:** Nếu ảnh quá lớn sẽ làm tăng latency hoặc gây lỗi Out Of Memory (OOM).
- **Đề xuất:** Quy định giới hạn kích thước tối đa cho mỗi request.
- **Quyết định:** Chấp thuận (Accepted). Chốt giới hạn **5MB/request**. Nếu vượt quá sẽ trả về **`413 Payload Too Large`**.
- **Rationale:** Phù hợp với câu hỏi Consumer đã đặt ra cho Provider và giúp ổn định hiệu năng dưới 1 giây/frame.
- **Tác động đến service:** Consumer bắt buộc resize hoặc nén ảnh trước khi gửi.
=======
- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
- **Bối cảnh:** Camera Stream thường ghi hình ở độ phân giải cao để đảm bảo độ sắc nét khi giám sát an ninh khuôn viên.
- **Vấn đề:** Nếu Consumer đẩy ảnh độ phân giải cao (4K) chưa nén, AI server sẽ bị quá tải bộ nhớ VRAM, dẫn đến sập ứng dụng (`OOM`) và làm tăng độ trễ inference.
- **Đề xuất:** Cần quy định giới hạn dung lượng tải trọng (payload size) tối đa cho mỗi lượt gửi Base64.
- **Quyết định:** Chấp thuận (Accepted). Chốt giới hạn kích thước tối đa là 5MB/request. Trả về mã lỗi `413 Payload Too Large` nếu ảnh vượt quá dung lượng.
- **Rationale:** Bảo vệ tính toàn vẹn và hiệu năng của phần cứng (GPU), giữ thời gian xử lý (latency) ổn định dưới 1 giây.
- **Tác động đến service:** Consumer bắt buộc phải thiết lập cơ chế nén ảnh hoặc giảm độ phân giải (resize) xuống (VD: 1080p) tại thiết bị Edge trước khi gọi API.
>>>>>>> fb47a7a3e0e6b4bcda95493aef3268cc795b63a3

---

## Issue #3: Chuẩn hóa định dạng Bounding Box

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
<<<<<<< HEAD
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
=======
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
>>>>>>> fb47a7a3e0e6b4bcda95493aef3268cc795b63a3

---

## Issue #5: Cơ chế chống quá tải Burst Traffic

- **Người nêu vấn đề:** Provider (AI Vision) - Endpoint: `POST /detect`
<<<<<<< HEAD
- **Bối cảnh:** Camera chỉ gửi ảnh khi có motion nhưng vẫn có thể tạo burst traffic.
- **Vấn đề:** Lượng request tăng đột biến có thể làm AI service quá tải.
- **Đề xuất:** Áp dụng Rate Limit ở phía Provider.
- **Quyết định:** Chấp thuận (Accepted). Provider trả về **`429 Too Many Requests`**; Consumer xử lý bằng **Queue + Exponential Backoff** hoặc **Drop Frame**.
- **Rationale:** Khớp với phân tích của cả hai bên.
- **Tác động đến service:** Bổ sung response `429` trong OpenAPI; Consumer bổ sung logic retry/backoff.
=======
- **Bối cảnh:** Camera Stream được cấu hình để gửi ảnh lên AI Vision liên tục mỗi khi có chuyển động (motion detection).
- **Vấn đề:** Nếu có sự kiện bất thường (ví dụ sinh viên ùa ra giờ tan tầm), Consumer có thể liên tục đẩy hàng nghìn frame/giây vào API khiến AI bị quá tải và sập toàn bộ dịch vụ (Thundering Herd).
- **Đề xuất:** Provider sẽ áp dụng HTTP Rate Limit để bảo vệ API.
- **Quyết định:** Chấp thuận (Accepted). Provider sẽ trả về lỗi `429 Too Many Requests` khi chạm ngưỡng. Consumer cam kết áp dụng thuật toán ngắt quãng (`Exponential Backoff`) để thử lại, hoặc chủ động bỏ qua khung hình (Drop frame).
- **Rationale:** Đảm bảo độ sẵn sàng (liveness) và ổn định của AI service trong tình trạng lượng truy cập đột biến.
- **Tác động đến service:** OpenAPI được cập nhật thêm response `429`. Consumer tốn thêm effort để viết code xử lý hàng đợi (Queue) nội bộ và cơ chế drop frame.
>>>>>>> fb47a7a3e0e6b4bcda95493aef3268cc795b63a3

---

## Issue #6: Lỗi dữ liệu Base64 bị hỏng

- **Người nêu vấn đề:** Consumer (Camera Stream) - Endpoint: `POST /detect`
<<<<<<< HEAD
- **Bối cảnh:** JSON hợp lệ nhưng chuỗi Base64 giải mã thất bại hoặc file ảnh bị corrupt.
- **Vấn đề:** Chọn status code phù hợp.
- **Đề xuất:** Phân biệt với lỗi JSON cú pháp (`400`).
- **Quyết định:** Chỉnh sửa đề xuất (Modified). Thống nhất dùng **`422 Unprocessable Entity`** kèm `Problem Details`.
- **Rationale:** Đúng với semantic error, phù hợp cả hai bản phân tích.
- **Tác động đến service:** Consumer dễ log đúng lỗi mã hóa thay vì nhầm lỗi mạng.
=======
- **Bối cảnh:** Consumer mã hóa ảnh thành chuỗi Base64 và nhúng vào JSON payload để gửi lên Server.
- **Vấn đề:** Nếu chuỗi Base64 truyền lên đúng cấu trúc JSON (không bị lỗi HTTP 400) nhưng nội dung file ảnh bên trong bị hỏng (corrupted), mô hình AI không thể giải mã thì sẽ dùng status code nào cho hợp lý?
- **Đề xuất:** Dùng mã lỗi `400 Bad Request`.
- **Quyết định:** Chỉnh sửa đề xuất (Modified). Thống nhất sử dụng mã `422 Unprocessable Entity` kèm cấu trúc chuẩn `Problem Details`.
- **Rationale:** Lỗi `422` phản ánh chính xác nhất tình huống này: Server hiểu Content-Type, cú pháp JSON đúng, nhưng không thể xử lý nội dung ngữ nghĩa bên trong khối dữ liệu ảnh. Giúp phân biệt rõ ràng với lỗi sai định dạng JSON thông thường.
- **Tác động đến service:** Bổ sung schema `UnprocessableEntity` vào `openapi.yaml`. Consumer có cơ sở rõ ràng để log lỗi hệ thống mã hóa từ đầu ghi camera thay vì đánh giá sai là do lỗi đường truyền mạng.
>>>>>>> fb47a7a3e0e6b4bcda95493aef3268cc795b63a3

---

# Chốt hợp đồng v1.1

- **Provider sign-off:** Nguyễn Văn Vinh (Đại diện Nhóm 2 - WL - AI Vision)
<<<<<<< HEAD
- **Consumer sign-off:** Vũ Bích Hợp (Đại diện Nhóm 3 - Camera Stream)
=======
- **Consumer sign-off:** Vũ Bích Hợp (Đại diện nhóm 3 - CameraStream)
>>>>>>> fb47a7a3e0e6b4bcda95493aef3268cc795b63a3
- **Witness (GV/TA):** FIT4110 TA
- **Date:** 2026-05-19