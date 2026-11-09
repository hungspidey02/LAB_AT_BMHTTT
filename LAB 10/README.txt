# BÁO CÁO BÀI THỰC HÀNH TƯỜNG LỬA PFSENSE - LAB 10

## 1. THÔNG TIN SINH VIÊN & BÀI THỰC HÀNH
* **Họ và tên:** Nguyễn Trường Hưng
* **Mã số sinh viên (MSSV):** 1150070013
* **Tên bài Lab:** Remote DNS Cache Poisoning trong môi trường pfSense cô lập
* **Thời gian thực hiện:** Tháng 11/2026

---

## 2. PHIÊN BẢN VÀ MÔI TRƯỜNG THỰC NGHIỆM
* **Nền tảng ảo hóa:** VMware Workstation Pro 17.x / Oracle VM VirtualBox[cite: 4]
* **Máy ảo 1 — pfSense Firewall (LAB 10 Gateway):**[cite: 4]
  * Phiên bản: pfSense 2.7.2-RELEASE (amd64)
  * Cấu hình Network Adapters:[cite: 4]
    * `WAN (em0)`: Chế độ NAT / DHCP (`192.168.250.0/24`)[cite: 4]
    * `DNSLAB (em1)`: Custom Internal Network / LAN Segment `DNS-LAB`, IP tĩnh `10.9.0.1/24`[cite: 2, 4]
    * `DMZ (em2)`: Custom Internal Network / LAN Segment `DMZ`, IP tĩnh `172.16.6.1/24`[cite: 2, 4]
    * `ATTACKNET (em3 / Adapter 6)`: Custom Internal Network / LAN Segment `ATTACK-NET`, IP tĩnh `10.10.0.1/24` (Upstream Gateway = None)[cite: 2, 4]
* **Máy ảo 2 — DNS-SERVER (DNSLAB):**[cite: 4]
  * Hệ điều hành: Ubuntu Server / Debian[cite: 4]
  * Card mạng: Phân vùng `DNS-LAB`[cite: 2, 4]
  * IP tĩnh: `10.9.0.10/24` | Gateway: `10.9.0.1`[cite: 4]
  * Dịch vụ: BIND 9.18 (baseline & hardening) / `vuln_resolver_lab10.py` (attack phase cố định UDP source port 33333)[cite: 4]
* **Máy ảo 3 — VICTIM (DNSLAB):**[cite: 4]
  * Hệ điều hành: Ubuntu / Windows Desktop[cite: 4]
  * Card mạng: Phân vùng `DNS-LAB`[cite: 2, 4]
  * IP tĩnh: `10.9.0.100/24` | Gateway: `10.9.0.1` | DNS Server: `10.9.0.10`[cite: 4]
* **Máy ảo 4 — ATTACKER (Kali Linux):**[cite: 4]
  * Hệ điều hành: Kali Linux[cite: 4]
  * Card mạng: Phân vùng `ATTACK-NET` (Promiscuous Mode = Deny, tháo rời card `DNS-LAB`)[cite: 2, 4]
  * IP tĩnh: `10.10.0.200/24` | Gateway: `10.10.0.1`[cite: 4]
  * Kịch bản: `evil_dns_lab10.py` (UDP 53), HTTP Server (TCP 80), `kaminsky_remote_lab10.py` (UDP 5300 relay)[cite: 4]
* **Máy ảo 5 — AUTH-DNS / APP-TEST & DMZ-WEB (DMZ):**[cite: 4]
  * Card mạng: Phân vùng `DMZ`[cite: 2, 4]
  * IP máy chủ hợp lệ: `172.16.6.20/24` chạy `auth_dns_relay_lab10.py` (Auth DNS + race harness UDP 5300)[cite: 4]
  * Website hợp lệ (DMZ-WEB): `172.16.6.10/24`[cite: 4]

---

## 3. HƯỚNG DẪN DỰNG VÀ KHỞI TẠO MÔI TRƯỜNG

### Bước 1: Cấu hình card mạng ảo và gán Interface pfSense
1. Trên phần mềm ảo hóa, thêm Network Adapter mới cho pfSense nối vào mạng nội bộ `attack-net`[cite: 4]. Thiết lập Promiscuous Mode = Deny[cite: 2, 4].
2. Khởi động WebGUI pfSense, vào **Interfaces** > **Assignments** > bấm **Add** interface mới[cite: 4]:
   * Description: `ATTACKNET`[cite: 4]
   * IPv4 Configuration Type: `Static IPv4`[cite: 4]
   * IPv4 Address: `10.10.0.1/24`[cite: 4]
   * **IPv4 Upstream Gateway**: Chọn `None` (Không gán gateway cho interface nội bộ để tránh xung đột routing/NAT của pfSense)[cite: 4].
   * Nhấn **Save** và **Apply Changes**[cite: 4].

### Bước 2: Thiết lập Firewall Rules cô lập trên pfSense
1. **Quy tắc trên Interface DNSLAB:**[cite: 4]
   * `Pass` | Protocol: `TCP/UDP` | Source: `10.9.0.100` | Port: `*` | Dest: `10.9.0.10` | Port: `53` (Victim -> Resolver)[cite: 4]
   * `Pass` | Protocol: `UDP` | Source: `10.9.0.10` | Port: `*` | Dest: `172.16.6.20` | Port: `53` (Resolver -> Auth-DNS)[cite: 4]
   * `Pass` | Protocol: `UDP` | Source: `10.9.0.10` | Port: `*` | Dest: `10.10.0.200` | Port: `53` (Resolver -> Malicious NS sau poison)[cite: 4]
   * `Pass` | Protocol: `TCP` | Source: `10.9.0.0/24` | Port: `*` | Dest: `10.10.0.200` | Port: `80` (Victim truy cập Web giả mạo)[cite: 4]
   * `Block + Log` | Source: `10.9.0.0/24` | Dest: `any` (Chặn các phân vùng còn lại và bảo vệ Mgmt)[cite: 4]
2. **Quy tắc trên Interface ATTACKNET:**[cite: 4]
   * `Pass` | Protocol: `TCP/UDP` | Source: `10.10.0.200` | Port: `*` | Dest: `10.9.0.10` | Port: `53` (Kali trigger Resolver)[cite: 4]
   * `Pass` | Protocol: `UDP` | Source: `10.10.0.200` | Port: `*` | Dest: `172.16.6.20` | Port: `5300` (TXID Race Relay)[cite: 4]
   * `Block + Log` | Source: `10.10.0.0/24` | Dest: `any` (Cô lập hoàn toàn Kali khỏi WAN, LAN và MGMT)[cite: 4]
3. Vào **Diagnostics** > **States** > tab **Reset States**, bấm **Reset** để làm sạch bảng phiên[cite: 4].

### Bước 3: Cấu hình các máy trạm
1. **Kali Linux:**[cite: 4]
   * Tháo bỏ adapter `dns-lab`, chỉ giữ card gắn vào `attack-net`[cite: 4].
   * Gán IP tĩnh và gateway:[cite: 4]
     ```bash
     sudo ip addr flush dev eth0
     sudo ip addr add 10.10.0.200/24 dev eth0
     sudo ip link set eth0 up
     sudo ip route add default via 10.10.0.1
     ```
2. **AUTH-DNS / APP-TEST (172.16.6.20):**[cite: 4]
   * Chạy máy chủ DNS có chủ quyền kèm race relay (trả về IP 172.16.6.10 cho domain hợp lệ):[cite: 4]
     ```bash
     sudo python3 auth_dns_relay_lab10.py
     ```
3. **DNS-SERVER (10.9.0.10):**[cite: 4]
   * Sao lưu và nạp cấu hình baseline BIND9 forward zone `example.lab10.test` về `172.16.6.20`:[cite: 4]
     ```bash
     sudo named-checkconf
     sudo systemctl restart bind9
     ```

---

## 4. CÁC TÌNH HUỐNG ĐÃ THỰC HIỆN VÀ KẾT QUẢ KIỂM THỬ

| Tình huống | Mục tiêu & Chính sách cấu hình | Phương thức kiểm thử | Kết quả mong đợi | Kết quả thực tế | Đánh giá |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Tình huống 1: Baseline trước tấn công** | Đo lường trạng thái phân giải hợp lệ ban đầu qua BIND9 khi hệ thống an toàn.[cite: 4] | Đứng trên VICTIM (10.9.0.100) chạy:<br>`sudo resolvectl flush-caches`<br>`dig www.example.lab10.test`<br>`curl http://www.example.lab10.test`[cite: 2, 4] | Phân giải chính xác về IP DMZ-WEB `172.16.6.10`. Trình duyệt web hiển thị trang web hợp lệ.[cite: 4] | Nhận A record `172.16.6.10`, lệnh `curl` phản hồi HTTP 200 từ web server hợp lệ. | **PASS**[cite: 2] |
| **Tình huống 2: Xác nhận tính Remote (Cô lập mạng)** | Chứng minh Kali nằm trên subnet riêng, bị cô lập bởi pfSense và không thể sniff lưu lượng nội bộ.[cite: 4] | Đứng trên Kali (10.10.0.200) chạy:<br>`sudo tcpdump -ni any udp port 53`<br>Đồng thời từ VICTIM gửi truy vấn `dig www.example.lab10.test`.[cite: 4] | Tcpdump trên Kali hoàn toàn im lặng, không bắt được bất kỳ gói tin nào trao đổi giữa 10.9.0.10 và 172.16.6.20.[cite: 4] | 0 packets captured liên quan đến lưu lượng phân giải nội bộ của DNS-SERVER. | **PASS**[cite: 2] |
| **Tình huống 3: Thực hiện tấn công Kaminsky** | Đầu độc bộ đệm Resolver mô phỏng (cố định source port 33333) bằng kỹ thuật Kaminsky đoán TXID.[cite: 4] | 1. DNS-SERVER: `sudo systemctl stop bind9` rồi chạy `sudo python3 vuln_resolver_lab10.py`<br>2. Kali: Terminal 1 chạy `sudo python3 evil_dns_lab10.py`; Terminal 2 chạy HTTP port 80; Terminal 3 chạy `python3 kaminsky_remote_lab10.py`[cite: 4] | AUTH-DNS log hiện `[PENDING]` và `[HIT]`. Resolver in `[CACHE] delegation poisoned`. Script báo SUCCESS.[cite: 4] | AUTH-DNS khớp batch TXID thành công (`[HIT]`). Resolver nhận forged packet và log `delegation poisoned`. Kali báo SUCCESS.[cite: 2, 4] | **PASS**[cite: 2] |
| **Tình huống 4: Kiểm chứng hậu quả trên VICTIM** | Kiểm tra lưu lượng của người dùng nạn nhân bị bẻ hướng sang máy chủ độc hại của Attacker.[cite: 4] | Đứng trên VICTIM (10.9.0.100) chạy:<br>`sudo resolvectl flush-caches`<br>`dig www.example.lab10.test`<br>`curl http://www.example.lab10.test`[cite: 2, 4] | Truy vấn trả về IP `10.10.0.200` của kẻ tấn công. Web tải về giao diện "LAB 10 - Fake Site".[cite: 4] | `dig` trả về IP `10.10.0.200`. Lệnh `curl` hiển thị mã nguồn trang "LAB 10 - Fake Site".[cite: 2, 4] | **PASS**[cite: 2] |
| **Tình huống 5: Khôi phục BIND9 & Hardening (SPR)** | Khôi phục BIND9 hiện đại, xác minh cơ chế ngẫu nhiên hóa cổng nguồn UDP chống tấn công Brute-force.[cite: 4] | 1. Dừng resolver mô phỏng, chạy `sudo systemctl start bind9`<br>2. Trên DNS-SERVER: `sudo tcpdump -ni any host 172.16.6.20 and udp port 53`<br>3. Trên VICTIM: Gửi nhiều truy vấn subdomain ngẫu nhiên liên tục.[cite: 4] | Cổng nguồn UDP upstream của DNS-SERVER liên tục thay đổi ngẫu nhiên qua các phiên (khác xa port cố định 33333).[cite: 4] | Tcpdump ghi nhận UDP source port biến thiên liên tục trong dải Ephemeral ports (>1024), vô hiệu hóa tấn công Kaminsky.[cite: 2, 4] | **PASS**[cite: 2] |

---

## 5. CÁC LỖI GẶP PHẢI VÀ BIỆN PHÁP KHẮC PHỤC

* **Lỗi 1: Sau khi sửa firewall rules hoặc reset mô phỏng, trạng thái kết nối vẫn giữ nguyên**[cite: 2]
  * *Hiện tượng:* Rule mới đã Apply nhưng lệnh ping hoặc gói tin DNS vẫn xử lý theo logic cũ.
  * *Nguyên nhân:* pfSense là Stateful Firewall; các phiên đang mở được duy trì trong State Table nên gói tin kế tiếp đi qua tự động mà không so khớp lại ruleset.[cite: 2]
  * *Cách khắc phục:* Vào **Diagnostics** > **States** > tab **Reset States**, chọn *Reset the firewall state table* và nhấn **Reset**.[cite: 2]
* **Lỗi 2: Tấn công thành công nhưng máy VICTIM vẫn phân giải ra IP cũ 172.16.6.10**[cite: 4]
  * *Hiện tượng:* Script của Kali đã in `SUCCESS` nhưng máy VICTIM không nhận IP giả mạo.[cite: 4]
  * *Nguyên nhân:* Bộ nhớ đệm cục bộ trên máy khách (systemd-resolved / DNS Client Cache) chưa được làm sạch, hoặc tiến trình BIND9 chưa dừng triệt để khiến port 53 vẫn do BIND9 chiếm giữ thay vì resolver mô phỏng.[cite: 4]
  * *Cách khắc phục:* Chạy `sudo resolvectl flush-caches` trên VICTIM và kiểm tra bằng `sudo ss -lunp | grep :53` trên DNS-SERVER để đảm bảo script `vuln_resolver_lab10.py` đang lắng nghe.[cite: 4]
* **Lỗi 3: Truy vấn sau khi bị poison bị timeout (không mở được trang Web giả mạo)**[cite: 4]
  * *Hiện tượng:* Resolver đã nhận ủy quyền nhưng VICTIM không nhận được IP hoặc lệnh `curl` tới port 80 bị treo.[cite: 4]
  * *Nguyên nhân:* Thiếu firewall rule cho phép Resolver (`10.9.0.10`) gửi truy vấn UDP 53 sang Kali (`10.10.0.200`) hoặc thiếu rule cho phép dải `DNSLAB` truy cập HTTP `TCP 80` sang Kali trên pfSense.[cite: 4]
  * *Cách khắc phục:* Thêm các rule tương ứng trên tab giao diện `DNSLAB` của pfSense và Reset States.[cite: 4]
