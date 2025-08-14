# GNS3 Enterprise Network with Site-to-Site VPN

## 📜 Project Overview

This project, built entirely within GNS3, simulates a realistic network infrastructure for a small enterprise with a main Head Office and a remote Branch Office. The two offices are connected securely over a simulated public internet using a **Site-to-Site GRE over IPsec VPN tunnel**.

The primary objective is to demonstrate a comprehensive range of fundamental and advanced networking skills, from initial LAN design and segmentation to secure multi-site WAN connectivity.

## 🗺️ Network Topology

<img width="2845" height="1561" alt="github" src="https://github.com/user-attachments/assets/c8502417-1461-408f-8d17-859bafdc2eaa" />

## ✨ Key Skills Demonstrated

This single project showcases a comprehensive set of highly valuable, real-world networking skills:

* **Network Design:** Implementation of a **"Collapsed Core"** topology in the Head Office, a common and efficient model for small to medium-sized enterprise networks where core and distribution functions are combined into a single, powerful device.
* **VLANs:** Segmentation of the Head Office network into distinct departments (HR, IT, Marketing) for enhanced security and network organization.
* **Inter-VLAN Routing:** Configuration of a **"Router on a Stick" (ROAS)** on the edge router to enable controlled communication between different VLANs.
* **Layer 3 Switching (Simulation):** Use of a Cisco C3745 router with an `NM-16ESW` switching module to simulate the functionality of a Layer 3 Catalyst switch, a standard and necessary practice in GNS3.
* **Network Address Translation (PAT):** Configuration of Port Address Translation (NAT Overload) on the Head Office edge router to allow all internal users to access the internet using a single public IP address.
* **Access Control Lists (ACLs):** Advanced use of extended ACLs to intelligently select traffic for NAT while specifically excluding private site-to-site traffic destined for the VPN.
* **Static Routing:** Implementation of static routes for WAN connectivity and to direct internal traffic through the secure VPN tunnel.
* **Site-to-Site VPN (GRE over IPsec):** Establishment of a secure, encrypted tunnel between the Head Office and Branch Office to protect all inter-office data traversing the simulated public internet.

---
## 🔢 IP Addressing Scheme

The IP addressing for this project utilizes distinct `/24` private networks for each department. **This was a deliberate design choice to provide significant room for future growth and scalability.** Allocating a full `/24` subnet to each segment ensures that a large number of additional hosts, servers, and devices can be added in the future without the need to re-design or re-address the network.

| Location          | Department / Link   | Network Address     | VLAN ID |
| :---------------- | :------------------ | :------------------ | :------ |
| **Head Office**   | HR Department       | `192.168.10.0/24`   | 10      |
| **Head Office**   | IT Department       | `192.168.11.0/24`   | 11      |
| **Head Office**   | Marketing Dept      | `192.168.12.0/24`   | 12      |
| **Branch Office** | Branch LAN          | `192.168.20.0/24`   | N/A     |
| **WAN**           | HQ <--> ISP Link    | `10.0.0.0/30`       | N/A     |
| **WAN**           | ISP <--> Branch Link| `20.0.0.0/30`       | N/A     |
| **VPN Tunnel**    | Tunnel Link         | `172.16.1.0/30`     | N/A     |

---

## ⚙️ Device Configurations

Below are the final, working configurations for the key network devices in this project, organized in a logical flow.

---

### 🖥 EdgeRouter-HQ Configuration

hostname EdgeRouter-HQ
!
! Interface Configuration
!
interface Tunnel0
 description VPN Tunnel to Branch Office
 ip address 172.16.1.1 255.255.255.252
 tunnel source FastEthernet1/0
 tunnel destination 20.0.0.2
!
interface FastEthernet0/0
 no ip address
 ip nat inside
 duplex full
!
interface FastEthernet0/0.10
 description Gateway for HR (VLAN 10)
 encapsulation dot1q 10
 ip address 192.168.10.1 255.255.255.0
!
interface FastEthernet0/0.11
 description Gateway for IT (VLAN 11)
 encapsulation dot1q 11
 ip address 192.168.11.1 255.255.255.0
!
interface FastEthernet0/0.12
 description Gateway for Marketing (VLAN 12)
 encapsulation dot1q 12
 ip address 192.168.12.1 255.255.255.0
!
interface FastEthernet1/0
 description Link to ISP
 ip address 10.0.0.1 255.255.255.252
 ip nat outside
 crypto map VPN-MAP
 duplex full
!
! Routing Configuration
!
ip route 0.0.0.0 0.0.0.0 10.0.0.2
ip route 192.168.20.0 255.255.255.0 172.16.1.2
!
! NAT and ACL Configuration
!
ip nat inside source list 101 interface FastEthernet1/0 overload
!
access-list 101 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 101 deny   ip 192.168.11.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 101 deny   ip 192.168.12.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 101 permit ip 192.168.10.0 0.0.0.255 any
access-list 101 permit ip 192.168.11.0 0.0.0.255 any
access-list 101 permit ip 192.168.12.0 0.0.0.255 any
!
access-list 110 permit gre host 10.0.0.1 host 20.0.0.2
!
! VPN Configuration
!
crypto isakmp policy 10
 encr aes
 authentication pre-share
 hash sha
!
crypto isakmp key CiscoVPN address 20.0.0.2
!
crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac
!
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 20.0.0.2
 set transform-set VPN-SET
 match address 110
!

---

### 🖥 EtherSwitch-Router Configuration
hostname EtherSwitch-Router
!
vlan database
 vlan 10 name HR
 vlan 11 name IT
 vlan 12 name Marketing
 apply
 exit
!
configure terminal
!
interface range FastEthernet1/0 - 3
 switchport mode access
 switchport access vlan 10
!
interface range FastEthernet1/4 - 9
 switchport mode access
 switchport access vlan 11
!
interface range FastEthernet1/10 - 14
 switchport mode access
 switchport access vlan 12
!
interface FastEthernet1/15
 description Link to EdgeRouter
 switchport mode trunk
!

---
### 🖥 ISP_Router Configuration

hostname ISP_Router
!
interface FastEthernet0/0
 description Link to Head Office
 ip address 10.0.0.2 255.255.255.252
 no shutdown
!
interface FastEthernet1/0
 description Link to Branch Office
 ip address 20.0.0.1 255.255.255.252
 no shutdown
!

---
### 🖥 EdgeRouter-Branch Configuration
hostname EdgeRouter-Branch
!
! Interface Configuration
!
interface Tunnel0
 description VPN Tunnel to Head Office
 ip address 172.16.1.2 255.255.255.252
 tunnel source FastEthernet1/0
 tunnel destination 10.0.0.1
!
interface FastEthernet0/0
 description Link to Branch LAN
 ip address 192.168.20.1 255.255.255.0
 no shutdown
!
interface FastEthernet1/0
 description Link to ISP
 ip address 20.0.0.2 255.255.255.252
 crypto map VPN-MAP
 no shutdown
!
! Routing Configuration
!
ip route 0.0.0.0 0.0.0.0 20.0.0.1
ip route 192.168.10.0 255.255.255.0 172.16.1.1
ip route 192.168.11.0 255.255.255.0 172.16.1.1
ip route 192.168.12.0 255.255.255.0 172.16.1.1
!
! ACL Configuration
!
access-list 110 permit gre host 20.0.0.2 host 10.0.0.1
!
! VPN Configuration
!
crypto isakmp policy 10
 encr aes
 authentication pre-share
 hash sha
!
crypto isakmp key CiscoVPN address 10.0.0.1
!
crypto ipsec transform-set VPN-SET esp-aes esp-sha-hmac
!
crypto map VPN-MAP 10 ipsec-isakmp
 set peer 10.0.0.1
 set transform-set VPN-SET
 match address 110
!

---

## 🔧 Lab Setup & Requirements

**Software:**  
- GNS3 version 2.2 or later.

**IOS Images:**  
The following Cisco IOS images were used in this build. Users must source these images legally:  
- **Edge/ISP Routers:** `c7200-adventerprisek9-mz.152-4.M7.bin` (or similar C7200 image)  
- **Core-Switch:** `c3745-advipservicesk9-mz.124-25d.bin` (configured with an `NM-16ESW` module)  

---

## ✅ Verification

- End-to-end connectivity is verified by a **successful ping** from a host in any Head Office VLAN  
  (e.g., `192.168.10.10`) to a host in the Branch Office LAN (e.g., `192.168.20.10`).  
- The `show crypto ipsec sa` command on the edge routers will show **active packet encryption and decryption counters**, confirming:  
  - The VPN tunnel is operational.  
  - Traffic is being securely encrypted and decrypted.




