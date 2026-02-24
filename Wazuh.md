### 3. Hướng dẫn thực thi hỏa tốc (Data Flow thực chiến)

Tôi sẽ không cầm tay chỉ việc từng cú click, mà tôi đưa cho cậu luồng đi (Data Flow). Cậu hãy làm theo trình tự này:

**Bước 1: Dựng Wazuh Manager (Trên `wazuh-manager`)** Wazuh có sẵn script All-in-one cực kỳ mạnh. Cậu chỉ cần SSH vào Ubuntu, chạy quyền root và gõ đúng 1 lệnh (nhớ lấy IP của con Ubuntu này):

Bash

```
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

_(Script này tự động cài Indexer, Server, và Dashboard. Đi pha ly cà phê, 15 phút sau nó trả về user `admin` và password cho cậu đăng nhập web)._

![](_assets/Pasted%20image%2020260224105439.png)


![](_assets/Pasted%20image%2020260224110650.png)

```bash
24/02/2026 11:01:56 INFO: --- Summary ---
24/02/2026 11:01:56 INFO: You can access the web interface https://<wazuh-dashboard-ip>:443
    User: admin
    Password: 1N8oW.Px5q3FhV6?tKMITfV7ttKccvL3
24/02/2026 11:01:56 INFO: Installation finished.
root@ubuntu:~#
```

Hai port 
Giao tiếp Agent - Manager: 1514 (TCP, UDP)
1515 : Xác thực Agent mới

![](_assets/Pasted%20image%2020260224110836.png)

![](_assets/Pasted%20image%2020260224110845.png)


![](_assets/Pasted%20image%2020260224110917.png)

![](_assets/Pasted%20image%2020260224110939.png)

- **Đăng nhập Web UI:** Mở trình duyệt web trên máy tính thật của cậu (Windows). Gõ `https://<IP_của_máy_Ubuntu>`. _Lưu ý: Trình duyệt sẽ cảnh báo "Kết nối không an toàn" (Your connection is not private) vì đây là chứng chỉ SSL tự sinh (Self-signed certificate). Cứ nhấn Advanced -> Proceed to (Tiếp tục truy cập)._
    
- **Lấy lệnh cài đặt Agent:** Đăng nhập bằng `admin` và password ở trên. Chờ hệ thống check API vài chục giây. Sau khi vào Dashboard chính, tìm mục **Add agent** (hoặc nhấn vào icon Wazuh góc trên bên trái -> **Agents** -> **Deploy new agent**).
![](_assets/Pasted%20image%2020260224111158.png)


- **Cấu hình tham số:** * Chọn OS: **Debian/Ubuntu** (.deb).
    
    - Architecture: **x86_64**.
        
    - Wazuh server address: Nhập **IP của con Ubuntu**.

![](_assets/Pasted%20image%2020260224111432.png)

![](_assets/Pasted%20image%2020260224111536.png)

![](_assets/Pasted%20image%2020260224111549.png)



- **Thực thi trên mục tiêu:** Giao diện web sẽ tự động sinh ra một câu lệnh `curl` ở ô bước 4. Copy nguyên câu lệnh đó, mở Terminal của con máy ảo `debian-12-x64` lên và dán vào chạy.

![](_assets/Pasted%20image%2020260224112202.png)

Pha xử lý sự cố (Troubleshooting) vừa rồi của cậu xứng đáng điểm 10 thực chiến! Cậu đã áp dụng đúng tư duy cô lập lỗi (Fault Isolation) của mô hình OSI.

Khi lệnh `wget` báo lỗi tên miền, cậu lập tức `ping 8.8.8.8`. Ping thông mạng nghĩa là Routing (Layer 3) ra Internet hoàn toàn bình thường, không bị Firewall hay NAT chặn. Vấn đề nằm duy nhất ở dịch vụ DNS (Phân giải tên miền). Máy của cậu đang không có "cuốn danh bạ" để dịch chữ `packages.wazuh.com` ra địa chỉ IP.

Tình trạng này rất hay gặp khi set IP tĩnh (Static IP) trên Linux mà quên cấu hình thông số DNS server, hoặc do DHCP cấu hình sai.

Tình trạng này rất hay gặp khi set IP tĩnh (Static IP) trên Linux mà quên cấu hình thông số DNS server, hoặc do DHCP cấu hình sai.

**Cách khắc phục (Fix lỗi DNS hỏa tốc):**

Vì cậu đang ở sẵn quyền `root`, hãy ép thẳng một Public DNS (của Google) vào file cấu hình phân giải tên miền bằng lệnh sau:

Bash

```
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Gõ lệnh trên xong, hệ thống sẽ lập tức nhận DNS mới. Cậu hãy nhấn nút mũi tên lên (Up arrow) để chạy lại nguyên cái lệnh `wget ... && dpkg -i ...` lúc nãy. Lần này chắc chắn nó sẽ tải vù vù.

![](_assets/Pasted%20image%2020260224112559.png)





- **Kích hoạt:** Sau khi tải xong, chạy tiếp 2 lệnh để khởi động Agent (Wazuh UI cũng có in sẵn cho cậu copy):
    
    Bash
    
    ```
    sudo systemctl daemon-reload
    sudo systemctl enable wazuh-agent
    sudo systemctl start wazuh-agent
    ```


![](_assets/Pasted%20image%2020260224112718.png)

![](_assets/Pasted%20image%2020260224112744.png)

![](_assets/Pasted%20image%2020260224115957.png)

![](_assets/Pasted%20image%2020260224120318.png)

![](_assets/Pasted%20image%2020260224120418.png)

Hãy nhìn lại chính đống output mà cậu vừa dán lên đây. Cậu hãy cuộn tìm đúng dòng này cho tôi: `tcp 0 0 192.168.15.225:55284 192.168.15.125:1514 ESTABLISHED`

**Giải mã luồng dữ liệu (Tại sao lại không thấy LISTEN):**

1. **Ai là người "Lắng nghe" (LISTEN)?** Wazuh Manager (máy Ubuntu - `192.168.15.125`) đóng vai trò là Tổng đài (Server). Nó là thằng mở port 1514 và 1515 ra và đặt ở trạng thái `LISTEN` (Chờ người khác gọi đến). Nếu cậu gõ `netstat -tulpn` trên máy **Ubuntu**, cậu sẽ thấy port 1514 đang LISTEN.
    
2. **Ai là người "Gọi" (Initiator/Client)?** Máy Debian của cậu (`192.168.15.225`) là Agent (Client). Khi nó muốn gửi log về cho Manager, hệ điều hành Debian sẽ tự động mở một **port ngẫu nhiên (Ephemeral port)** - trong trường hợp này là port `55284` - để "gọi" sang port đích `1514` của Manager. Đó là lý do cậu thấy trạng thái là `ESTABLISHED` (Đã kết nối) chứ không phải `LISTEN`. Cậu không bao giờ mở port 1514 trên Agent để chờ Manager gọi đến cả!
    
3. **Port 1515 đâu?** Port 1515 chỉ dùng để xác thực (Enrollment) lúc cậu mới chạy lệnh cài Agent. Lấy chứng chỉ xong là nó ngắt kết nối luôn. Dòng dữ liệu log hiện tại và mãi về sau sẽ chỉ chạy qua port 1514.


![](_assets/Pasted%20image%2020260224131614.png)

![](_assets/Pasted%20image%2020260224131632.png)

![](_assets/Pasted%20image%2020260224131646.png)

