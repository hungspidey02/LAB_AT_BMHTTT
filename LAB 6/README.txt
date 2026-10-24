# LAB 6: XÂY DỰNG HỆ THỐNG TƯỜNG LỬA VÀ PHÂN VÙNG MẠNG VỚI PFSENSE

- **Họ và tên:** Nguyễn Trường Hưng
- **Mã số sinh viên:** 1150070013
- **Môn học:** An toàn mạng / Quản trị hệ thống mạng

---

## 1. Mục tiêu bài Lab
- Triển khai và cấu hình tường lửa pfSense với 3 phân vùng mạng: WAN, LAN, DMZ.
- Thiết lập định tuyến và quy tắc tường lửa (Firewall Rules) bảo vệ từng vùng mạng.
- Kiểm soát luồng lưu lượng truy cập giữa Client nội bộ (LAN) và máy chủ dịch vụ (DMZ) theo nguyên tắc Least Privilege.

---

## 2. Thông số kỹ thuật & Địa chỉ IP

| Phân vùng | Tên Interface (pfSense) | Dải mạng / IP Gateway | Thiết bị kết nối |
| :--- | :---: | :--- | :--- |
| **WAN** | `em0` | DHCP / Cấp từ Router ngoài | Kết nối Internet |
| **MGMT** | `em1` | `192.168.136.254/24` | Máy thật quản trị pfSense |
| **LAN** | `em2` (`DMZ`/OPT1) | `10.6.0.1/24` | Windows Server DC01, Windows 10, Web-Proxy |
| **DMZ** | `em3` (`OPT2`) | `172.16.6.1/24` | DMZ-WEB (Kali Linux: `172.16.6.10`) |

---

## 3. Các bước thực hiện chính
1. Khởi tạo máy ảo pfSense trên VMware Workstation, gán 4 card mạng tương ứng.
2. Cấu hình IP tĩnh cho các interface thông qua Console Menu.
3. Truy cập WebGUI qua `https://192.168.136.254` để cấu hình Firewall Rules.
4. Cấu hình rule cho phép LAN truy cập dịch vụ Web (HTTP/HTTPS) trên DMZ, cấm các truy cập không cần thiết khác.
5. Kiểm tra thông mạch và kiểm thử cô lập vùng mạng.
