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

## Cập nhật 2026-09-21

### Điều chỉnh so với mục 2026-09-19
- "RuView aggregator receiving UDP CSI packets": đã có bằng chứng gói UDP từ board tới PC (cổng 5005). Việc xác minh CSI thô xem bên dưới.

### Đã kiểm chứng (đọc source + dữ liệu thu)
- **Cấu hình CSI** (`main/csi_collector.c`): bật `lltf_en`, `htltf_en`, `stbc_htltf2_en`, `ltf_merge_en`; tắt `channel_filter_en`, `manu_scale`. CSI do callback của ESP-IDF (`esp_wifi_set_csi_rx_cb`) cung cấp, I/Q được chép từ buffer của driver.
- **Nguồn khung CSI:** firmware ping gateway 50 Hz (`CONFIG_CSI_SELF_PING_HZ=50`, mức tối đa cho phép). Bộ lọc khung là MGMT, nâng lên MGMT+DATA cho board không có màn hình (RuView#893). Callback ping là hàm rỗng nên firmware không biết router có trả lời hay không.
- **Không có đường giả lập trên board:** code mock CSI (`CONFIG_CSI_MOCK_ENABLED`) chỉ dành cho QEMU; log khởi động của board không có dòng `Mock CSI active`.
- **Bảng magic UDP:** `0xC5110001` raw CSI, `...02` vitals, `...03` feature vector, `...04` fused vitals, `...05` compressed CSI, `...06` feature state, `...07` WASM output, `0xC511A110` sync, `0xC5118100` mesh.
- **Header raw CSI (20 byte):** magic, node ID, số anten (cố định 1), số subcarrier, tần số MHz (tính từ kênh), seq, RSSI, noise floor, ppdu_type, cờ; sau đó I/Q thô.
- **Thu 15 giây phòng yên:** 29 gói = 16 feature state (60 B), 5 raw CSI (148 B), 4 vitals (32 B), 4 feature vector (48 B). Tốc độ raw CSI chỉ khoảng 0,33 gói/s.
- **Giải mã 5 gói raw CSI:** node=1, 1 anten, 64 subcarrier, 2427 MHz (kênh 4), seq 321 đến 325 liên tiếp, RSSI -24 (4 gói) và -84 (1 gói) khớp log serial, noise -94, cờ 0x10 (bit ESP-NOW sync). Mỗi gói có đúng 12 subcarrier biên độ 0 (gói 1: chỉ số 0 và 27 đến 37). Biên độ các subcarrier còn lại 10 đến 17, thay đổi trơn giữa các subcarrier kề nhau. Chênh lệch biên độ trung bình giữa các gói liền kề: 0,5 / 1,0 / 4,4 / 3,6 (hai giá trị lớn liên quan gói RSSI -84).

### Diễn giải (giả thuyết, chưa kiểm chứng đầy đủ)
- 12 subcarrier bằng 0 khớp cấu trúc LLTF 20 MHz của 802.11 (52 subcarrier dùng, còn lại là DC và guard band), phù hợp với CSI thật của radio.
- Độ dài 128 B (LLTF) và 384 B (HT-LTF + STBC) có thể tùy loại khung; log từng có `len=384`. Chưa kiểm chứng.
- Gói RSSI -84 có thể do thiết bị phát khác; header không có MAC nguồn nên chưa xác minh.

### Chưa kết luận
- Chưa chứng minh CSI phản ánh chuyển động/vị trí: mới có 5 gói phòng yên, chưa thu có người.
- Tốc độ CSI thô quá thấp cho phát hiện chuyển động; cần tìm nguyên nhân và tăng tốc độ trước khi thu dataset.

### Việc tiếp theo
1. Xác định vì sao CSI thô chỉ ~0,33 gói/s (router có trả lời ping không; loại khung nào tạo callback). Không bỏ gate, không tăng tốc độ ping.
2. Khi CSI đủ dày: thu phòng yên và phòng có người di chuyển, so biên độ theo thời gian.
3. Đặt cố định `IDF_TARGET`; xử lý IP (DHCP reservation hoặc IP tĩnh); theo dõi brownout.