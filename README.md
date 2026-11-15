# ESP32-C3-điều khiển-Daikin-FTKC35PVMV
Dưới đây là dự án riêng biệt cho ESP32-C3 điều khiển Daikin FTKC35PVMV qua UART → MAX485 → level shift 3.3V → dây P1/P2 (DIII-NET).
1. Lưu ý phần cứng
thành phần	chi tiết
ESP32-C3 UART	TX = GPIO18, RX = GPIO19 (có thể đổi nếu trùng chân)
MAX485	DE/RE nối chung vào một GPIO ESP32 (ví dụ GPIO21) để chuyển chế độ gửi/nhận
Level shift	3.3V ESP32 ↔ 12V DIII-NET (MAX485 hỗ trợ, cần nguồn 5V/12V riêng cho RS485)
P1/P2	Nối vào extrac port DIII-NET của máy trong Daikin, không nối ngược
GND	Phải nối chung giữa ESP32, MAX485 và mạch level shift.

2. Firmware quét (sniffer) – nạp file này đầu tiên
Mục đích: bấm nút trên điều khiển từ xa IR hoặc hoạt động máy lạnh, xem log ESPHome in ra các byte hex thực tế trên UART.
Tên file: daikin_sniffer.yaml
Cách quét frame:
Nạp file sniffer vào ESP32-C3.
Mở log ESPHome (tab Logs trong web server hoặc esphome logs daikin-ftkc35pvmv-sniffer).
Bấm lần lượt các nút trên remote: Bật, Tắt, Làm mát, Sưởi, Giữ ấm, Tự động, Điều nhiệt 18/25/30, Gió tự động, Gió mạnh, Swing...
Ghi lại từng dãy hex in ra. Một bấm nút thường sinh ra một burst (nhiều byte liền nhau).
Sau khi có đủ frame, sang bước 3.

3. Firmware điều khiển (mẫu sau khi đã có frame)
Tên file: daikin_control.yaml
Thay các mảng {0xXX, ...} bằng frame thực tế bạn đã quét được.

4. Cách lấy frame thực tế và hoàn thiện code
Chạy sniffer → bấm nút remote → copy các dòng RX 0x.. ra file văn bản.
Nhận dạng frame: Một bấm nút thường là một chuỗi byte có đầu/cuối cố định (ví dụ 0xAA ... 0xEE). Nhóm các byte theo burst.
Điền vào file: daikin_control.yaml. Thay {0xXX, 0xXX, 0xXX} bằng mảng byte thực tế cho từng nút (Bật, Tắt, Làm mát, Sưởi, Nhiệt độ, Swing...).
Viết parser nhận (2-way): Trong đoạn while (id(uart_daikin).available()), lắp ráp frame từ các byte đến, kiểm tra byte địa chỉ/lệnh của máy trong, rồi cập nhật id(daikin_power), id(daikin_temp), v.v. và gọi publish_state() để Home Assistant cập nhật ngay lập tức.
Nạp và kiểm tra: Trong Home Assistant sẽ hiện các entity: switch điện, select chế độ/gió, number nhiệt độ, text sensor trạng thái.
