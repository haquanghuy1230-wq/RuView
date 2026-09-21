# RuView ESP32-S3 Project Status

## Date
2026-09-19

## Environment
- OS: Windows
- IDE: VS Code
- ESP-IDF: v6.0.2
- Target: ESP32-S3

## Project
Path:
C:\RuView\RuView\firmware\esp32-csi-node

## Git
- origin: haquanghuy1230-wq/RuView
- upstream: ruvnet/RuView
- Working branch: esp32-s3-setup

## ESP32-S3
- Node ID: 1
- Wi-Fi SSID: Hai Huy Nhan
- ESP32 IP: 192.168.1.87 (có thay đổi và nếu có server thì có thể cố định)
- Wi-Fi channel: 4
- Aggregator IP: 192.168.1.84
- UDP port: 5005

## Verified
- ESP-IDF environment: OK
- ESP32-S3 target: OK
- Build: OK
- Flash: OK
- Wi-Fi connection: OK
- CSI collection: OK
- CSI streaming initialized: OK

## Not yet verified
- RuView aggregator receiving UDP CSI packets
- End-to-end CSI visualization
- Multi-node operation
- Team development workflow

## Cập nhật 2026-09-20

### Đã làm và đã kiểm chứng
- **Build được firmware.** Nguyên nhân build lỗi là `IDF_TARGET=esp32` trong terminal, lệch với `sdkconfig` của project (`esp32s3`), nên `idf.py` từ chối chạy. Sau khi đặt `IDF_TARGET=esp32s3` (chỉ trong phiên terminal), `reconfigure` và `build` đều `exit=0`. Toolchain không hỏng: gcc biên dịch trực tiếp một file C tối thiểu, `exit=0`. Cách đặt cố định `IDF_TARGET` chưa làm.
- **Firmware vừa build:** `esp32-csi-node` app 0.8.12, ESP-IDF v6.0.2, compile time Sep 20 2026 22:10:43, kích thước 0x126420 byte (partition app 2 MB, còn trống khoảng 43%). **Chưa flash bản này.** Board vẫn chạy bản 0.8.12 build ngày Sep 19 2026 17:20:31 (checksum khác, cùng version).
- **Nguồn board:** log trước đó có `Brownout detector was triggered` (board reset liên tục). Sau khi kiểm tra và đổi cáp/cổng USB, log không còn brownout. Cần theo dõi thêm khi thu dữ liệu dài.
- **Mạng:** IP đích trong firmware (192.168.1.84) không còn đúng vì router cấp IP tự động (DHCP). PC hiện là 192.168.1.89, board là 192.168.1.98. Router Viettel, chưa có quyền quản trị nên chưa đặt được DHCP reservation.
- **[CONFIG] Provision NVS:** dùng `provision.py` (chạy `--dry-run` trước) rồi ghi vùng NVS (offset 0x9000, 24 KB) bằng esptool, đặt `target_ip=192.168.1.89`. Không đụng app, bootloader, bảng partition, otadata. Log board xác nhận: `NVS override` đã nạp, `UDP sender initialized: 192.168.1.89:5005`, không còn `sendto ENOMEM`.
- **[EXPERIMENT] Thu UDP:** thu 15 giây phòng yên (`idle_01.pkl`, lưu ngoài repo). Kết quả: 29 gói, 1,9 gói/s, nguồn 192.168.1.98, độ dài gói 60/148/32/48 byte. Đường mạng thông. **Chưa kết luận đã có CSI.**

### Phát hiện chính (đọc `main/csi_collector.c`)
- Chỉ có 1 đến 4 callback CSI mỗi giây, thấp hơn nhiều so với 50 Hz cấu hình (`CONFIG_CSI_SELF_PING_HZ=50`, mức tối đa cho phép).
- Bộ giới hạn tốc độ (gate) **không phải nguyên nhân** (chỉ bỏ bớt khi callback đến dày, trần khoảng 50 Hz). Nút thắt nằm trước gate.
- Callback ping trong firmware là hàm rỗng, nên firmware không biết router có trả lời ping hay không.
- Comment trong code cho biết cơ chế giới hạn tồn tại để tránh crash Wi-Fi khi có quá nhiều ngắt: **không bỏ gate, không tăng tốc độ ping** khi chưa hiểu.

### Giả thuyết (chưa kiểm chứng)
- Nguyên nhân CSI thưa: router không trả lời ping, hoặc bộ lọc khung, hoặc cấu hình CSI.
- Gói 148 byte có thể là CSI thô (header khoảng 20 byte cộng 128 byte I/Q). Giá trị đầu gói `0xC5110006` có thể là magic của một loại gói. Cả hai mới là quan sát, chưa đối chiếu với source.

### Việc tiếp theo
1. Đọc đầu hàm callback và cấu hình CSI trong `csi_collector.c` (các cờ `wifi_csi_config_t`).
2. Kiểm tra router có trả lời ping của board không, và loại khung nào tạo ra callback.
3. Khi CSI đủ dày: đối chiếu magic và cấu trúc gói với source, rồi thu phòng yên và phòng có người di chuyển để so sánh.
4. Đặt cố định `IDF_TARGET` cho terminal ESP-IDF.
5. IP có thể đổi lại: kiểm tra IP của PC đầu mỗi buổi. Cách bền vững là có quyền quản trị router (DHCP reservation) hoặc đặt IP tĩnh trên Windows với địa chỉ ngoài dải DHCP.