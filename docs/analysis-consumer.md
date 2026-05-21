# Phân tích yêu cầu — vai Consumer

- Cặp đàm phán: aivision-camerastream
- Product: A
- Consumer service: AI Vision
- Provider service: CameraStream
- Người viết: Nguyễn Văn Vinh
- Ngày: 18/05/2026

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `DetectionResult` |Kết quả phân tích bóc tách từ model AI (chứa bounding box, class...). |detectionId, timestamp, objects (array), confidence |riskLevel, processingTimeMs|
| `ModelInfo` |Thông tin về phiên bản model đang chạy trên server. |modelName, version, status |supportedClasses |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| POST | `/detect` |Gọi khi hệ thống Edge Camera phát hiện có chuyển động (motion detection) nhằm phân tích đối tượng.|200 OK kèm theo schema DetectionResult chứa tọa độ và nhãn.|
| GET | `/health` |Gọi định kỳ (ví dụ 1 phút/lần) bằng job ngầm để kiểm tra xem server AI có đang hoạt động không trước khi đẩy ảnh.|200 OK với status UP.|

---

## 3. Error case Consumer cần xử lý

Tối thiểu 5 case.

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 |Payload gửi đi sai định dạng (JSON hỏng, thiếu field bắt buộc).|Ghi log lỗi (Bad Request), bỏ qua frame ảnh đó không gửi lại.|
| 401 | Thiếu hoặc sai Bearer Token xác thực.|Ngừng gửi request, bật cảnh báo (alert) cho Admin biết để cấu hình lại Token.|
| 413 | Payload Too Large (Dung lượng ảnh quá lớn). | Hạ độ phân giải (resize) hoặc tăng mức nén ảnh rồi retry gửi lại. |
| 422 | Định dạng file ảnh không được AI hỗ trợ (Unprocessable Entity). | Ghi log cảnh báo cấu hình camera sai định dạng, loại bỏ frame. |
| 429 | Gửi quá nhiều ảnh trong 1 giây (Too Many Requests). | Đưa ảnh vào hàng đợi (Queue), delay vài mili-giây rồi mới thử lại (Exponential Backoff). |
| 500 | Lỗi crash từ model phía AI Vision. | Retry 1-2 lần, nếu vẫn lỗi thì tạm dừng gửi và alert cho team AI Vision kiểm tra server.|

---

## 4. Giả định bổ sung

- Giả định 1: Định dạng dữ liệu truyền ảnh đi sẽ dùng chuẩn JSON với thuộc tính chứa chuỗi Base64 (application/json) thay vì upload file vật lý (multipart/form-data) để dễ dàng đính kèm các metadata như cameraId, timestamp. 
- Giả định 2: Tốc độ xử lý của Provider đủ nhanh (dưới 1 giây/frame) để gọi đồng bộ (REST sync). Nếu vượt quá giới hạn timeout, Consumer sẽ ngắt kết nối và bỏ qua kết quả đó.
- Giả định 3: Provider có cơ chế bảo vệ hệ thống bằng Rate Limit, do đó API sẽ trả về lỗi 429 nếu Consumer đẩy ảnh lên quá nhanh trong lúc cao điểm.

---

## 5. Câu hỏi cho Provider

1. Dung lượng tối đa (Max File Size) cho mỗi bức ảnh đẩy qua API /detect là bao nhiêu Megabyte để Consumer thiết lập việc nén ảnh trước khi gửi?
2. Thời gian phản hồi kỳ vọng (Expected Response Time) của AI Vision đối với 1 request /detect trung bình là bao nhiêu để Consumer cấu hình số Timeout hợp lý (Ví dụ: 1s hay 3s)?
3. AI Vision hỗ trợ nhận diện được danh sách các đối tượng (classes) nào? (VD: person, car, motorbike, v.v.) để Consumer biết cách mapping dữ liệu đầu ra?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| AI Vision xử lý chậm gây treo kết nối (Timeout). | Consumer bị tắc nghẽn luồng xử lý do phải chờ kết quả quá lâu, gây tốn tài nguyên. | Hai bên chốt thời gian Timeout tĩnh (VD: 2000ms). Consumer chủ động drop kết nối nếu quá hạn. |
| Định nghĩa field tọa độ (Bounding Box) không khớp nhau. | Consumer parse mảng tọa độ bị sai, vẽ sai vị trí đối tượng lên màn hình giám sát. | Provider cần đặc tả rõ Bounding Box trả về dạng [x_min, y_min, x_max, y_max] hay dạng [x_center, y_center, width, height] trong openapi.yaml. |
| Gửi ảnh dồn dập (Burst Traffic) làm sập AI Server. | Mất kết nối diện rộng, các tính năng thông minh bị tê liệt. | Thống nhất cấu hình HTTP Rate Limit và Consumer phải có cơ chế bỏ qua bớt frame (Frame Dropping) lúc cao điểm. |