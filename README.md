# Lab: Triển khai Wazuh SIEM & Giám sát SSH Brute-force

**Mục tiêu (Objective):** Xây dựng hệ thống giám sát an ninh tập trung (SIEM), bóc tách log hệ thống (`auth.log`) và mô phỏng tấn công để kiểm thử các cảnh báo thời gian thực đối với hành vi dò quét mật khẩu (SSH Brute-force).

**Kiến trúc Hệ thống (Architecture):**

- **Wazuh Manager (Server):** Ubuntu Server (`192.168.15.125`)
![](_assets/Pasted%20image%2020260224134819.png)

- **Wazuh Agent (Client):** Debian 12 (`192.168.15.225` - FusionPBX Server)
![](_assets/Pasted%20image%2020260224134801.png)

## Giai đoạn 1: Khởi tạo Wazuh Manager (Ubuntu Server)

Sử dụng kịch bản cài đặt tự động (All-in-one) của Wazuh để khởi tạo Indexer, Server và Dashboard.
```bash
curl -sO https://packages.wazuh.com/4.8/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

![](_assets/Pasted%20image%2020260224110650.png)

```bash
24/02/2026 11:01:56 INFO: --- Summary ---
24/02/2026 11:01:56 INFO: You can access the web interface https://<wazuh-dashboard-ip>:443
    User: admin
    Password: 1N8oW.Px5q3FhV6?tKMITfV7ttKccvL3
24/02/2026 11:01:56 INFO: Installation finished.
root@ubuntu:~#
```

> **Cấu hình & Thông tin đăng nhập:**
> 
> - URL: `https://192.168.15.125:443`
>     
> - User: `admin`
>     
> - Password: `1N8oW.Px5q3FhV6?tKMITfV7ttKccvL3` 

**Quy hoạch Port giao tiếp cốt lõi:**

- `Port 443 (TCP)`: Giao diện Web UI (Dashboard).
    
- `Port 1514 (TCP/UDP)`: Port nhận Log từ Agent gửi về Manager.
    
- `Port 1515 (TCP)`: Port cấp phát chứng chỉ xác thực (Enrollment) cho Agent mới.

## Giai đoạn 2: Cấu hình và Triển khai Wazuh Agent (Debian 12)

![](_assets/Pasted%20image%2020260224110836.png)

![](_assets/Pasted%20image%2020260224110845.png)

![](_assets/Pasted%20image%2020260224110917.png)

![](_assets/Pasted%20image%2020260224110939.png)

### 1. Khởi tạo cấu hình trên Web UI

Đăng nhập vào Wazuh Dashboard, điều hướng tới **Agents** -> **Deploy new agent**. 

![](_assets/Pasted%20image%2020260224111158.png)

Thiết lập các thông số môi trường:

- Operating System: `Debian (Linux) (.deb)`
    
- Architecture: `amd64`
![](_assets/Pasted%20image%2020260224111432.png)

- Wazuh server address: `192.168.15.125`
![](_assets/Pasted%20image%2020260224111549.png)
### 2. Triển khai & Khắc phục sự cố (Troubleshooting DNS)

Khi thực thi lệnh `wget` để tải Agent trên Debian 12, hệ thống báo lỗi không thể phân giải tên miền `packages.wazuh.com`.

![](_assets/Pasted%20image%2020260224112202.png)

**Phân tích & Xử lý (Fault Isolation):**

- Lệnh `ping 8.8.8.8` trả về kết quả thành công $\rightarrow$ Lớp mạng (Routing/Layer 3) ra Internet hoạt động bình thường, không bị chặn bởi Firewall/NAT.
    
- **Nguyên nhân gốc rễ (Root Cause):** Hệ thống thiết lập Static IP nhưng thiếu cấu hình DNS Server.
    
- **Khắc phục hỏa tốc:** Ghi đè Public DNS của Google vào file cấu hình phân giải tên miền.

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

_Sau khi fix DNS, lệnh tải và cài đặt Agent qua `dpkg` chạy thành công._

![](_assets/Pasted%20image%2020260224112559.png)

### 3. Kích hoạt dịch vụ Agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

![](_assets/Pasted%20image%2020260224112718.png)

## Giai đoạn 3: Phân tích luồng dữ liệu mạng (Data Flow Analysis)

Kiểm tra kết nối bằng lệnh `netstat` trên máy Agent (Debian 12):

![](_assets/Pasted%20image%2020260224120318.png)

> **Phân tích trạng thái Port (Client-Server Architecture):**
> 
> Nhận diện kết nối: `tcp 0 0 192.168.15.225:55284 192.168.15.125:1514 ESTABLISHED`
> 
> 1. **Trạng thái LISTEN (Lắng nghe):** Chỉ tồn tại trên Wazuh Manager (`192.168.15.125`). Manager mở port 1514/1515 để chờ dữ liệu.
>     
> 2. **Trạng thái ESTABLISHED (Đã thiết lập):** Trên máy Debian (`192.168.15.225`), hệ điều hành tự động mở một **port ngẫu nhiên (Ephemeral port)** - cụ thể là `55284` - để chủ động kết nối (Initiator) sang port `1514` của Manager.
>     
> 3. **Về Port 1515:** Chỉ sử dụng 1 lần duy nhất cho việc cấp phát chứng chỉ (Enrollment) khi mới cài đặt Agent, sau đó ngắt kết nối. Luồng log duy trì hoàn toàn qua port 1514.
>     

![](_assets/Pasted%20image%2020260224112744.png)

## Giai đoạn 4: Thu hoạch cảnh báo (Alerts & MITRE ATT&CK)

Sau khi ép hệ thống sinh log xác thực (`/var/log/auth.log`) và định tuyến lại cấu hình đọc file `<localfile>` trên Agent, thực hiện chạy script tự động mô phỏng tấn công Brute-force SSH.

![](_assets/Pasted%20image%2020260224141112.png)

Hệ thống SIEM đã thu thập, phân tích và bắn cảnh báo thành công trên giao diện Threat Hunting:

![](_assets/Pasted%20image%2020260224131614.png)

**Kết quả nhận diện:**

- Phát hiện hàng loạt hành vi dò quét bằng thông tin xác thực sai.
    
- **Rule ID kích hoạt:** `5710` (sshd: Attempt to login using a non-existent user).
    
- **Mapping với MITRE ATT&CK Framework:** * `T1110.001` (Credential Access - Password Guessing).
    
    - `T1021.004` (Lateral Movement - SSH).

![](_assets/Pasted%20image%2020260224131632.png)

![](_assets/Pasted%20image%2020260224131646.png)

# Demo 
Link youtube: https://youtu.be/XM_Gu42Pb2s





