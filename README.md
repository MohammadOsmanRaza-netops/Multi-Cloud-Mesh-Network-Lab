# Multi-Cloud-Mesh-Network-Lab

# 🌐 Multi-Cloud Mesh Network Lab

> **On-Premise CML · AWS Cloud DC · GCP DR Site — connected via Tailscale WireGuard at ₹0/month**

**Built by Mohammad Osman Raza — Network & Cloud Engineer · Hyderabad, India**

---
## 🗺 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    TAILSCALE WIREGUARD MESH                 │
│              Encrypted · Free Forever · ₹0/month           │
└──────────────┬──────────────────┬───────────────────────────┘
               │                  │                    │
   ┌───────────▼──────┐  ┌────────▼───────┐  ┌────────▼───────┐
   │    CML ON-PREM   │  │  AWS CLOUD DC  │  │  GCP DR SITE   │
   │   192.168.1.0/24 |  |                |  |                |
   |   10.1.1.0/24    │  │  172.31.0.0/16 │  │ 10.128.0.0/20  │
   │  10.1.2.0/24     │  │  us-east-1     │  │  us-central1   │
   │                  │  │                │  │                │
   │  ubuntu-0        │  │  EC2 t2.micro  │  │  e2-micro VM   │
   │  (Tailscale GW)  │  │  nginx server  │  │  DR workload   │
   │  iol-0 (Router)  │  │                │  │                │
   │  iol-l2-0 (SW)   │  │                │  │                │
   │  ubuntu-1 (Host) │  │                │  │                │
   └──────────────────┘  └────────────────┘  └────────────────┘
```

---

## 📋 Project Summary

| Item | Detail |
|---|---|
| **Sites** | 3 — CML On-Prem, AWS, GCP |
| **Subnets** | CML 192.168.1.0/24 10.1.1.0/24 + 10.1.2.0/24 · AWS 172.31.0.0/16 · GCP 10.128.0.0/20 |
| **VPN** | Tailscale WireGuard — free personal plan |
| **AWS Region** | us-east-1 |
| **GCP Region** | us-central1 (always free) |
| **Total Cost** | ₹0 / month |
| **Remote Access** | Zero Trust via Tailscale |

---

## 🖥 CML Topology

```
ext-conn-0 (Proxmox NAT — internet)
     |
    ens2 (192.168.1.10/24)
     |
 ubuntu-0  ←── Tailscale Gateway
     |
    ens3 (10.1.1.1/24)
     |
   E0/0
     |
  iol-0  ←── Router
     |     10.1.1.2/24 (E0/0)
   E0/1   10.1.2.1/24 (E0/1)
     |
   E0/0
     |
 iol-l2-0  ←── Switch
     |
   E0/1
     |
    ens2
     |
 ubuntu-1  ←── End Host (10.1.2.2/24)
```

---

## 🔧 Device IP Reference

| Device | Interface | IP Address | Role |
|---|---|---|---|
| ubuntu-0 | ens2 | 192.168.1.10 DHCP | Internet / Tailscale |
| ubuntu-0 | ens3 | 10.1.1.1/24 | Link to CML Router |
| iol-0 | E0/0 | 10.1.1.2/24 | Link to ubuntu-0 |
| iol-0 | E0/1 | 10.1.2.1/24 | Link to Switch |
| ubuntu-1 | ens2 | 10.1.2.2/24 | End Host |
| AWS EC2 | eth0 | 172.31.0.0/16 | Cloud DC |
| GCP VM | ens4 | 10.128.0.0/20 | DR Site |

---

## ⚡ Tailscale Node Reference

| Node | Tailscale IP | Advertised Subnet |
|---|---|---|
| ubuntu-0 (CML Gateway) | 100.0.0.0 | 10.1.1.0/24, 10.1.2.0/24 |
| AWS EC2 | 100.0.0.0 | 172.31.0.0/16 |
| GCP VM | 100.0.0.0 | 10.128.0.0/20 |

---

## 🚀 How to Reproduce This Lab

### Step 1 — Tailscale Account
```bash
# Go to tailscale.com — sign up free
# Enable subnet routing in admin settings
```

### Step 2 — AWS EC2
```bash
# Launch t2.micro Ubuntu 22.04 in us-east-1
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Enable IP forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Advertise AWS subnet
sudo tailscale up --advertise-routes=172.31.0.0/16 --accept-routes
```

### Step 3 — GCP e2-micro
```bash
# Launch e2-micro Ubuntu 22.04 in us-central1
# Standard persistent disk 30GB — no backups — no Ops Agent
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Enable IP forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Advertise GCP subnet
sudo tailscale up --advertise-routes=10.128.0.0/20 --accept-routes
```

### Step 4 — CML Ubuntu Gateway
```bash
# Configure ens3 with static IP in netplan
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    ens2:
      dhcp4: true
    ens3:
      dhcp4: false
      addresses:
        - 10.1.1.1/24
      routes:
        - to: 10.1.2.0/24
          via: 10.1.1.2

# Enable IP forwarding
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Advertise CML subnets
sudo tailscale up --advertise-routes=10.1.1.0/24,10.1.2.0/24 --accept-routes
```

### Step 5 — CML Router Static Routes
```bash
# On iol-0
ip route 172.31.0.0 255.255.0.0 10.1.1.1
ip route 10.128.0.0 255.255.240.0 10.1.1.1
```

### Step 6 — Approve Routes in Tailscale Admin
```
tailscale.com/admin/machines
→ Click each node
→ Edit route settings
→ Approve all advertised subnets
```

---

## ✅ Connectivity Verification

```bash
# From ubuntu-0 — ping AWS
ping -c 4 172.31.x.x

# From ubuntu-0 — ping GCP
ping -c 4 10.128.x.x

# From ubuntu-1 — ping AWS (proves end-to-end routing)
ping -c 4 172.31.x.x

# From any Tailscale device — open browser
http://100.79.220.34
# Shows: Mohammad Osman Raza Lab private web server
```

---

## 💰 Cost Breakdown

| Resource | Free Tier | Cost |
|---|---|---|
| AWS EC2 t2.micro | 750 hrs/month (12 months) | ₹0 |
| AWS VPC + Subnet | Always free | ₹0 |
| GCP e2-micro | Always free forever | ₹0 |
| GCP Standard disk 30GB | Always free forever | ₹0 |
| Tailscale | Free personal plan forever | ₹0 |
| Cisco CML | Free license | ₹0 |
| Proxmox | Free open source | ₹0 |
| **Total** | | **₹0 / month** |

---

## 🛠 Tech Stack

![Cisco](https://img.shields.io/badge/Cisco_CML-1BA0D7?style=flat&logo=cisco&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-FF9900?style=flat&logo=amazonaws&logoColor=white)
![GCP](https://img.shields.io/badge/GCP-4285F4?style=flat&logo=googlecloud&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-WireGuard-6B7FD4?style=flat)
![Ubuntu](https://img.shields.io/badge/Ubuntu_22.04-E95420?style=flat&logo=ubuntu&logoColor=white)

---

## 📁 Repo Structure

```
hydtech-network-lab/
├── README.md
├── configs/
│   ├── ubuntu-0-netplan.yaml   ← gateway netplan config
│   ├── iol-0-running.txt       ← router config
│   └── tailscale-acl.json      ← Tailscale ACL policy
└── docs/
    └── architecture.png        ← topology diagram screenshot
```

---

## 👤 Author

**Mohammad Osman Raza**
Network & Cloud Engineer · Hyderabad, India
Available for Network Automation and Cloud Networking roles
