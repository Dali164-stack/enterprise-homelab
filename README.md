# Enterprise Multi-VM Homelab Data Center

## 1. Network Architecture & Topology
This repository documents the design, deployment, and management of an enterprise-grade virtualized infrastructure environment built inside **VMware Workstation Pro**. 

The architecture simulates a secure corporate network perimeter utilizing a multi-NIC firewall topology to segregate internal corporate workloads from external public routing.

### Network Segmentation Layout
* **WAN Interface (External Routing):** Configured via `VMnet8 (NAT)`. This allows the edge gateway to pull updates and route outbound traffic safely through the host network.
* **LAN Interface (Isolated Corporate Network):** Configured via a customized host-only network switch `VMnet1`. Local DHCP services from the hypervisor have been disabled to ensure the perimeter firewall acts as the sole authoritative DHCP and DNS gateway for this segment.

### IP Subnet Allocations

| Network Interface | VMware Switch | IP Subnet Scope | Authorized DHCP Gateway |
| :--- | :--- | :--- | :--- |
| **WAN Interface** | VMnet8 (NAT) | Hypervisor Assigned | VMware Network Engine |
| **LAN Internal**  | VMnet1 (Host-Only) | `192.168.100.0/24` | pfSense Firewall Appliance (Pending) |

---

## 2. Perimeter Edge Deployment: pfSense Firewall
The perimeter security gateway has been successfully provisioned as a virtual appliance running on a **FreeBSD 14 x64** kernel kernel architecture. 

### Core Interface Assignments & DHCP Scope Configuration
The network interface drivers have been statically mapped to align with hypervisor switches, and the internal local area network gateway variables have been initialized via the console interface:

* **WAN Routing:** Initialized via driver `em0`, pulling a dynamic IP configuration from the hypervisor NAT gateway (`VMnet8`) to facilitate outbound internet transit.
* **LAN Configuration:** Statically assigned to IP address **`192.168.100.1`** with a subnet mask allocation of **`/24`** (Class C network topology).
* **DHCP Scope Engine:** Enabled on the internal LAN interface. The active IP allocation lease pool is strictly bound to the range of **`192.168.100.10` through `192.168.100.50`**, reserving lower and upper host registers for future static infrastructure nodes.

---
