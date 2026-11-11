# SmartPark-IoT: Bãi Đỗ Xe Thông Minh Tích Hợp Đặt Chỗ & Cảnh Báo Khẩn Cấp

> **Đề tài cuối kỳ môn học: phát triển ứng dụng IoT**  
> **Giảng viên:** Nguyễn Anh Tuấn  
> **Sinh viên thực hiện:** Nguyễn Tuấn Vũ - 23001639  
> **Mã nguồn:** PlatformIO | ESP32 | Node-RED
> 
## 1. Đặt Bài Toán & Nhu Cầu Thực Tế

Trong các đô thị hiện đại, việc tìm kiếm chỗ đỗ xe ô tô tại các trung tâm thương mại hay tòa nhà văn phòng gây lãng phí nhiều thời gian và nhiên liệu. Các hệ thống bãi đỗ xe truyền thống gặp phải những hạn chế chính:
- **Thiếu thông tin thời gian thực:** Người lái xe không biết bãi đỗ còn bao nhiêu vị trí trống trước khi tiến vào bãi.
- **Tình trạng chiếm giữ ô đỗ sai quy định:** Không có cơ chế quản lý việc đặt chỗ trước hoặc cảnh báo khi đỗ sai ô đã đăng ký.
- **Ứng phó sự cố kém:** Khi xảy ra sự cố cháy nổ, hệ thống barrier cố định làm chậm quá trình di chuyển thoát hiểm của phương tiện.

**Mục tiêu dự án:** Xây dựng hệ thống quản lý bãi đỗ xe thông minh ứng dụng vi điều khiển ESP32 và giao thức MQTT, cho phép giám sát/điều khiển từ xa qua giao diện Node-RED Dashboard, tích hợp xác thực thẻ từ RFID ra/vào, cơ chế đặt chỗ linh hoạt và xả trạm tự động khi xảy ra khẩn cấp.

---

## 2. Tính Sáng Tạo & Điểm Mới Của Đề Tài

1. **Xác thực RFID & Quản lý lượt xe hai cổng:** Tách biệt hai trạm quẹt thẻ ra/vào độc lập trên cùng bus SPI. Hỗ trợ đối chiếu thông tin thẻ, lưu trữ thời gian và tự động mở thanh chắn Barrier tương ứng.
2. **Giám sát ô đỗ Siêu âm & Đèn LED KY-009 3 trạng thái:** Sử dụng cảm biến siêu âm HC-SR04 đo khoảng cách chính xác, kết hợp Module LED RGB KY-009 trực quan tại từng ô đỗ:
   - **Xanh lá (Green):** Chỗ trống.
   - **Vàng (Yellow):** Chỗ đã được đặt trước qua Dashboard.
   - **Đỏ (Red):** Chỗ đã có xe đỗ.
3. **Đặt chỗ trước linh hoạt (Dynamic Reservation):** Người dùng có thể đặt chỗ qua Web/Mobile Dashboard. Ô đỗ sẽ kích hoạt đếm ngược thời gian chờ và chuyển LED chỉ dẫn sang màu vàng.
4. **Cảnh báo đỗ sai vị trí (Anti-Poaching Protection):** Khi phát hiện xe cố tình đỗ vào vị trí đã được đặt trước mà chưa xác thực, còi Buzzer tại chỗ sẽ phát báo động và gửi thông báo vi phạm tới máy chủ trung tâm.
5. **Chế độ Xả trạm khẩn cấp (Smart Evacuation Mode):** Tích hợp cảm biến DHT11 giám sát nhiệt độ môi trường. Khi phát hiện nhiệt độ vượt ngưỡng báo cháy ($> 50^\circ\text{C}$), toàn bộ Barrier tự động mở $90^\circ$, còi hú báo động liên tục và gửi tín hiệu ưu tiên cao nhất về Dashboard.

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
                                    | - Báo động và đặt chỗ     |
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

## 5. Giao Diện Quản Lý Node-RED Dashboard

Giao diện điều khiển trung tâm được thiết kế tương thích trên cả máy tính và thiết bị di động với các vùng chức năng:
1. **Sơ đồ bãi xe thời gian thực (Interactive Map):** Hiển thị ô xanh (Trống), ô vàng (Đã đặt trước) và ô đỏ (Có xe).
2. **Bảng quản lý thẻ & Đặt chỗ:** Cho phép nhập UID thẻ đặt trước vị trí đỗ, hủy đặt chỗ khi hết hạn.
3. **Bộ đếm & Thống kê (Analytics Panel):** Hiển thị tổng số chỗ, số chỗ còn trống, đếm lượt xe ra/vào trong ngày.
4. **Hệ thống Điều khiển Khẩn cấp (Emergency Control):** Nút kích hoạt mở cưỡng chế toàn bộ rào chắn và hiển thị cảnh báo nhiệt độ cao từ DHT11.

---

