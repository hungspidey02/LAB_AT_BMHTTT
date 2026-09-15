# LAB 1 - Bắt gói tin Telnet - SSH

- Họ và tên: Nguyễn Trường Hưng
- MSSV: 1150070013
- Lớp: 11_TMĐT

## Nội dung bài Lab
Lab 1.2: Bắt và phân tích gói tin Telnet – SSH bằng Wireshark (Examining SSH & Telnet in Wireshark)
## Nội dung đã thực hiện
### ▪ Nội dung đã thực hiện

* **Thiết lập môi trường mạng Client/Server trên VMware Workstation:**
  * **Máy chủ (Server):** Ubuntu Server (`192.168.110.130`) cài đặt và cấu hình đồng thời 2 dịch vụ: Telnet Server (cổng 23) và OpenSSH Server (cổng 22).
  * **Máy khách & Bắt gói (Client & Attacker):** Windows 10 cài đặt PuTTY để kết nối điều khiển và Wireshark (kèm Npcap) để bắt, phân tích lưu lượng.

* **Quản trị người dùng thử nghiệm:**
  * Tạo tài khoản mới `nguyentruonghung` và gán quyền `sudo` trên máy chủ Ubuntu.
  * Kiểm tra tính thông suốt giữa hai máy qua lệnh `ping` (kết quả 0% packet loss).

* **Thực nghiệm bắt và phân tích gói tin Telnet (Port 23):**
  * Thiết lập bộ lọc Wireshark `tcp.port == 23`.
  * Kết nối PuTTY qua cổng 23, đăng nhập bằng tài khoản sinh viên và thực thi các câu lệnh kiểm tra (`whoami`, `pwd`, `ls`).
  * Sử dụng tính năng `Follow TCP Stream` trên Wireshark để trích xuất và phân tích luồng dữ liệu.
  * Đổi mật khẩu tài khoản sang chuỗi phức tạp dài hơn 10 ký tự (`sudo passwd nguyentruonghung`) và thực hiện bắt gói lại để kiểm chứng mức độ an toàn.

* **Thực nghiệm bắt và phân tích gói tin SSH (Port 22):**
  * Thiết lập bộ lọc Wireshark `tcp.port == 22`.
  * Kết nối PuTTY qua SSH, xác thực máy chủ qua Host Key Fingerprint và đăng nhập thực thi lệnh.
  * Trích xuất và phân tích luồng dữ liệu mã hóa qua tính năng `Follow TCP Stream`.

* **Tổng kết và trả lời câu hỏi thu hoạch:**
  * Phân tích bản chất kỹ thuật theo tam giác bảo mật CIA (Confidentiality, Integrity, Authentication).
  * Trình bày nguyên lý xác thực bằng Public-key Authentication và đề xuất các giải pháp Hardening cho SSH trong môi trường thực tế.

---

### ▪ Kết quả thực hiện

* **Đối với giao thức Telnet:**
  * Toàn bộ phiên làm việc truyền tải dưới dạng bản rõ (*Plaintext*).
  * Wireshark khôi phục được 100% thông tin nhạy cảm: tên đăng nhập (`nguyentruonghung`), mật khẩu tài khoản và toàn bộ câu lệnh quản trị mà không cần qua bước giải mã.
  * Việc sử dụng mật khẩu dài, phức tạp hoàn toàn không có tác dụng bảo vệ dữ liệu do thiếu vắng cơ chế mã hóa trên kênh truyền.

* **Đối với giao thức SSH:**
  * Ngoại trừ các gói tin thỏa thuận phiên bản (`SSH-2.0-PuTTY_Release_0.85`, `OpenSSH_10.2p1`) và danh sách thuật toán đàm phán ban đầu, toàn bộ quá trình xác thực và câu lệnh thao tác phía sau đều được mã hóa đối xứng thành dạng bản mã (*Ciphertext*).
  * Không thể đọc trộm hay trích xuất nội dung thông qua bắt gói tin, bảo đảm tuyệt đối tính bí mật và toàn vẹn của dữ liệu trước kỹ thuật nghe lén (*Sniffing*) trong mạng nội bộ.
