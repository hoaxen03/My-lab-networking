# Enterprise Network Design Simulation (Cisco Packet Tracer)

![Network Topology](sodolab.png)


## Tổng quan dự án (Project Overview)
Dự án này mô phỏng hạ tầng mạng cho một doanh nghiệp vừa và nhỏ (SME), được thiết kế và cấu hình trên Cisco Packet Tracer. Hệ thống bao gồm phân vùng mạng LAN cho các phòng ban, vùng Server Farm tách biệt, và kết nối ra Internet giả lập thông qua Router biên.

Mục tiêu của lab là thể hiện kỹ năng cấu hình **Routing & Switching**, quản lý dịch vụ mạng (**DHCP, DNS, Web**) và triển khai các chính sách bảo mật cơ bản (**ACL, NAT**).

## Kiến trúc mạng (Architecture)

Mô hình được thiết kế theo cấu trúc phân lớp (Hierarchical Design):

*   **Core/Distribution Layer:** Sử dụng **Multilayer Switch** để thực hiện định tuyến giữa các VLAN (Inter-VLAN Routing) và áp dụng các chính sách truy cập (ACL).
*   **Access Layer:** Sử dụng Switch Layer 2 (Cisco 2960) kết nối các thiết bị đầu cuối.
*   **Edge Layer:** Router biên (ISR4331) thực hiện NAT và định tuyến ra Internet.
*   **Server Farm:** Khu vực riêng biệt chứa các server quan trọng (Web, DNS, DHCP).

### Quy hoạch VLAN & IP Subnet

| VLAN ID | Tên vùng (Name) | Subnet | Mô tả |
| :--- | :--- | :--- | :--- |
| **100** | Nhân sự (HR) | `153.27.10.0/24` | Vùng truy cập dành cho phòng nhân sự. |
| **200** | Kế toán (ACC) | `153.27.20.0/24` | Vùng truy cập hạn chế, bảo mật cao. |
| **300** | Wifi (Guest) | `153.27.30.0/24` | Mạng không dây, bị hạn chế truy cập server nội bộ. |
| **Server** | Server Farm | `10.20.10.0/25` | Chứa Web Server, DNS Server, DHCP Server. |
| **WAN** | Internet Sim | `8.8.8.0/24` | Giả lập kết nối ISP. |

## Tính năng & Công nghệ triển khai

### 1. Routing & Switching
*   **Inter-VLAN Routing:** Cấu hình SVI trên Multilayer Switch để các VLAN giao tiếp với nhau.
*   **Static Routing:** Định tuyến tĩnh từ Core Switch ra Router biên và Default Route ra Internet.
*   **Trunking (802.1Q):** Cấu hình đường trục giữa các Switch.

### 2. Dịch vụ mạng (Network Services)
*   **DHCP Relay:** Sử dụng `ip helper-address` trên Multilayer Switch để chuyển tiếp yêu cầu xin IP từ PC ở các VLAN tới DHCP Server (10.20.10.30) nằm ở vùng mạng khác.
*   **NAT/PAT (Network Address Translation):** Cấu hình NAT Overload trên Router biên (Router 2) cho phép nhiều thiết bị nội bộ truy cập Internet thông qua một Public IP.

### 3. Bảo mật (Security & ACL)
*   **Access Control Lists (ACL):**
    *   **ACL 101 (VLAN 300 - Wifi):** Chặn vùng Wifi truy cập vào dải quản trị Server, chỉ cho phép truy cập Web (80/443) và DNS (53).
    *   **Standard ACL:** Kiểm soát lưu lượng đi ra Internet.

---

## Chi tiết cấu hình (Configuration Details)

Dưới đây là cấu hình chi tiết trích xuất từ các thiết bị trong sơ đồ:

<details>
<summary><b> Click để xem cấu hình: Router 2 (Gateway/NAT Router)</b></summary>

```cisco
hostname Router
!
interface GigabitEthernet0/0/0
 ip address 10.10.10.2 255.255.255.248
 ip nat inside
!
interface GigabitEthernet0/0/1
 ip address 10.20.10.1 255.255.255.128
 ip nat inside
!
interface Serial0/1/0
 ip address 8.8.8.1 255.255.255.0
 ip nat outside
 clock rate 2000000
!
! Cấu hình NAT Overload
ip nat pool pool1 8.8.8.1 8.8.8.5 netmask 255.255.255.0
ip nat inside source list 1 pool pool1 overload
!
! Routing
ip route 0.0.0.0 0.0.0.0 8.8.8.8 
ip route 153.27.0.0 255.255.0.0 10.10.10.1 
!
access-list 1 permit 153.27.10.0 0.0.0.255
access-list 1 permit 153.27.20.0 0.0.0.255
access-list 1 permit 153.27.30.0 0.0.0.255
access-list 1 permit 10.20.0.0 0.0.255.127
```
</details>

<details>
<summary><b> Click để xem cấu hình: Multilayer Switch0 (Core/Distribution)</b></summary>

```cisco
hostname Switch
ip routing
!
interface Vlan100
 ip address 153.27.10.1 255.255.255.0
 ip helper-address 10.20.10.30
 ip access-group 1 out
!
interface Vlan200
 ip address 153.27.20.1 255.255.255.0
 ip helper-address 10.20.10.30
 ip access-group 1 out
!
interface Vlan300
 ip address 153.27.30.1 255.255.255.0
 ip helper-address 10.20.10.30
 ip access-group 101 in
!
ip route 0.0.0.0 0.0.0.0 10.10.10.2 
!
! Security Policies
access-list 1 deny 153.27.30.0 0.0.0.255
access-list 1 permit any
!
! ACL for Wifi Zone (VLAN 300)
access-list 101 permit udp 153.27.30.0 0.0.0.255 host 10.20.10.20 eq domain
access-list 101 permit tcp 153.27.30.0 0.0.0.255 host 10.20.10.10 eq www
access-list 101 permit tcp 153.27.30.0 0.0.0.255 host 10.20.10.10 eq 443
access-list 101 deny ip 153.27.30.0 0.0.0.255 10.20.10.0 0.0.0.127
access-list 101 permit ip any any
```
</details>

<details>
<summary><b> Click để xem cấu hình: Switch 1 & Switch 2 (Access Layer)</b></summary>

```cisco
! Cấu hình chung cho Access Switch
interface FastEthernet0/1
 switchport trunk allowed vlan 100,200,300
 switchport mode trunk
!
interface FastEthernet0/2
 switchport access vlan 200 (hoặc 300)
 switchport mode access
!
interface FastEthernet0/3
 switchport access vlan 100 (hoặc 200)
 switchport mode access
!
interface FastEthernet0/4
 switchport access vlan 300 (hoặc 100)
 switchport mode access
```
</details>

<details>
<summary><b> Click để xem cấu hình: Router 1 (ISP Simulation)</b></summary>

```cisco
hostname Router
!
interface Serial0/1/0
 ip address 8.8.8.8 255.255.255.0
!
```
</details>

---

## Hướng phát triển (Future Improvements)
Để nâng cao tính sẵn sàng và bảo mật cho hệ thống, các giải pháp sau có thể được triển khai thêm:
1.  **High Availability (HA):** Triển khai giao thức **HSRP/VRRP** bằng cách thêm một Router và Core Switch dự phòng để tránh điểm chết đơn lẻ (Single Point of Failure).
2.  **VPN Site-to-Site:** Cấu hình IPSec VPN để kết nối an toàn với các chi nhánh khác.
3.  **Port Security:** Cấu hình trên Access Switch để ngăn chặn việc cắm thiết bị lạ vào mạng.
4.  **EtherChannel:** Gộp băng thông các đường link giữa Switch và Router để tăng tốc độ và dự phòng.

## 👨‍💻 Author
*   **Name:** Vũ Thành Trung
*   **Contact:** vttrung11a@gmail.com
