# Triển khai SIEM Tập trung: Giám sát An ninh Tổng đài VoIP (FusionPBX) & OS

**Vai trò:** System/Security Administrator
**Mục tiêu (Objective):** Triển khai Wazuh SIEM trên môi trường Lab để thu thập log tập trung. Xây dựng Custom Parser và Ruleset nhằm phát hiện, cảnh báo thời gian thực các cuộc tấn công Brute-force vào dịch vụ cốt lõi (SIP/FreeSWITCH) và tầng OS (SSH).

**Kiến trúc Hệ thống:**
* **Wazuh Manager (Server):** Ubuntu 24.04 (`192.168.15.125`)
* **Wazuh Agent (Target):** Debian 12 chạy FusionPBX/FreeSWITCH (`192.168.15.225`)

### Link demo Wazuh detect Bruteforce SIP/VoIP: https://youtu.be/QOj0m5xKv0k
### Link demo Wazuh detect Bruteforce SSH: https://youtu.be/XM_Gu42Pb2s

## Phase 1: Giám sát An ninh Chuyên sâu - VoIP/SIP 

**Bối cảnh:** Khác với các dịch vụ tiêu chuẩn, hệ thống SIEM mặc định không hỗ trợ đọc hiểu log của tổng đài FusionPBX (FreeSWITCH). Để giám sát các luồng tấn công ngập lụt (SIP Flood) và dò quét mật khẩu vào port `5060`, hệ thống cần được chỉ định luồng log, xây dựng Custom Decoder và Custom Rules.

### 1.1. Thu thập & Phân tích Luồng tấn công (Traffic & Raw Log)
Sử dụng công cụ `sngrep` để bắt các luồng tin SIP REGISTER rác ở lớp mạng, đồng thời trích xuất Raw Log từ FreeSWITCH khi bị tấn công bằng Nmap.

![Mô phỏng tấn công bằng Nmap](_assets/Pasted%20image%2020260224161103.png)

![Phân tích gói tin ngập lụt bằng sngrep](_assets/Pasted%20image%2020260224171440.png)

![Raw Log của FreeSWITCH văng lỗi Auth Failure](_assets/Pasted%20image%2020260224161147.png)

### 1.2. Phẫu thuật Dữ liệu: Custom Decoder với PCRE2
**Sự cố (Troubleshooting):** Log của FreeSWITCH chứa các ký tự đặc biệt `[]` gây lỗi vỡ cấu trúc `XML Regex` mặc định của Wazuh (`wazuh-analysisd: ERROR 1452: Syntax error on regex`).

![Lỗi Syntax XML khi khởi động Wazuh Manager](_assets/Pasted%20image%2020260224161835.png)

**Giải pháp:** Chuyển đổi Engine phân tích sang chuẩn `PCRE2` và sử dụng kỹ thuật Escaping `\` để bóc tách chính xác mục tiêu (`srcuser`) và nguồn tấn công (`srcip`).

![Cấu hình Custom Decoder với PCRE2](_assets/Pasted%20image%2020260224170704.png)

![Kiểm thử bóc tách dữ liệu mượt mà bằng công cụ wazuh-logtest](_assets/Pasted%20image%2020260224170825.png)

### 1.3. Khai báo Luật (Custom Rules) & MITRE ATT&CK
Viết tập luật cảnh báo (Level 5 & Level 10) dựa trên tần suất (Frequency Thresholds) để chống báo động giả, nội suy biến số để hiển thị trực quan và ánh xạ thẳng vào chuẩn MITRE ATT&CK.

![Cấu hình Custom Rules cho FreeSWITCH](_assets/Pasted%20image%2020260224170245.png)

### 1.4. Xử lý sự cố "Ngập lụt Buffer" (Rate Limiting)
Khi Nmap nã >50.000 requests, hàng đợi (Event Queue) của Wazuh Agent bị quá tải, tự động drop log để bảo vệ OS, dẫn đến hiện tượng "Cảnh báo câm".

![Cảnh báo Rule 203: Agent event queue is full](_assets/Pasted%20image%2020260224164019.png)

**Hành động khắc phục:** 1. Tối ưu hóa Buffer trên Agent bằng cách nới lỏng giới hạn trong file `local_internal_options.conf`.
![Nới lỏng giới hạn Buffer của Agent](_assets/Pasted%20image%2020260224164210.png)

2. Sử dụng tham số `--max-rate 50` trên Nmap để điều tiết băng thông mô phỏng tấn công.
![Điều tiết hỏa lực bằng max-rate](_assets/Pasted%20image%2020260224171447.png)

### 1.5. Kết quả Phase 1
Hệ thống Manager tiếp nhận mượt mà hàng ngàn sự kiện, kích hoạt thành công Cảnh báo Đỏ (Rule 100002) ứng với mã **MITRE T1110.001**, đồng thời gắp chính xác thông tin User bị tấn công.

![Biểu đồ nhận diện >2800 lượt dò quét SIP](_assets/Pasted%20image%2020260224171352.png)

![Chi tiết Rule 100001 và 100002 nảy liên tục với dữ liệu User/IP đầy đủ](_assets/Pasted%20image%2020260224171421.png)


## Phase 2: Giám sát OS & SSH Brute-force

### 2.1. Cấu hình luồng Log & Khắc phục kiến trúc OS
**Sự cố:** Debian 12 đã loại bỏ `rsyslog` mặc định, chuyển sang dùng `journald`. Hệ thống không sinh ra file vật lý `/var/log/auth.log` khiến Wazuh Agent bị "mù".
**Giải pháp:** Cài đặt lại gói `rsyslog` và chỉ định `<localfile>` trong cấu hình OSSEC để ép Agent đọc đúng luồng.

![Cấu hình OSSEC đọc auth.log](_assets/Pasted%20image%2020260224141112.png)

### 2.2. Kiểm thử và Kết quả
Sử dụng script vòng lặp `for` trên PowerShell của Windows để nã hàng chục nỗ lực đăng nhập sai vào cổng 22.

![Mô phỏng SSH Brute-force bằng PowerShell](_assets/image_42b035.png)

Hệ thống ghi nhận và ánh xạ hoàn hảo vào Rule `5710` (Attempt to login using a non-existent user).
![Biểu đồ 72 hits từ Dashboard](_assets/image_433ddd.png)

![Chi tiết ánh xạ MITRE T1021.004 cho SSH](_assets/image_433b13.png)


## Appendix: Deploy & Trouble-shooting System (Layer 3/4)

Ghi nhận lại quá trình xử lý sự cố thiết lập nền tảng ban đầu:

**1. Lỗi phân giải tên miền (DNS Resolution) khi cài Agent:**
Do Server thiết lập Static IP thiếu DNS, lệnh `wget` tải gói cài đặt Agent bị lỗi. Xử lý hỏa tốc bằng cách can thiệp `sed` chuyển Mirror Ubuntu quốc tế (nếu cần) và nạp Public DNS `8.8.8.8` vào `/etc/resolv.conf`.
![Khắc phục lỗi DNS khi cài Agent](_assets/Pasted%20image%2020260224112559.png)

**2. Phân tích luồng kết nối (Data Flow):**
Sử dụng `netstat` kiểm chứng kiến trúc Client-Server. Agent (Debian) sử dụng cổng ngẫu nhiên (Ephemeral Port `33608`) để chủ động thiết lập kết nối (ESTABLISHED) đến Port `1514` đang Lắng nghe (LISTEN) trên Manager.
![Luồng kết nối hiển thị qua netstat](_assets/Pasted%20image%2020260224120318.png)

**3. Triển khai Wazuh Manager (All-in-one Architecture):**
Thực thi Installation Script (Quickstart) để cài đặt toàn bộ stack (Indexer, Server, Dashboard) lên máy chủ Ubuntu `192.168.15.125`. 
![Cài đặt Manager hoàn tất kèm thông tin Credentials](_assets/Pasted%20image%2020260224110650.png)

*Lưu ý kiến trúc (Self-signed Certificate):* Ở môi trường Lab, Wazuh tự động sinh chứng chỉ SSL tự ký (Self-signed). Trình duyệt sẽ cảnh báo "Connection isn't private" là hành vi bình thường. Chấp nhận rủi ro để truy cập UI.
![Cảnh báo SSL tự ký trên Chrome](_assets/Pasted%20image%2020260224110836.png)

![Giao diện đăng nhập Wazuh Dashboard](_assets/Pasted%20image%2020260224110917.png)

**4. Xác minh trạng thái Listening Port (Manager side):**
Sử dụng `netstat -tlunp` trên con Ubuntu để đảm bảo các tiến trình lõi của SIEM đã được bind đúng cổng, sẵn sàng đón lõng Agent:
- Port `1514/tcp`: `wazuh-remoted` (Cổng giao tiếp/nhận log chính).
- Port `1515/tcp`: `wazuh-authd` (Cổng tiếp nhận đăng ký/Enrollment cho Agent mới).
- Port `55000/tcp`: `python3` (Wazuh API).
![Kiểm tra tiến trình và Port trên Manager](_assets/Pasted%20image%2020260224120418.png)

**5. Đăng ký & Kích hoạt Agent (Enrollment Process):**
Thực thi lệnh cài đặt sinh ra từ Dashboard trên con Debian. Gán cứng biến môi trường `WAZUH_MANAGER='192.168.15.125'` và `WAZUH_AGENT_NAME='agent-debian-fusionpbx'`.
Sau khi sửa lỗi DNS, package được cài đặt. Khởi động dịch vụ bằng `systemctl`:
![Enable và Start service wazuh-agent](_assets/Pasted%20image%2020260224112718.png)

**Kết quả nền tảng:** Giao tiếp TLS giữa Agent và Manager thiết lập thành công. Agent hiển thị trạng thái `Active` với độ bao phủ `100.00%`, sẵn sàng cho bước Cấu hình Localfile.
![Trạng thái Agent Active trên Dashboard](_assets/Pasted%20image%2020260224112744.png)

![Chi tiết thông tin Agent đã enroll](_assets/Pasted%20image%2020260224115957.png)
