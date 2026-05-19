# Phân tích yêu cầu — vai Provider

- Cặp đàm phán: aivision-camerastream
- Product: A
- Provider service: A2 (CameraStream)
- Consumer service: A4 (AI vision)
- Người viết: Vũ Bích Hợp
- Ngày: 19/05/2026

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `DetectionResult` | Kết quả phân tích bóc tách từ model AI (chứa class, bounding box, độ tin cậy).| detectionId, timestamp, objects (array), status | riskLevel, processingTimeMs |
| `ModelInfo` | Thông tin về cấu hình và phiên bản model AI đang chạy trên server (VD: YOLOv8, RT-DETR). | modelId, name, version | supportedClasses, inputResolution |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| POST | `/detect` | Nhận frame ảnh từ Camera và thực hiện suy luận (inference) để tìm đối tượng. | Gọi liên tục mỗi khi Camera ở biên (edge) phát hiện có chuyển động (motion). |
| GET | `/detections/{id}` | Lấy lại chi tiết kết quả phân tích của một request trước đó. | Gọi khi Consumer cần tra cứu lại log cảnh báo do Timeout hoặc retry. |
| GET | `/models/info` | Truy xuất thông tin model AI và các nhãn (classes) được hỗ trợ. | Gọi lúc khởi tạo hệ thống để Consumer tự động mapping cấu hình. |

---

## 3. Error case

Tối thiểu 5 case.

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload sai định dạng (thiếu file ảnh, thiếu cameraId, hoặc cấu trúc JSON không hợp lệ). | `Problem` |
| 401 | Thiếu Bearer token | `Problem` |
| 403 | Token hợp lệ nhưng không có quyền | `Problem` |
| 404 | Resource không tồn tại | `Problem` |
| 413 | Xung đột nghiệp vụ | `Problem` |
| 422 | Dữ liệu đúng JSON nhưng vi phạm nghiệp vụ | `Problem` |
| 429 | Gửi quá nhiều request cùng lúc, vượt quá số lượng frame GPU có thể xử lý. | `Problem` |


---

## 4. Giả định bổ sung

Ghi rõ những điểm user story chưa nói nhưng Provider cần giả định.

- Giả định 1: Định dạng payload gửi ảnh sẽ dùng chuẩn JSON với chuỗi base64 (application/json) thay vì multipart/form-data để dễ đóng gói kèm các siêu dữ liệu như cameraId, timestamp.
- Giả định 2: Consumer (Camera Stream) đã cấu hình lọc chuyển động (motion filter), chỉ gửi ảnh khi có biến động chứ không stream toàn bộ 30fps/60fps tĩnh lên để tránh sập server AI.
- Giả định 3: Tọa độ bounding box trả về sẽ theo chuẩn [x_min, y_min, x_max, y_max] tương đối (từ 0.0 đến 1.0) để Consumer dễ dàng scale lên bất kỳ độ phân giải màn hình nào.

---

## 5. Câu hỏi cho Consumer

1. Consumer kỳ vọng thời gian phản hồi (Timeout) tối đa cho mỗi request POST /detect là bao nhiêu mili-giây để Provider tính toán việc resize ảnh và cấu hình batch size cho model?

2. Trong lúc cao điểm, lượng request tối đa (Burst Rate) mà Camera Stream có thể đẩy sang cùng 1 giây là bao nhiêu?

3. Nếu GPU bị quá tải và trả về mã lỗi 429 (Too Many Requests), Consumer sẽ xử lý bằng cách bỏ qua frame đó (Drop) hay đưa vào hàng đợi (Queue) để gửi lại?
---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field không thống nhất | Consumer parse lỗi | Chốt naming trong `openapi.yaml` |
| Payload lớn | Timeout/mock lỗi | Thống nhất content-type và size limit |
