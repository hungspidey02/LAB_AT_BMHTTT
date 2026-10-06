# BÁO CÁO BÀI THỰC HÀNH TƯỜNG LỬA PFSENSE

---

## 1. THÔNG TIN SINH VIÊN & BÀI THỰC HÀNH
* **Họ và tên:** Nguyễn Trường Hưng
* **Mã số sinh viên (MSSV):** 1150070013
* **Tên bài Lab:** Cấu hình và kiểm thử chính sách an ninh mạng trên tường lửa pfSense (Firewall Rules, NAT, Network Isolation)
* **Thời gian thực hiện:** Tháng 10/2026

---

## 2. PHIÊN BẢN VÀ MÔI TRƯỜNG THỰC NGHIỆM
* **Nền tảng ảo hóa:** VMware Workstation Pro 17.x
* **Máy ảo 1 — pfSense Firewall (FreeBSD/amd64):**
  * Phiên bản: pfSense 2.7.2-RELEASE (amd64)
  * Cấu hình Network Adapters:
    * `WAN (em0)`: Chế độ NAT, nhận IP động qua DHCP (`192.168.2.53/24`).
    * `LAN (em1)`: Chế độ Custom `VMnet1 (Host-only)`, IP tĩnh `10.0.0.1/8`.
    * `DMZ (em2)`: Chế độ Custom `VMnet2`, IP tĩnh `172.16.0.1/16`.
* **Máy ảo 2 — Domain Controller (LAN):**
  * Hệ điều hành: Windows Server 2025 Datacenter
  * Card mạng: `VMnet1 (Host-only)`
  * IP tĩnh: `10.0.0.2/8` | Gateway: `10.0.0.1` | DNS: `8.8.8.8`
* **Máy ảo 3 — DMZ-Web / LAN-Test:**
  * Hệ điều hành: Metasploitable2-Linux (Ubuntu derivative)
  * Card mạng:
    * Khi ở DMZ: `VMnet2`, IP tĩnh `172.16.0.2/16`, Gateway `172.16.0.1`.
    * Khi kiểm thử LAN: `VMnet1 (Host-only)`, IP tĩnh `10.0.0.3/8`, Gateway `10.0.0.1`.

---

## 3. HƯỚNG DẪN DỰNG VÀ KHỞI TẠO MÔI TRƯỜNG

### Bước 1: Cấu hình card mạng ảo trên VMware Virtual Network Editor
1. Thiết lập `VMnet1`: Host-only, Subnet IP `10.0.0.0/8`, tắt DHCP nội bộ.
2. Thiết lập `VMnet2`: Custom virtual network, Subnet IP `172.16.0.0/16`, tắt DHCP nội bộ.

### Bước 2: Khởi tạo và gán giao diện mạng trên pfSense
1. Khởi động pfSense trên VMware, chọn mục gán interface (`1) Assign Interfaces`):
   * Gán `em0` $\rightarrow$ `WAN`
   * Gán `em1` $\rightarrow$ `LAN`
   * Gán `em2` $\rightarrow$ `OPT1` (đổi tên thành `DMZ`)
2. Chọn mục `2) Set interface(s) IP address`:
   * `LAN`: Đặt IP `10.0.0.1`, subnet mask bit count `8`, không bật DHCP.
   * `DMZ`: Đặt IP `172.16.0.1`, subnet mask bit count `16`, không bật DHCP.

### Bước 3: Cấu hình máy trạm
1. **Windows Server 2025 (LAN):**
   * Đặt card mạng về `VMnet1`.
   * Gán IP tĩnh trong Network Connections: IP `10.0.0.2`, Subnet Mask `255.0.0.0`, Default Gateway `10.0.0.1`, DNS `8.8.8.8`.
2. **Metasploitable2 (DMZ ban đầu):**
   * Đặt card mạng về `VMnet2`.
   * Mở terminal gán IP và Gateway:
     ```bash
     sudo ifconfig eth0 172.16.0.2 netmask 255.255.0.0 up
     sudo route add default gw 172.16.0.1
     echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
     ```

---

## 4. CÁC TÌNH HUỐNG ĐÃ THỰC HIỆN VÀ KẾT QUẢ KIỂM THỬ

| Tình huống | Mục tiêu & Chính sách cấu hình | Phương thức kiểm thử | Kết quả mong đợi | Kết quả thực tế | Đánh giá |
| :--- | :--- | :--- | :--- | :--- | :---: |
| **Tình huống 1** | Cho phép máy chủ vùng DMZ (`172.16.0.2`) truy cập ra ngoài Internet thông qua Outbound NAT. | Đứng trên Metasploitable2 (DMZ) chạy lệnh:<br>`ping -c 4 8.8.8.8` | Nhận phản hồi thành công từ DNS Google, `0% packet loss`. | `4 packets transmitted, 4 received, 0% packet loss` | **PASS** |
| **Tình huống 2** | Chỉ cho phép duy nhất máy Domain Controller (`10.0.0.2`) được ra Internet; chặn toàn bộ các máy trạm khác trong mạng LAN (`10.0.0.3`). | **Phần 1:** Trên Windows Server (`10.0.0.2`) chạy CMD:<br>`ping 8.8.8.8`<br><br>**Phần 2:** Chuyển Metasploitable2 sang `VMnet1` (`10.0.0.3`) chạy:<br>`ping -c 4 8.8.8.8` | • Host `10.0.0.2`: Ping thành công (`0% loss`).<br><br>• Host `10.0.0.3`: Bị pfSense chặn lại (`100% packet loss`). | • `10.0.0.2`: Reply from 8.8.8.8, Lost = 0 (0% loss).<br><br>• `10.0.0.3`: 4 packets transmitted, 0 received, 100% packet loss. | **PASS** |
| **Tình huống 3** | **Cô lập DMZ khỏi mạng LAN:**<br>1. *Bước Baseline:* Tạm mở Windows Firewall và tắt rule Block để chứng minh mạng thông suốt.<br>2. *Bước Cô lập:* Bật rule `Block DMZ subnets to LAN subnets` nằm trên rule `Allow DMZ to Internet`. | Đứng trên Metasploitable2 (DMZ `172.16.0.2`) chạy:<br>1. Baseline ping DC `10.0.0.2`<br>2. Sau khi Block ping DC `10.0.0.2`<br>3. Ping ra ngoài Internet `8.8.8.8` | • Baseline: Ping thông suốt (`0% loss`).<br>• Sau chặn: Ping sang LAN bị chặn (`100% loss`).<br>• Ra Internet: Vẫn thông suốt (`0% loss`). | • Baseline: Nhận 4 reply (0% loss).<br>• Sau chặn: 100% packet loss khi ping `10.0.0.2`.<br>• Ra Internet: 0% packet loss khi ping `8.8.8.8`. | **PASS** |

---

## 5. CÁC LỖI GẶP PHẢI VÀ BIỆN PHÁP KHẮC PHỤC

### Lỗi 1: Sau khi thay đổi rule hoặc vô hiệu hóa rule, kết quả kiểm thử không thay đổi ngay
* **Hiện tượng:** Rule chặn Internet hoặc chặn DMZ đã tạo nhưng lệnh ping vẫn thông.
* **Nguyên nhân:** pfSense là một Stateful Firewall. Các phiên kết nối cũ vẫn còn được lưu vết trong bảng trạng thái (**State Table**). Gói tin tiếp theo của phiên cũ sẽ tự động được đi qua mà không cần so khớp lại bảng quy tắc mới.
* **Cách khắc phục:** 
  1. Sau khi chỉnh sửa rule và nhấn `Apply Changes`, truy cập vào menu **`Diagnostics`** $\rightarrow$ **`States`** $\rightarrow$ chọn tab **`Reset States`**.
  2. Tích chọn ô `Reset the firewall state table` và bấm nút **`Reset`**.
  3. Tải lại trang (F5) nếu phiên WebGUI bị treo tạm thời.

---

### Lỗi 2: Trình duyệt bị treo xoay vòng sau khi bấm "Reset States"
* **Hiện tượng:** Trình duyệt Edge trên Windows Server không phản hồi, biểu tượng tải trang quay liên tục sau khi bấm nút Reset.
* **Nguyên nhân:** Hành động Reset States xóa sạch toàn bộ các phiên kết nối đang mở của tường lửa, bao gồm cả phiên HTTPS giữa trình duyệt và WebGUI pfSense (`10.0.0.1`).
* **Cách khắc phục:** Đây là cơ chế hoạt động bình thường của pfSense. Chỉ cần nhấn phím **F5** trên trình duyệt để nạp lại trang và đăng nhập lại nếu có yêu cầu.

---

### Lỗi 3: Metasploitable2 báo lỗi `connect: Network is unreachable` khi ping
* **Hiện tượng:** Terminal Linux báo `connect: Network is unreachable` ngay khi vừa gõ lệnh `ping 8.8.8.8`.
* **Nguyên nhân:** Khi thực hiện lại lệnh `ifconfig eth0 <IP> netmask ... up`, hệ điều hành Linux tự động xóa bỏ tuyến đường mặc định (Default Gateway) đã gán trước đó trong bảng định tuyến.
* **Cách khắc phục:** Bổ sung lại Default Gateway bằng lệnh:
  ```bash
  sudo route add default gw 10.0.0.1
