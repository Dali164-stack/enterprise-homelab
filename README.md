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
---

## 3. Compute Infrastructure Node: Ubuntu Server
The primary internal workload node has been successfully provisioned running a headless **Ubuntu Server 24.04 LTS** platform architecture.

### Host Profile & Administrative Credentials
* **Hostname:** `ubuntu-srv`
* **Primary Administrative User:** `sysadmin`
* **Access Method:** OpenSSH Server enabled via port `22` (Password Authentication active).

### Interface Mapping & Network Verification
The host relies on a single virtual network adapter mapped strictly to the isolated corporate LAN switch.

* **Primary Interface:** `ens33`
* **Network Assignment Engine:** DHCP via pfSense Firewall Appliance.
* **Assigned IPv4 Address:** **`192.168.100.10`** with a subnet mask allocation of **`/24`**.
* **Default Network Gateway:** `192.168.100.1` (pfSense LAN Interface).

### Baseline Network Verification Logs
* **Internal Connectivity Test:** `ping -c 4 192.168.100.1` execution successful. **0% packet loss** verified. 
* **Layer 2 Hypervisor Switching:** Confirmed operational on switch `VMnet1`.
### Edge Routing & DNS Troubleshooting Logs
* **Diagnostic Incident:** Initial deployment showed successful external IP routing (`ping 8.8.8.8`), but full domain name resolution failed (`temporary failure in name resolution`).
* **Root Cause Analysis:** The edge gateway (pfSense) lacked explicit upstream DNS server configurations in its core kernel routing engine, resulting in blocked or dropped DNS queries (Port 53 UDP/TCP) originating from the `VMnet1` segment.
* **Remediation Implemented:** 
  1. Configured the Windows Host `VMnet1` adapter to static IP `192.168.100.99/24` to establish a management bridge.
  2. Bypassed self-signed certificate warnings to access the pfSense web configuration interface at `https://192.168.100.1`.
  3. Committed Google (`8.8.8.8`) and Cloudflare (`1.1.1.1`) to **System > General Setup > DNS Server Settings** with DNS Override features active.
* **Post-Remediation Verification:** Complete end-to-end outbound DNS resolution successfully restored. Command `ping -c 4 google.com` executed on `ubuntu-srv` yields **0% packet loss**.
### 3.1 Host Security Hardening & Production Rules Matrix
To transition the compute workload from a troubleshooting state to a hardened production deployment, the local kernel-level defense parameters have been completely re-initialized:

* **Host-Based Firewall (UFW Status):** `Active`
* **Default Security Policy:** Deny (Incoming), Allow (Outgoing), Disabled (Routed)
* **Active Firewall Access Control Rules:**
  ```text
  To                         Action      From
  --                         ------      ----
  22/tcp                     ALLOW IN    Anywhere
  22/tcp (v6)                ALLOW IN    Anywhere (IPv6)
  ```
* **Credential Complexity Compliance:** Account `sysadmin` credentials have been migrated to a fully compliant corporate complexity string, satisfying internal dictionary check mechanisms (`pam_pwquality`).
* **Management Persistence Status:** End-to-end verification confirms remote management access over Port 22 is stable, resilient across host restarts, and functioning at static anchor address `192.168.100.10`.
### 3.2 Graphical Remote Management (XRDP Gateway)
To provide administrative flexibility without relying on the hypervisor framework, a secure remote graphical pipeline has been established:

* **Remote Display Engine:** XRDP Server (Xorg back-end architecture)
* **Active Networking Port:** `3389/tcp` (Hardened via local UFW profile rules)
* **Production Status:** Confirmed operational. Graphical desktop management sessions launch seamlessly via native Windows Remote Desktop Connection (`mstsc`) over static channel `192.168.100.10`.
* **Session Integrity Rules:** Validated that local VMware console sessions must remain logged out to prevent multi-session display rendering crashes.

### 3.3 Permanent Local Routing Engine Stabilization
Following infrastructure power cycles, the compute node experienced localized routing table truncation, dropping the gateway configuration and triggering hard `Network is unreachable` kernel states.

#### Root Cause Analysis
NetworkManager configuration profiles handling dynamic DHCP allocations failed to persistently bind the upstream perimeter gateway anchor metric upon initialization loops.

#### Remediation Implemented
1. **Volatile Kernel Patch:** Executed `sudo ip route add default via 192.168.100.1 dev ens33` to instantly restore active application-layer transit routes.
2. **Persistent Netplan Hardening:** Modified the core system network configurations under `/etc/netplan/` to explicitly enforce fallback gateway bindings beneath the active rendering daemon.

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
      dhcp4: true
      routes:
        - to: default
          via: 192.168.100.1
```

#### Post-Remediation Verification Logs
* **Reboot Persistence Test:** Triggered `sudo reboot`. Verified that the `default via 192.168.100.1 proto dhcp` channel instantiates automatically on cold initialization.
* **Stateful Firewall Integrity Check:** Executing `curl -I https://google.com` yields immediate `HTTP/2 200` payloads, validating that local outbound routing remains online and synchronized with perimeter rulesets.

## 4. Perimeter Edge Security Policies (pfSense Core Rules)
The edge firewall appliance utilizes strict stateful packet inspection. The following access control entries (ACE) have been deployed on the private LAN segment (`VMnet1`):

### Active Rules Matrix (LAN Interface)

| Order | Action | Source | Port | Destination | Protocol | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | **BLOCK** | `192.168.100.10` | * | Any | ICMP (Echo Req) | Block Ubuntu Outbound Ping Diagnostics |
| 2 | **BLOCK** | Any | * | LAN Address | TCP (80) | Block unencrypted HTTP management Access |
| 3 | **ALLOW** | LAN net | * | Any | Any | Default allow LAN to any rule |

### Live Operational Verification Logs
* **ICMP Egress Control Test:** Command `ping -c 4 8.8.8.8` executed on `ubuntu-srv` results in **100% packet loss** (4 packets transmitted, 0 received). Confirmed firewall drop behavior.
* **HTTP/HTTPS Egress Transit Test:** Command `curl -I https://google.com` returns a clean `HTTP 200 OK` response. Confirmed application-layer data flow remains fully intact.

### 4.1 Strict Management Plane Hardening (Port 80 De-provisioning)
To align the perimeter edge framework with zero-trust corporate security standards, unencrypted web management vectors have been strictly deprecated.

#### Step 1: WebConfigurator Anti-Lockout / Redirect Override
* **Configuration Path:** `System > Advanced > Admin Access`
* **Variable Change:** Enabled **"Disable webConfigurator redirect rule"**.
* **Engineering Impact:** Removed the hidden default firewall rule allowing pfSense to listen on port 80 for automatic upgrades. This forces encryption at the network boundary.

#### Step 2: Custom Access Control Entry Deployment
* **Action:** `Block`
* **Source:** `Any` (Global subnet coverage)
* **Destination:** `LAN Address` (pfSense Gateway)
* **Destination Port:** `80 (HTTP)`
* **Status:** Operational. Applied as Rule Order #2 on the LAN interface.

#### Post-Remediation Verification Logs
To ensure browser HSTS and local caches did not obscure testing results, raw socket execution was validated via command line:
* **Host-Side Browser Verification (Incognito Mode):** Navigating to `http://192.168.100.1` results in an immediate connection timeout/unreachable page.
* **Compute Node Terminal Verification:** Executing `curl -Iv http://192.168.100.1` on `ubuntu-srv` fails to establish a 3-way handshake on port 80, terminating in a hard connection failure.
* **Encrypted Access Validation:** Forcing a connection via `https://192.168.100.1` successfully builds a secure TLS session and exposes the webConfigurator dashboard.