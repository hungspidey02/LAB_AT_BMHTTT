# BÁO CÁO THỰC HÀNH: KHẢO SÁT VÀ ĐÁNH GIÁ BỀ MẶT MẠNG BẰNG NMAP

## 1. THÔNG TIN SINH VIÊN & BÀI THỰC HÀNH
* **Họ và tên:** Nguyễn Trường Hùng
* **Mã số sinh viên (MSSV):** 1150070013
* **Học phần:** An toàn hệ thống thông tin
* **Tên bài thực hành:** Khảo sát và đánh giá bề mặt mạng bằng Nmap (LAB 4 - Network Surface Reconnaissance & Vulnerability Assessment with Nmap)

---

## 2. PHIÊN BẢN MÔI TRƯỜNG THỰC HÀNH

| Thành phần / Thiết bị | Hệ điều hành & Bản phân phối | Địa chỉ IP / Subnet | Địa chỉ MAC | Phiên bản Nmap / Dịch vụ then chốt | Vai trò |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Kali Linux** | Kali Linux 64-bit (Kernel 6.x) | `192.168.136.129/24` (`eth0`) | Local Interface | **Nmap 7.99**, `xsltproc` | Máy quét tấn công / Kiểm thử bảo mật (Attacker) |
| **Metasploitable 2** | Ubuntu 8.04 LTS (Kernel 2.6.x) | `192.168.136.130/24` (`eth0`) | `00:0C:29:34:B6:86` | vsftpd 2.3.4, Apache 2.2.8, Samba 3.0.20, OpenSSH 4.7p1 | Máy chủ mục tiêu chứa lỗ hổng (Target) |
| **Windows 10 VM** | Windows 10 Pro 64-bit (Build 19045.3803) | `192.168.136.128/24` (`Ethernet0`) | `00:0C:29:AD:80:DE` | **Nmap 7.991** (Npcap 1.88), Windows Defender Firewall | Máy mục tiêu đối chứng chính sách bảo vệ (Control Target) |
| **Hạ tầng mạng ảo** | VMware Workstation Pro | Dải Host-Only: `192.168.136.0/24` | Gateway: `192.168.136.1`<br>DHCP: `192.168.136.254` | VMware Virtual Switch (Host-Only) | Môi trường mạng thử nghiệm cách ly an toàn |

---

## 3. CÁCH THỨC XÂY DỰNG MÔI TRƯỜNG (SETUP LAB)

1. **Cấu hình Card mạng cách ly (Host-Only):**
   * Trong VMware Workstation, mở **Virtual Network Editor** $\rightarrow$ chọn card mạng chế độ **Host-Only** (dải subnet: `192.168.136.0/24`, Netmask: `255.255.255.0`).
   * Gán cả 3 máy ảo (**Kali Linux**, **Metasploitable 2**, **Windows 10**) kết nối vào cùng card mạng Host-Only này.
2. **Khởi tạo và kiểm tra cấu hình mạng trên máy đích:**
   * **Metasploitable 2:** Khởi động máy, đăng nhập với tài khoản `msfadmin`/`msfadmin`. Chạy lệnh `ifconfig` để xác nhận IP `192.168.136.130`.
   * **Windows 10 VM:** Mở Command Prompt (cmd), chạy `ipconfig` kiểm tra IP `192.168.136.128`. Cài đặt bộ công cụ Nmap for Windows (`nmap-7.991-setup.exe`).
3. **Kiểm tra thông mạng trên máy quét Kali Linux:**
   * Mở Terminal chạy `ip -br addr` xác nhận IP `192.168.136.129`.
   * Kiểm tra định tuyến nội bộ: `ping -c 3 192.168.136.130` (đảm bảo phản hồi tốt, không mất gói).

---

## 4. CÁC TÌNH HUỐNG VÀ NỘI DUNG ĐÃ THỰC HIỆN

1. **Host Discovery (Rà quét phát hiện máy sống):**
   * Lệnh: `sudo nmap -sn 192.168.136.0/24`
   * Xác định được 5 thực thể mạng đang hoạt động (`.1`, `.128`, `.129`, `.130`, `.254`).
2. **Khảo sát và đối chiếu kỹ thuật quét cổng TCP:**
   * Quét TCP Connect: `nmap -sT 192.168.136.130` (Bắt tay 3 bước hoàn chỉnh, không cần root).
   * Quét SYN Stealth: `sudo nmap -sS 192.168.136.130` (Quét nửa mở Half-open, cần quyền root, tối ưu che giấu log).
   * Đồng nhất tìm thấy 23 cổng TCP đang mở trên máy đích.
3. **Thực nghiệm kỹ thuật quét gói tin dị thường (RFC 793):**
   * Chạy các kỹ thuật FIN Scan (`-sF`), Null Scan (`-sN`), Xmas Scan (`-sX`).
   * Nhân Linux trên Metasploitable 2 tuân thủ chuẩn RFC 793, phân tách rõ ràng cổng mở (`open|filtered`) và cổng đóng (`closed`).
   * Quét ACK Scan (`-sA`) phân tích chính sách lọc: kết quả trả về `1000 unfiltered tcp ports (reset)`, khẳng định máy đích không bật Stateful Firewall.
4. **Quét cổng UDP & Đánh giá rủi ro:**
   * Lệnh: `sudo nmap -sU -p 53,69,111,137,161 192.168.136.130`
   * Phân tích các điểm yếu dịch vụ DNS (53), TFTP (69), RPC (111), NetBIOS (137) và SNMP (161).
5. **Dò quét phiên bản dịch vụ & Dấu vân tay Hệ điều hành:**
   * Lệnh: `sudo nmap -sV -O --osscan-guess 192.168.136.130` và `sudo nmap -A 192.168.136.130`
   * Bóc tách chính xác phiên bản các dịch vụ: `vsftpd 2.3.4`, `Apache httpd 2.2.8`, `Samba 3.0.20`, `OpenSSH 4.7p1` và OS Linux 2.6.x.
6. **Rà soát lỗ hổng bảo mật chuyên sâu bằng NSE Scripts:**
   * Lệnh: `sudo nmap --script "vuln" -p 21,80,445 192.168.136.130`
   * Phát hiện lỗ hổng Backdoor cực kỳ nguy hiểm **CVE-2011-2523** trên vsftpd 2.3.4 (trả về root shell `uid=0(root)`), nguy cơ tấn công từ chối dịch vụ Slowloris (CVE-2007-6750), HTTP TRACE và lộ thư mục nhạy cảm (`/phpinfo.php`, `/phpMyAdmin/`).
7. **Kiểm tra và so sánh lỗ hổng SMB MS17-010 (EternalBlue):**
   * Quét đối chiếu giữa Linux (`.130`) và Windows VM (`.128`) bằng kịch bản `smb-vuln-ms17-010`.
   * Windows 10 VM kích hoạt Firewall chặn cổng 445 (`filtered`), bảo vệ hệ thống trước sự thăm dò trực tiếp từ bên ngoài.
8. **Xuất hồ sơ bằng chứng kiểm thử & Sinh báo cáo HTML:**
   * Xuất các định dạng: Normal Text (`-oN ket_qua.txt`), XML (`-oX ket_qua.xml`), Grepable (`-oG smb.txt`).
   * Sử dụng công cụ `xsltproc ket_qua.xml -o bao_cao.html` để tạo báo cáo giao diện Web trực quan.
9. **Thực nghiệm trước và sau khi Hardening (Gia cố hệ thống):**
   * **Before:** Quét cổng 21, 23 trên `.130` ghi nhận cả hai đều `open`.
   * **Hardening Action:** Thiết lập quy tắc tường lửa `iptables` drop lưu lượng cổng 21 trên Metasploitable 2:
     ```bash
     sudo iptables -A INPUT -p tcp --dport 21 -j DROP
     ```
   * **After:** Quét lại ghi nhận cổng 21 chuyển ngay sang trạng thái **`filtered`**, chứng minh biện pháp phòng thủ tầng mạng đã phát huy tác dụng cô lập dịch vụ.

---

## 5. BẢNG TỔNG HỢP ĐÁNH GIÁ KẾT QUẢ (PASS / FAIL)

| STT | Nhiệm vụ / Tình huống kiểm thử | Tiêu chí đánh giá hoàn thành | Kết quả |
| :---: | :--- | :--- | :---: |
| 1 | Chuẩn bị môi trường & Host Discovery | Phát hiện đầy đủ các IP và MAC trong subnet Host-Only | **PASS** |
| 2 | Khảo sát cổng TCP (`-sT` và `-sS`) | Liệt kê đủ 23 cổng open, so sánh được cơ chế bắt tay TCP | **PASS** |
| 3 | Quét nâng cao (FIN, NULL, Xmas, ACK) | Giải thích được cơ chế RFC 793 và trạng thái bộ lọc firewall | **PASS** |
| 4 | Rà soát cổng UDP & Phân tích rủi ro | Định danh chính xác trạng thái UDP và giải thích ICMP rate-limit | **PASS** |
| 5 | Dò phiên bản dịch vụ & OS Fingerprint | Xác định đúng danh mục service version và nhân Linux kernel | **PASS** |
| 6 | Đánh giá an toàn bằng NSE (`vuln`) | Phát hiện backdoor CVE-2011-2523 và các lỗi Web/HTTP | **PASS** |
| 7 | Kiểm tra đối chiếu SMB MS17-010 | Đánh giá được sự khác biệt giữa cổng open (Samba) và filtered (Win) | **PASS** |
| 8 | Tạo hồ sơ kiểm thử & Báo cáo HTML | Xuất đầy đủ `.txt`, `.xml`, `.html` và hiển thị trên Firefox | **PASS** |
| 9 | Tình huống Before / After Hardening | Minh chứng rõ ràng sự thay đổi trạng thái cổng từ `open` sang `filtered` | **PASS** |
| 10 | Báo cáo Word & Ảnh minh chứng | Hoàn thành 8/8 danh mục ảnh minh chứng và 10 câu hỏi lý thuyết | **PASS** |

---

## 6. CÁC LỖI GẶP PHẢI VÀ BIỆN PHÁP KHẮC PHỤC

### Lỗi 1: Gõ nhầm cú pháp tham số né ping (`Failed to resolve "Pn"`)
* **Hiện tượng:** Chạy lệnh quét Windows báo lỗi `Failed to resolve "Pn"`.
* **Nguyên nhân:** Thiếu dấu gạch ngang `-` trước chữ `Pn` (khiến Nmap tưởng là một tên miền cần phân giải DNS).
* **Khắc phục:** Sửa lại đúng cú pháp tham số chuẩn: `-Pn`.

### Lỗi 2: Trình biên dịch `xsltproc` không phân tích được tệp XML (`parser error : Start tag expected, '<' not found`)
* **Hiện tượng:** Lệnh `xsltproc ket_qua.xml -o bao_cao.html` văng lỗi không parse được dòng đầu tiên.
* **Nguyên nhân:** Đã dùng nhầm cờ văn bản thường `-oN` ghi vào file có tên đuôi `.xml`.
* **Khắc phục:** Xuất lại file XML chuẩn bằng cờ `-oX`: `sudo nmap -sV -O 192.168.136.130 -oX ket_qua.xml`.

### Lỗi 3: Không thể tắt dịch vụ trên Metasploitable 2 (`sudo: service: command not found`)
* **Hiện tượng:** Gõ `sudo service vsftpd stop` hoặc đường dẫn `/etc/init.d/vsftpd` đều báo không tìm thấy lệnh.
* **Nguyên nhân:** Ubuntu 8.04 LTS cũ không hỗ trợ `service` trong môi trường sudo và các dịch vụ mạng được siêu tiến trình `inetd` điều khiển.
* **Khắc phục:** Chuyển sang mô phỏng Hardening bằng Tường lửa `iptables` drop cổng 21 (`sudo iptables -A INPUT -p tcp --dport 21 -j DROP`). Cổng chuyển ngay sang `filtered`.

### Lỗi 4: Máy quét Kali báo `0 hosts up` khi quét Windows 10 VM
* **Hiện tượng:** Quét `smb-vuln-ms17-010` tới IP `.128` báo `0 hosts up` sau hơn 1 giây.
* **Nguyên nhân:** Máy ảo Windows 10 tự động Sleep/khóa màn hình khi để lâu, không phản hồi gói tin ARP Ping.
* **Khắc phục:** Bật lại màn hình Windows 10 và có thể thêm cờ `--disable-arp-ping`.
