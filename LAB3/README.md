# BÁO CÁO THỰC HÀNH LAB 3: NHẬN DIỆN VÀ ỨNG PHÓ CÁC MỐI ĐE DỌA ĐẾN AN TOÀN THÔNG TIN

## 1. THÔNG TIN CHUNG
- **Họ và tên:** Nguyễn Trường Hưng
- **Mã số sinh viên:** 1150070013
- **Lớp:** 11TMĐT
- **Bộ môn:** An toàn Thông tin - Khoa CNTT
- **Năm học:** 2026–2027
- **Tên bài thực hành:** Lab 3 - Nhận diện và ứng phó các mối đe dọa đến an toàn thông tin
- **Link video báo cáo thực hành:** [Dán link video demo tại đây]

---

## 2. PHIÊN BẢN MÔI TRƯỜNG THỰC HÀNH
- **Hệ điều hành:** Windows 10 Pro 64-bit Version 22H2 (Build 19045.3803)
- **Phần mềm ảo hóa:** VMware Workstation (Chế độ mạng: Host-only)
- **Mã Snapshot ban đầu:** `LAB3_CLEAN_WIN10`
- **Công cụ & Phiên bản:**
  - Python: 3.12.3
  - Wireshark / TShark: 4.6.8
  - Microsoft Sysinternals: Autorunsc v14.3, Process Explorer v17.14, Sysmon v15.22
  - Microsoft Defender Antivirus: Real-Time Protection & Antivirus Enabled (Engine 1.1.26080.3)
  - Windows Firewall: All Profiles Enabled (Domain, Private, Public)

---

## 3. CÁCH DỰNG MÔI TRƯỜNG (SETUP ENVIRONMENT)
1. **Cấu hình máy ảo:** Thiết lập Network Adapter của máy ảo Windows 10 về chế độ `Host-only` nhằm cô lập lưu lượng, đảm bảo an toàn không ảnh hưởng ra mạng bên ngoài.
2. **Khởi tạo Snapshot:** Chụp snapshot sạch ban đầu mang tên `LAB3_CLEAN_WIN10`.
3. **Cấu trúc thư mục làm việc:** 
   - Khởi tạo thư mục gốc `C:\LAB3` gồm các thư mục con: `Evidence`, `Tools`, `Downloads`, `Assets`.
   - Ghi nhận mốc thời gian bắt đầu vào tệp `C:\LAB3\Evidence\start_time.txt`.
4. **Giải nén tài nguyên:** Đưa tệp tài nguyên `LAB3_Threats_Assets.zip` vào `Downloads`, kiểm tra SHA-256 khớp mẫu và giải nén vào `C:\LAB3`.
5. **Cài đặt công cụ:** Cài đặt Python 3.12.3, Wireshark 4.6.8 và giải nén bộ công cụ Sysinternals vào `C:\LAB3\Tools\`.

---

## 4. CÁC TÌNH HUỐNG THỰC HIỆN VÀ KẾT QUẢ

| Tình huống | Mô tả nội dung thực hiện | Kết quả | Minh chứng chính |
| :--- | :--- | :---: | :--- |
| **TH0** | Thiết lập Baseline hệ thống, phiên bản công cụ, trạng thái Defender và Firewall | **PASS** | `H2_ToolVersions.png`, `H3_Baseline_Defender_Firewall.png`, các file txt baseline |
| **TH1** | Phân loại nguồn đe dọa, xác định tài sản, lỗ hổng và lập Risk Register theo C-I-A | **PASS** | Bảng ma trận rủi ro trong báo cáo |
| **TH2** | Mô phỏng mã độc kiểm thử EICAR, kiểm chứng khả năng ngăn chặn của Microsoft Defender | **PASS** | `H4_EICAR_Detection.png`, `eicar_detection.txt` |
| **TH3** | Bật Logon Audit, mô phỏng Brute-Force sinh Event 4625, xoay vòng mật khẩu an toàn | **PASS** | `H5_Event4625.png`, `auth_events_before_rotation.txt` |
| **TH4** | Cài cắm Persistence qua Registry Run key, mở socket 8443, phát hiện bằng Autoruns | **PASS** | `H6_Backdoor_Autoruns_Process.png`, `backdoor_listener_8443.txt` |
| **TH5** | Bắt gói tin trên loopback bằng Wireshark, phát hiện lộ credentials qua HTTP plaintext | **PASS** | `H7_Wireshark_HTTP_Plaintext.png`, `http_login.pcapng` |
| **TH6** | Kiểm thử tải DoS nội bộ trên loopback đo độ trễ; phân tích ngoại tuyến mailbombing | **PASS** | `H8_DoS_Load.png`, `dos_metrics.txt`, `mail_bombing_analysis.txt` |
| **TH7** | Phân tích email lừa đảo offline (phishing_email.txt), nhận diện 4 chỉ dấu bất thường | **PASS** | `H9_Phishing_Analysis.png`, `phishing_analysis.txt` |
| **TH8** | Cleanup hệ thống, thu thập autoruns_after, băm SHA-256 bảo toàn chứng cứ và revert VM | **PASS** | `evidence_sha256.csv`, `end_time.txt`, `autoruns_after.csv` |

---

## 5. LỖI GẶP PHẢI VÀ CÁCH KHẮC PHỤC (TROUBLESHOOTING)
1. **Lỗi `Set-LocalUser : User lab3user was not found` (Tình huống 3):**
   - *Nguyên nhân:* Do script cleanup ở lần chạy trước đã xóa tài khoản `lab3user`, khiến lệnh đổi mật khẩu không tìm thấy đối tượng.
   - *Khắc phục:* Bổ sung lệnh khởi tạo lại người dùng bằng `New-LocalUser -Name 'lab3user'` trước khi áp dụng mật khẩu mới `Lab3_Rotate_2026!#Safe`.
2. **Lỗi `No MSFT_NetTCPConnection objects found` trên cổng 8443 và 8080 (Tình huống 4 & 5):**
   - *Nguyên nhân:* Lệnh `Start-Process python` chạy bất đồng bộ trong nền, câu lệnh kiểm tra cổng mạng chạy ngay lập tức khi socket Python chưa kịp hoàn tất hàm `bind()` và `listen()`.
   - *Khắc phục:* Thêm lệnh trễ `Start-Sleep -Seconds 1` ngay sau khi kích hoạt tiến trình Python để đảm bảo cổng mạng đã mở hoàn toàn trước khi kiểm tra hoặc gửi request.
3. **Wireshark không bắt được gói `POST /login` (Tình huống 5):**
   - *Nguyên nhân:* Chọn nhầm card mạng vật lý hoặc bộ lọc cổng chưa tối ưu.
   - *Khắc phục:* Bắt chính xác trên card `Adapter for loopback traffic capture` (Npcap) và sử dụng bộ lọc `tcp.port == 8080` thay vì chỉ lọc `http`.

---

## 6. DANH MỤC TỆP TRONG THƯ MỤC LAB3/
- `README.md`: Báo cáo tóm tắt thực hành và môi trường.
- `[MãLớp]-LAB3_1150070013-NguyenTruongHung.docx`: Báo cáo chi tiết đầy đủ câu hỏi và hình ảnh minh chứng.
- `evidence_sha256.csv`: Bảng mã băm SHA-256 xác thực tính toàn vẹn của tất cả các file log và ảnh minh chứng.
- `Logs & Outputs/`: Thư mục chứa các file log đã làm sạch (`start_time.txt`, `end_time.txt`, `baseline_*.txt`, `dos_metrics.txt`, `http_login.pcapng`,...).
