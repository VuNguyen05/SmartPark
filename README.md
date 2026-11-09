# SmartPark-IoT: Bãi Đỗ Xe Thông Minh Tích Hợp Đặt Chỗ & Cảnh Báo Khẩn Cấp

> **Đề tài giữa kỳ môn học IoT**  
> **Sinh viên thực hiện:** [Họ và Tên] - [MSSV]  
> **Mã nguồn & Mô phỏng:** PlatformIO | ESP32 | Node-RED | Wokwi  

---

## 1. Đặt Bài Toán & Nhu Cầu Thực Tế

Trong các đô thị hiện đại, việc tìm kiếm chỗ đỗ xe ô tô tại các trung tâm thương mại hay tòa nhà văn phòng gây lãng phí nhiều thời gian và nhiên liệu. Các hệ thống bãi đỗ xe truyền thống gặp phải các hạn chế:
- **Thiếu thông tin thời gian thực:** Người lái xe không biết bãi đỗ còn bao nhiêu vị trí trống trước khi tiến vào.
- **Tình trạng chiếm giữ ô đỗ trái phép:** Không có cơ chế quản lý việc đặt chỗ trước hoặc cảnh báo khi đỗ sai ô đã đặt.
- **Ứng phó sự cố kém:** Khi xảy ra sự cố cháy nổ, hệ thống barrier cố định làm chậm quá trình di chuyển thoát hiểm của phương tiện.

**Mục tiêu dự án:** Xây dựng hệ thống quản lý bãi đỗ xe thông minh ứng dụng ESP32 và MQTT, cho phép giám sát/điều khiển từ xa qua giao diện Node-RED Dashboard, tích hợp tính năng đặt chỗ thông minh và cơ chế xả trạm tự động khi xảy ra khẩn cấp.

---

## 2. Tính Sáng Tạo & Điểm Mới Của Đề Tài

1. **Đặt chỗ trước linh hoạt (Dynamic Reservation System):** Người dùng có thể đặt trước vị trí đỗ qua Web/Phone Dashboard. Hệ thống sẽ kích hoạt đếm ngược thời gian chờ và đổi màu đèn LED chỉ dẫn tại vị trí đỗ.
2. **Phát hiện & Cảnh báo vi phạm (Anti-Poaching Protection):** Nếu một xe khác cố tình đỗ vào vị trí đã được đặt trước mà chưa được xác thực, hệ thống sẽ kích hoạt chuông Buzzer báo động tại chỗ và gửi cảnh báo tức thì về màn hình giám sát trung tâm.
3. **Chế độ Tháo chạy Khẩn cấp (Smart Evacuation Mode):** Tích hợp cảm biến phát hiện cháy/nhiệt độ cao. Khi có sự cố, toàn bộ Barrier vào/ra tự động mở tối đa, đồng thời gửi tín hiệu báo động ưu tiên cao nhất qua MQTT để người quản lý kích hoạt kịch bản di tản.

---

## 3. Sơ Đồ Khối Hệ Thống (System Block Diagram)
+-------------------------------------------------------+
   |                  CẢM BIẾN & ĐẦU VÀO                   |
   |  - HC-SR04 (Cổng vào / Ra)                            |
   |  - IR Sensors (Giám sát ô đỗ Slot 1, Slot 2)         |
   |  - DHT11 / Flame Sensor (Cảnh báo cháy/nhiệt độ)      |
   +---------------------------+---------------------------+
                               |
                               v
   +-------------------------------------------------------+
   |                  VI ĐIỀU KHIỂN TRUNG TÂM              |
   |                     (ESP32 Board)                     |
   +-------------+---------------------------+-------------+
                 |                           |
                 v                           v
   +---------------------------+   +-----------------------+
   |     THIẾT BỊ THỰC THI     |   |   GIAO THỨC TRUYỀN THÔNG  |
   |  - Servo SG90 (Barrier)   |   |        MQTT Broker    |
   |  - LED RGB / Đèn báo slot |   |    (HiveMQ / EMQX)    |
   |  - LCD 1602 I2C           |   +-----------+-----------+
   |  - Buzzer Cảnh báo        |               |
   +---------------------------+               v
                                   +-----------------------+
                                   | NODE-RED DASHBOARD    |
                                   | - Giao diện PC/Mobile |
                                   | - Quản lý đặt chỗ     |
                                   | - Nút điều khiển thủ công|
                                   +-----------------------+
& Sơ Đồ Kết Nối Pin

| Tên linh kiện | Loại | Chân ESP32 kết nối | Chức năng |
| :--- | :--- | :--- | :--- |
| **ESP32 DevKit V1** | Vi điều khiển | - | Xử lý trung tâm & Kết nối Wi-Fi/MQTT |
| **HC-SR04 (Gate IN)**| Cảm biến siêu âm | Trig: 5, Echo: 18 | Phát hiện xe chờ mở cổng vào |
| **HC-SR04 (Gate OUT)**| Cảm biến siêu âm | Trig: 19, Echo: 21 | Phát hiện xe ra để mở cổng ra |
| **IR Sensor Slot 1** | Cảm biến hồng ngoại | Pin 32 | Giám sát xe tại ô đỗ 1 |
| **IR Sensor Slot 2** | Cảm biến hồng ngoại | Pin 33 | Giám sát xe tại ô đỗ 2 |
| **Servo Barrier IN** | Động cơ Servo | Pin 13 | Đóng/Mở thanh chắn cổng vào |
| **Servo Barrier OUT**| Động cơ Servo | Pin 12 | Đóng/Mở thanh chắn cổng ra |
| **Buzzer** | Còi báo | Pin 27 | Phát âm thanh cảnh báo |
| **LCD 1602 I2C** | Màn hình | SDA: 21, SCL: 22 | Hiển thị thông số tại trạm |

---

## 5. Cấu Trúc MQTT Topics

- `smartpark/slots/status` `(Publish)`: Gửi trạng thái real-time của các ô đỗ (`JSON`).
- `smartpark/barrier/cmd` `(Subscribe)`: Nhận lệnh đóng/mở barrier thủ công từ Node-RED.
- `smartpark/reservation` `(Subscribe)`: Nhận lệnh giữ chỗ cho người dùng.
- `smartpark/alert` `(Publish/Subscribe)`: Kênh cảnh báo sự cố khẩn cấp / Đỗ sai vị trí.

---

## 6. Giao Diện Quản Lý Node-RED Dashboard Dự Kiến

Giao diện Dashboard trên máy tính và điện thoại bao gồm:
1. **Sơ đồ bãi xe trực quan (Visual Parking Map):** Hiển thị ô xanh (Trống), ô đỏ (Có xe), ô vàng (Đã đặt trước).
2. **Bộ đếm thời gian thực (Live Analytics):** Số lượng chỗ còn lại, tổng lượt xe trong ngày, ước tính doanh thu.
3. **Bảng điều khiển Trung tâm (Control Panel):** Nút mở barrier cưỡng chế, Nút kích hoạt kịch bản Báo cháy/Thoát hiểm khẩn cấp.

---

## 7. Tài Liệu Tham Khảo

1. **OneButton Library GitHub:** [https://github.com/mathertel/OneButton](https://github.com/mathertel/OneButton)
2. **PubSubClient (MQTT for ESP32):** [https://github.com/knolleary/pubsubclient](https://github.com/knolleary/pubsubclient)
3. **Node-RED Dashboard Documentation:** [https://flows.nodered.org/node/node-red-dashboard](https://flows.nodered.org/node/node-red-dashboard)
4. **Hackaday IoT Projects Reference:** [https://hackaday.com/category/iot-2/](https://hackaday.com/category/iot-2/)
