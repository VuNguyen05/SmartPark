# SmartPark-IoT: Bãi Đỗ Xe Thông Minh Tích Hợp Đặt Chỗ & Cảnh Báo Khẩn Cấp

> **Đề tài môn học phát triển ứng dụng IoT**  
> **Sinh viên thực hiện:** Nguyễn Tuấn Vũ - 23001639  
> **Mã nguồn:** PlatformIO | ESP32 | Node-RED  

---

## 1. Kiến Trúc Phần Cứng Mới

Hệ thống cải tiến cơ chế xác thực ra/vào và giám sát vị trí đỗ bằng các thiết bị:

- **Quản lý Cổng vào/ra (RFID RC522):** Sử dụng công nghệ nhận dạng qua tần số vô tuyến để kiểm soát lượt xe ra/vào. Mỗi thẻ từ tương ứng với một UID người dùng.
- **Giám sát Ô đỗ (HC-SR04):** Sử dụng cảm biến siêu âm xác định chính xác sự có mặt của xe tại từng slot theo ngưỡng khoảng cách thiết lập.
- **Điều khiển Chấp hành (Servo SG90):** Đóng/mở thanh chắn tự động sau khi quẹt thẻ thành công.

**Mục tiêu dự án:** Xây dựng hệ thống quản lý bãi đỗ xe thông minh ứng dụng ESP32 và MQTT, cho phép giám sát/điều khiển từ xa qua giao diện Node-RED Dashboard, tích hợp tính năng đặt chỗ thông minh và cơ chế xả trạm tự động khi xảy ra khẩn cấp.

---

## 2. Tính Sáng Tạo & Điểm Mới Của Đề Tài

1. **Đặt chỗ trước linh hoạt:** Người dùng có thể đặt trước vị trí đỗ qua Web/Phone Dashboard. Hệ thống sẽ kích hoạt đếm ngược thời gian chờ và đổi màu đèn LED chỉ dẫn tại vị trí đỗ.
2. **Phát hiện & Cảnh báo vi phạm:** Nếu một xe khác cố tình đỗ vào vị trí đã được đặt trước mà chưa được xác thực, hệ thống sẽ kích hoạt chuông Buzzer báo động tại chỗ và gửi cảnh báo tức thì về màn hình giám sát trung tâm.
3. **Chế độ Tháo chạy Khẩn cấp:** Tích hợp cảm biến phát hiện cháy/nhiệt độ cao. Khi có sự cố, toàn bộ Barrier vào/ra tự động mở tối đa, đồng thời gửi tín hiệu báo động ưu tiên cao nhất qua MQTT để người quản lý kích hoạt kịch bản di tản.

---

## 3. Sơ Đồ Khối Hệ Thống (System Block Diagram)
    +-------------------------------------------------------+
    |                  ĐẦU VÀO / CẢM BIẾN                   |
    |  - RFID RC522 (Cổng Vào & Cổng Ra - Giao tiếp SPI)    |
    |  - HC-SR04 UltraSonic (Slot 1 & Slot 2)               |
    |  - DHT11 đo nhiệt độ bãi đỗ xe                        |
    +---------------------------+---------------------------+
                                |
                                v
    +-------------------------------------------------------+
    |                  VI ĐIỀU KHIỂN TRUNG TÂM              |
    |                     (ESP32 Devkit V1)                 |
    +-------------+---------------------------+-------------+
                  |                           |
                  v                           v
    +---------------------------+   +---------------------------+
    |     THIẾT BỊ THỰC THI     |   |   GIAO THỨC TRUYỀN THÔNG  |
    |  - 2 x Servo SG90         |   |        MQTT Broker        |
    |  - Màn hình LCD 1602 I2C  |   |                           |
    |  - Buzzer Cảnh báo        |   +-----------+---------------+
    +---------------------------+               |
                                                v
                                    +---------------------------+
                                    | NODE-RED DASHBOARD        |
                                    | - Quản lý UID thẻ         |
                                    | - Hiển thị ô đỗ real-time |
                                    | - Tính tiền & Quản lý     |
                                    +---------------------------+
Sơ Đồ Kết Nối Pin

| Tên linh kiện | Loại | Chân ESP32 kết nối | Chức năng |
| :--- | :--- | :--- | :--- |
| **ESP32 DevKit V1** | Vi điều khiển | - | Bộ xử lý trung tâm & Kết nối Wi-Fi / MQTT |
| **RFID RC522 (Cổng vào)** | Đầu đọc thẻ RFID | `SCK: 18`, `MISO: 19`, `MOSI: 23`, `RST: 4`, **`SDA (SS): 5`** | Đọc thẻ từ xác thực xe vào |
| **RFID RC522 (Cổng ra)** | Đầu đọc thẻ RFID | `SCK: 18`, `MISO: 19`, `MOSI: 23`, `RST: 4`, **`SDA (SS): 15`** | Đọc thẻ từ xác thực xe ra & tính tiền |
| **HC-SR04 (Ô đỗ Slot 1)** | Cảm biến siêu âm | `Trig: 13`, `Echo: 12` | Đo khoảng cách phát hiện xe tại Slot 1 |
| **HC-SR04 (Ô đỗ Slot 2)** | Cảm biến siêu âm | `Trig: 14`, `Echo: 27` | Đo khoảng cách phát hiện xe tại Slot 2 |
| **DHT11** | Cảm biến nhiệt độ & độ ẩm | `Data: 2` | Kích hoạt kịch bản Báo cháy / Xả trạm khẩn cấp khi nhiệt độ cao |
| **KY-009 RGB (Slot 1)** | Module LED RGB | `R: 16`, `G: 17`, `GND: GND`| Báo trạng thái Slot 1 (Xanh lá: Trống / Vàng: Đặt trước / Đỏ: Có xe) |
| **KY-009 RGB (Slot 2)** | Module LED RGB | `R: 33`, `G: 32`, `GND: GND`| Báo trạng thái Slot 2 (Xanh lá: Trống / Vàng: Đặt trước / Đỏ: Có xe) |
| **Servo SG90 IN** | Động cơ Servo | `PWM: 25` | Đóng/Mở thanh chắn cổng vào |
| **Servo SG90 OUT**| Động cơ Servo | `PWM: 26` | Đóng/Mở thanh chắn cổng ra |
| **LCD 1602 I2C** | Màn hình | `SDA: 21`, `SCL: 22` | Hiển thị thông báo, mã thẻ và số chỗ trống |
| **Buzzer** | Còi báo | `Pin 0` | Phản hồi âm thanh khi quẹt thẻ / Báo động khẩn cấp |

---

Thư Viện Hình Ảnh Dự Án

| 1. ESP32 Devkit V1 | 2. RFID RC522  | 3. HC-SR04 | 4. DHT11 |
| :---: | :---: | :---: | :---: |
| <img width="100%" src="https://github.com/user-attachments/assets/6c3218fa-9bf8-40f7-a460-95329e9f7be1"  /> | <img width="100%" src="https://github.com/user-attachments/assets/d34d79f6-f44f-48aa-87a5-9f69e9fb5c1e"  /> | <img width="100%" src="https://github.com/user-attachments/assets/af1ace36-b7ef-4aa8-9838-ff1983bd36ef" /> | <img width="50%" src="https://github.com/user-attachments/assets/dd7a3be6-08e5-4544-a3e7-3cbebb487833" /> |

| 5. Module led RGB | 6. Servo SG90 | 7. LCD 1602 I2C | 8. Buzzer cảnh báo |
| :---: | :---: | :---: | :---: |
| <img width="100%" src="https://github.com/user-attachments/assets/36558aff-7b20-4271-83fc-7265cd71a886" /> | <img width="100%" src="https://github.com/user-attachments/assets/d47ab87e-b3ff-437a-8ab5-c6640e55371b"/> | <img width="100%" src="https://github.com/user-attachments/assets/f18101f2-e105-4c1a-b39e-5cfd38cf8b1e"/> | <img width="100%" src="https://github.com/user-attachments/assets/4630b5b6-5509-4ce8-8792-2f8421c9caf1"/> |


## 4. Cấu Trúc MQTT Topics

- `smartpark/slots/status` `(Publish)`: Gửi trạng thái real-time của các ô đỗ (`JSON`).
- `smartpark/barrier/cmd` `(Subscribe)`: Nhận lệnh đóng/mở barrier thủ công từ Node-RED.
- `smartpark/reservation` `(Subscribe)`: Nhận lệnh giữ chỗ cho người dùng.
- `smartpark/alert` `(Publish/Subscribe)`: Kênh cảnh báo sự cố khẩn cấp / Đỗ sai vị trí.

---

## 5. Giao Diện Quản Lý Node-RED Dashboard Dự Kiến

Giao diện Dashboard trên máy tính và điện thoại bao gồm:
1. **Sơ đồ bãi xe trực quan:** Hiển thị ô xanh (Trống), ô đỏ (Có xe), ô vàng (Đã đặt trước).
2. **Bộ đếm thời gian thực:** Số lượng chỗ còn lại, tổng lượt xe trong ngày, ước tính doanh thu.
3. **Bảng điều khiển Trung tâm:** Nút mở barrier cưỡng chế, Nút kích hoạt kịch bản Báo cháy/Thoát hiểm khẩn cấp.

---

