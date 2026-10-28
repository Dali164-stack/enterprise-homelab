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

### 3.4 Production Docker Container Engine Deployment
The compute infrastructure node has been successfully provisioned with the official Docker containerization subsystem to facilitate secure multi-tenant microservice orchestration.

#### Volatile Initialization Challenges & Resolution
* **Dynamic DHCP Drift:** Upon host infrastructure cold-boot, the compute node's IPv4 network boundary migrated dynamically from `192.168.100.10` to `192.168.100.11` due to standard DHCP lease pool cycling. Remote management targets were updated accordingly.
* **Cryptographic GPG Security Block:** Initial installation runs failed with `NO_PUBKEY 7EA0A9C3F273FCD8` signature verification anomalies. This occurred because standard plain-text armor structures (`.asc`) failed to map correctly with the strict validation engines native to Ubuntu 24.04 (Noble Numbat).

#### Remediation and Hardening Suite
The installation topology was systematically rewritten to align with strict enterprise application standards:
1. **Keyring Isolation:** Formatted a secure local storage tier at `/etc/apt/keyrings/`.
2. **Binary De-armoring:** Streamed the official cryptographic public key directly from the verified Docker source subdomain and translated the payload into a native binary format via `gpg --dearmor`:
   ```bash
   curl -fsSL https://download.docker.com | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   ```
3. **Targeted Access Validation:** Managed the software source pointers by locking down the exact tracking file (`/etc/apt/sources.list.d/docker.list`) to scan against the hard-linked binary key:
   ```text
   deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu noble stable
   ```

#### Post-Deployment Verification Logs
* **Daemon Integrity Matrix:** Verification utility `sudo systemctl status docker` confirms background system status is active, green, and initialized.
* **Egress Plumbing Check:** Execution of `sudo docker run hello-world` successfully forces the container runtime layer to connect outbound through the pfSense perimeter gateway (`192.168.100.1`), pull down remote data blocks from Docker Hub, and parse an execution greeting locally.

### 3.5 Virtualized Application Layer Deployment (Nginx Workload)
To validate the end-to-end data plane integrity of the host-to-firewall network fabric, a standalone microservice workload has been provisioned.

#### Application Architecture & Port Mapping
* **Deployment Command:** `sudo docker run -d --name homelab-web -p 8080:80 nginx`
* **Port Translation Vector:** Forwards external requests hitting the Ubuntu host on port `8080` down into the containerized sandbox's native HTTP port `80`. This architecture bypasses local host conflicts while complying with perimeter web blocks on port 80.

#### Post-Remediation Verification Logs
* **Local Loopback Validation:** Running `curl -I http://localhost:8080` inside `ubuntu-srv` natively yields an immediate `HTTP/1.1 200 OK` header payload from the Nginx daemon.
* **Host-to-Guest Inter-Network Verification:** Navigating to `http://192.168.100.11:8080` from a dedicated host browser window successfully processes the graphical web application layer.
* **Browser HSTS Mitigation:** Local browser history aggressively forced unencrypted HTTP traffic to rewrite as HTTPS, throwing connection drops. Forcing connection tracking via an **Incognito Session** successfully isolated the transport layers and exposed the true application payload.

### 3.6 Enterprise Reverse Proxy Deployment (Nginx Proxy Manager)
To eliminate messy IP-and-port combinations (such as `192.168.100.11:8080`) and simulate real corporate application gateways, a containerized reverse proxy architecture has been deployed[cite: 1].

#### Application Architecture & Port Allocations
* **Container Orchestration:** Nginx Proxy Manager (NPM) deployed via Docker Compose on the Ubuntu Server (`192.168.100.11`)[cite: 1, 2].
* **Active Listening Ports:** 
  * `80/tcp` (Public HTTP Traffic)[cite: 2]
  * `443/tcp` (Public HTTPS Traffic)[cite: 2]
  * `81/tcp` (Administrative Control Panel UI)[cite: 2]

### 4.2 Internal Local DNS Resolver Integration (pfSense Host Overrides)
To translate human-readable enterprise domains to internal microservices without relying on public DNS infrastructure, the local network name resolution path was hardened:

#### Configuration Workflow
1. **pfSense DNS Resolver (Unbound):** Configured explicit **Host Overrides** mapping internal domain zones (`*.homelab.local`) directly to the Ubuntu server compute node (`192.168.100.11`)[cite: 1].
2. **Host-Only Bridge Routing:** Updated the Windows host `VMnet1` adapter configuration to enforce `192.168.100.1` as its primary Preferred DNS Server, establishing a stable local resolution loopback.

#### Post-Remediation Verification Logs
* **DNS Resolution Check:** Executing `nslookup nginx.homelab.local` successfully queries the pfSense nameserver (`pfsense.home.arpa` at `192.168.100.1`) and returns target address `192.168.100.11`[cite: 5].
* **Application Layer Routing:** Navigating to `http://nginx.homelab.local` seamlessly proxies traffic through Nginx Proxy Manager down to the underlying backend test container, achieving clean domain translation.

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

## 5. Current Implementation Status: Secure Edge Proxy & PKI

### Architecture & Components
* **Edge Reverse Proxy:** Deployed **Nginx Proxy Manager (NPM)** via Docker on Ubuntu Server (`192.168.100.11`) to manage ingress traffic, routing, and SSL termination.
* **Internal PKI & TLS:** Implemented custom local wildcard certificates (`*.homelab.local`) generated via `mkcert` and trusted at the OS level to simulate enterprise internal certificate authorities.
* **Upstream Testing:** Configured dynamic proxy hosts routing to containerized workloads with TLS enforcement (`Force SSL`, `HTTP/2`) and valid trust chains.

### Engineering & Troubleshooting Highlights
* **Cross-Platform Cryptographic Parsing:** Resolved strict OpenSSL backend decoder errors (`ossl_store_handle_load_result`) by ensuring precise UTF-8 PEM file structuring and handling cross-platform file transfer routines between Windows administrative hosts and Linux Docker volumes.
* **Edge Security:** Established secure HTTPS connections for local service discovery, avoiding insecure plaintext traffic within internal networks.

## 🔐 Identity & Access Management (IAM) — Keycloak Setup

### 📌 Overview
This section documents the deployment and configuration of an enterprise-grade Identity and Access Management (IAM) infrastructure using **Keycloak 26** backed by **PostgreSQL 16**, hosted on an **Ubuntu Server** container environment behind **Nginx Proxy Manager (NPM)** with wildcard TLS/PKI certificates (`*.homelab.local`).

The goal of this deployment is to establish a centralized Single Sign-On (SSO) authority enforcing **Role-Based Access Control (RBAC)** across all downstream self-hosted services and infrastructure management applications.

---

### 🏗 Network & Proxy Architecture

* **Host Node:** Ubuntu Server (`192.168.100.11`) running Docker Engine.
* **External Network:** Dedicated bridge network (`homelab-net`) ensuring container isolation and inter-service routing.
* **Reverse Proxy:** Nginx Proxy Manager terminating SSL (`https://auth.homelab.local`).
* **Header Forwarding:** Keycloak configured with `KC_PROXY_HEADERS: "xforwarded"` to properly evaluate reverse proxy HTTP headers (`X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Host`) supplied by NPM, eliminating proxy loopback errors.

---

### 🔐 Identity & RBAC Design

To adhere to the Principle of Least Privilege and maintain proper separation of concerns:
1. **Master Realm:** Reserved strictly for root-level identity provider administration.
2. **Homelab Realm:** Isolated production realm created for homelab users, groups, and downstream client application SSO integrations.

#### Role-Based Access Control (RBAC) Matrix

| Group Name | Realm Role | Description / Access Scope |
| :--- | :--- | :--- |
| **Administrators** | `homelab-admin` | Full administrative access across integrated homelab services. |
| **Standard Users** | `homelab-user` | Regular authenticated user privileges and standard service access. |
| **Auditors** | `homelab-readonly` | Read-only / view-only access across monitoring and auditing dashboards. |


[ User Account ] ──> ( Group Membership ) ──> [ Realm Role ] ──> [ OIDC JWT Claim ] ──> ( App Enforces Local RBAC )

---

### 🛠 Deployment & Configuration Steps

#### 1. Keycloak Core Privileges & Realm Provisioning
* Initialized system administrator (`sysadmin`) in the `master` realm.
* Assigned global administrative scope (`admin` and `create-realm` realm roles) to allow self-managed realm creation and tenant configuration.
* Provisioned the custom production realm: **`Homelab`**.

#### 2. RBAC Construction
* Created custom application-level realm roles:
  * `homelab-admin`
  * `homelab-user`
  * `homelab-readonly`
* Configured target user groups (`Administrators`, `Standard Users`, `Auditors`) and mapped their respective realm roles.

#### 3. Downstream OIDC Integration Model
Downstream client applications (e.g., Grafana, Portainer, Gitea) authenticate users via OpenID Connect (OIDC). Upon successful authentication, Keycloak issues a signed JSON Web Token (JWT) containing the user's mapped group/role claims. The client application inspects these claims to dynamically enforce local application permissions.

---

Homelab Infrastructure & OIDC Authentication

This repository tracks the configuration and setup of my self-hosted homelab services, featuring centralized Single Sign-On (SSO) via Keycloak and reverse proxy routing via Nginx Proxy Manager.

## 🏗️ Architecture & Containers

All services run inside isolated Docker containers connected via a shared external Docker network (`nginx-proxy-manager_default`).

* **Nginx Proxy Manager (`nginx-proxy-manager`)**: Acts as the reverse proxy managing local domains (`*.homelab.local`) and SSL certificates.
* **Keycloak (`keycloak`)**: Identity Provider (IdP) handling OpenID Connect (OIDC) authentication.
* **PostgreSQL (`keycloak-postgres`)**: Dedicated database backend for Keycloak.
* **Grafana (`grafana`)**: Visualization and monitoring platform integrated with Keycloak OIDC.

## 🔐 Authentication & OIDC Integration

* **Realm**: `Homelab`
* **Client ID**: `grafana`
* **Role Mapping**: 
  * `homelab-admin` $\rightarrow$ Grafana **Admin**
  * `homelab-user` $\rightarrow$ Grafana **Editor**
  * Default $\rightarrow$ **Viewer**
* **Network Communication**: Grafana communicates with Keycloak internally over the Docker network (`http://keycloak:8080`) to bypass proxy loops during token exchange, while external user traffic flows securely through Nginx Proxy Manager.

---

## 📊 Observability & Metrics Stack (Prometheus & Grafana)

To ensure proactive visibility into host health and container performance, a complete systems monitoring stack has been deployed.

### 🏗️ Monitoring Architecture & Core Components
* **Prometheus (`prometheus`):** Acts as the centralized time-series metrics engine. It periodically scrapes targets based on a defined configuration interval.
* **Node Exporter (`node-exporter`):** A lightweight hardware metrics collector running natively on the host system, exposing system CPU, RAM, disk usage, and network statistics on port `9100`.
* **Grafana (`grafana`):** The visualization presentation layer that ingests metrics from Prometheus and renders real-time performance dashboards (Node Exporter Full dashboard, ID `1860`).

### 🌐 Internal Container Networking & DNS Resolution
* **Bypassing the Reverse Proxy:** Container-to-container communication (e.g., Grafana fetching data from Prometheus via `http://prometheus:9090`) occurs entirely on a private, shared Docker network (`nginx-proxy-manager_default`).
* **Internal Docker DNS:** Docker automatically handles internal name resolution, routing traffic straight to target container IP addresses behind the scenes.
* **Role of Nginx Proxy Manager (NPM):** NPM is strictly reserved for **external incoming front-door traffic** and SSL/TLS certificate termination for browsers accessing services via subdomains (e.g., `grafana.homelab.local`). Internal API polling and backend telemetry bypass NPM entirely.