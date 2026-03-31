# FLEETS.md — Infrastructure Fleet Specifications

**Purpose:** Complete inventory of compute nodes, machines, and hardware in the infrastructure.

**Last Updated:** 2026-03-24

---

## COMPUTE NODES

### Node6
**MacBook Pro 16-inch (MacBookPro18,2)**

**Role:** Mobile Heavy Node / Dev Machine / High-End Execution Layer

**Specifications:**
- **Chip:** Apple M1 Max
- **CPU:** 10-core (8 Performance + 2 Efficiency)
- **GPU:** 24-core Apple GPU — Metal 4
- **ANE:** Apple Neural Engine (on-chip)
- **RAM:** 32GB LPDDR5 (Samsung) — unified memory
- **Storage:** ~1TB Apple SSD AP1024R (NVMe, APFS) — ~584GB free
- **Display:** Built-in Liquid Retina XDR, 3456×2234
- **OS:** macOS 26.3.1 (Build 25D2128)
- **Firmware:** 13822.81.10
- **Network:** Wi-Fi, Thunderbolt ×3, 3× Ethernet adapters

**Capabilities:**
- Apple M1 Max — unified memory architecture (CPU+GPU share 32GB)
- 24-core GPU + Metal 4 — best GPU in the fleet for ML inference
- Apple Neural Engine — on-chip ML acceleration
- Mobile high-performance compute
- Development workstation
- High-bandwidth I/O (3x Thunderbolt, 3x Ethernet)

**Use Cases:**
- On-the-go development
- High-performance agent execution
- ML/AI compute tasks (best local inference in fleet)
- Docker/container workloads
- Mobile orchestration node

---

### Node2 ⭐
**MacBook Pro 16-inch, 2019 (MacBookPro16,1)**

**Name:** Owner's MacBook Pro
**Serial:** SERIAL_REDACTED

**Role:** Primary Heavy Node / Main AI Brain / Core Compute Server

**Specifications:**
- **CPU:** Intel Core i9-9980HK @ 2.4GHz base — 8 cores / 16 threads (Hyper-Threading)
  - L2: 256KB per core | L3: 16MB shared | 64-bit Intel
- **RAM:** 64GB DDR4 @ 2667MHz — dual-channel (2x 32GB Micron, soldered, not upgradeable)
- **GPU:** Dual GPU (auto-switching)
  - Intel UHD Graphics 630 — 1536MB dynamic VRAM, Metal 3
  - AMD Radeon Pro 5500M — 4GB GDDR6, PCIe x16, Metal 3
- **Display:** 16-inch Retina LCD, 3072×1920, 24-bit color
- **Storage:**
  - Internal: Apple NVMe SSD 1TB (AP1024N, PCIe) — S.M.A.R.T. Verified
    - Mac HD: 692GB APFS — 264.53GB free
    - BOOTCAMP: 308.24GB NTFS — 254.8GB free (Windows partition)
  - External: 2TB One Touch HDD (USB) — 2TB free (Infrastructure drive)
- **OS:** macOS 26.3.1 (Build 25D2128), Darwin kernel 25.3.0
- **Security:** SIP enabled, Secure Virtual Memory, Activation Lock
- **Firmware:** 2094.80.5.0.0 | iBridge: 23.16.13120.0.0
- **Boot Mode:** Normal
- **Uptime:** 17+ days observed

**Capabilities:**
- Maximum RAM (64GB) — primary heavy compute node
- 8-core Intel i9 with 16 threads
- Dedicated AMD GPU (4GB GDDR6) for ML/graphics workloads
- Dual GPU switching — integrated for efficiency, dedicated for heavy tasks
- NVMe SSD — fast local storage
- Windows via Boot Camp partition
- Main agent runtime environment

**Use Cases:**
- Primary AI/ML workload server
- Main Ollama inference host
- Multi-agent orchestration hub
- API server hosting
- Telegram bot runtime
- Heavy background processing

**Current Services:**
- Ollama (tmux: ollama)
- API Server (port 5001)
- Telegram Bot (tmux: agent)
- Agent workspace

---

### Node3
**MacBook Pro Mid-2015 (MacBookPro11,5)**

**Role:** Security Node / Utility Node / Storage-Support Machine

**Specifications:**
- **CPU:** Intel Core i7 @ 2.8GHz — 4 cores / 8 threads (Hyper-Threading)
  - L2: 256KB per core | L3: 6MB shared
- **RAM:** 16GB DDR3 @ 1600MHz — 2x 8GB (non-upgradeable)
- **Storage:** 500GB Apple SSD SM0512G (PCIe, APFS) — ~358.5GB free
- **GPU:** Dual GPU (auto-switching)
  - Intel Iris Pro — 1536MB dynamic VRAM (integrated)
  - AMD Radeon R9 M370X — 2GB VRAM (discrete)
- **OS:** macOS 12.7.6 Monterey | Darwin kernel 21.6.0
- **Network:**
  - USB Ethernet (AX88179A): 1000baseT — IP 192.168.1.30
  - Wi-Fi (AirPort): en0
- **Uptime:** 24 days observed

**Capabilities:**
- Stable macOS Monterey (max supported OS for this hardware)
- Discrete GPU (AMD R9 M370X 2GB) for light GPU workloads
- Dual GPU auto-switching
- Moderate compute — 4-core Intel i7
- Large free storage (~358GB)
- Wired Ethernet via USB adapter

**Use Cases:**
- Network security monitoring
- Log aggregation node
- Backup coordination
- Utility services host
- Legacy macOS compatibility layer (Monterey — useful for older app support)

---

### Node4
**Lenovo ThinkPad (node4-hostname)**

**Role:** Flexible Worker / Lightweight Agent Box / Deploy-Anywhere Utility Machine

**Specifications:**
- **CPU:** Intel Core i7-6600U @ 2.60GHz (turbo 3.4GHz) — 2 cores / 4 threads (Skylake)
- **RAM:** 16GB total — 3GB used, 12GB available, 187GB swap (disk)
- **Storage:** Root `/dev/sda2`: 49GB total, 21GB used, 27GB free | Boot `/dev/sda1`: 511MB
- **GPU:** Intel HD Graphics 520 (integrated, Skylake)
- **OS:** Arch Linux (rolling release), kernel 6.12.74-1-lts
- **Hostname:** node4-hostname
- **Network:**
  - Ethernet `enp0s31f6`: UP — 192.168.1.40 (LAN), IPv6 active
  - WiFi `wlp4s0`: DOWN
  - Tailscale `tailscale0`: UP — 100.64.0.1
  - Docker bridge: DOWN

**Capabilities:**
- Low-resource footprint
- Arch Linux (cutting-edge packages, AUR access)
- Rolling release updates
- Tailscale mesh VPN node (always accessible remotely)
- Edge compute node
- Lightweight container host (Docker installed, bridge currently down)
- Minimalist base system
- Long uptime stability (23+ days observed)

**Use Cases:**
- Remote agent deployment
- Edge AI inference
- SSH jump host (reachable via Tailscale at 100.64.0.1)
- Lightweight automation tasks
- Bleeding-edge software testing
- Pacman package management

**Notes:**
- RAM upgraded to 16GB (previously listed as 12GB — corrected 2026-03-27)
- 27GB free on root — limited but workable for stateless agents
- Tailscale is active — reachable from anywhere on the mesh
- WiFi down — Ethernet only
- Load average very low (0.32) — good for background tasks

---

### Node5
**Sony Vaio Laptop**

**Role:** Storage Node / Light Automation Box / Backup and Low-Intensity Utility Node

**Specifications:**
- **CPU:** Intel Core i7-3537U @ 2.00GHz (Ivy Bridge, 3rd gen)
- **RAM:** 12GB
- **Storage:** 1TB HDD (root partition: 915GB, 859GB free)
- **OS:** Xubuntu 20.04.6 LTS (Ubuntu-based with XFCE desktop)
- **Kernel:** 5.15.0-139-generic
- **Network:** WiFi, Ethernet, USB

**Capabilities:**
- Large HDD storage (1TB)
- Moderate RAM (12GB)
- Low-power operation
- Backup/archive host
- Legacy hardware support
- Xubuntu (lightweight XFCE desktop)
- Debian/Ubuntu package ecosystem

**Use Cases:**
- File storage and archival
- Backup coordination node
- Low-intensity automation tasks
- Samba/NFS file server
- Media/document repository
- Scheduled batch processing (cron jobs)
- Rsync backup host
- Long-running background services

**Notes:**
- Older CPU (2012-era Ivy Bridge)
- HDD (not SSD) - slower I/O, good for bulk storage
- Good for low-priority background tasks
- Xubuntu = lightweight, good for older hardware
- APT package manager (stable, LTS-based)

---

## MOBILE DEVICES

### iPhone 16
**Apple iPhone 16**

**Role:** Primary Mobile Device / Edge Agent Platform / On-the-Go Interface

**Specifications:**
- **CPU:** Apple A18 Bionic
- **RAM:** ~8GB (estimated)
- **Storage:** 128GB
- **OS:** iOS 18+
- **Network:** 5G, WiFi 6E, Bluetooth 5.3
- **Sensors:** GPS, Accelerometer, Gyroscope, Proximity, etc.

**Capabilities:**
- Latest iOS features
- 5G connectivity
- Advanced camera system
- On-device ML inference
- Telegram client access
- Shortcuts automation
- SSH client (via apps)

**Use Cases:**
- Primary Telegram bot interface
- Mobile agent interaction
- Remote monitoring and alerts
- Voice command interface
- Photo/video capture for vision tasks
- Location-based triggers
- Push notification endpoint
- Emergency system control

**Notes:**
- Latest hardware (2024 release)
- Good battery life
- Best for interactive agent tasks

---

### iPhone 11 Pro
**Apple iPhone 11 Pro**

**Role:** Secondary Mobile Device / Backup Interface / Dedicated Test Device

**Specifications:**
- **CPU:** Apple A13 Bionic
- **RAM:** 4GB
- **Storage:** 64GB
- **OS:** iOS 15-17 (max supported)
- **Network:** 4G LTE, WiFi 6, Bluetooth 5.0
- **Sensors:** GPS, Accelerometer, Gyroscope, Proximity, etc.

**Capabilities:**
- Stable older iOS version
- Good camera system
- On-device ML (limited)
- SSH client access
- Shortcuts automation
- Telegram client

**Use Cases:**
- Backup Telegram interface
- Dedicated testing device
- Legacy iOS compatibility testing
- Secondary notification endpoint
- Dedicated automation device
- Development testing platform
- Fixed-location sensor node

**Notes:**
- Older hardware (2019 release)
- Limited to iOS 17 max
- 64GB storage constraint
- Good for dedicated single-purpose use

---

## STORAGE HARDWARE

### Storage1 🏢
**Seagate One Touch 2TB External Drive**

**Role:** Infrastructure Shared Storage / Backup Repository / Data Exchange Hub

**Specifications:**
- **Capacity:** 2TB (1.8TB usable)
- **Connection:** USB 3.0/3.1
- **File System:** APFS (Case-sensitive)
- **Mount Point:** `/Volumes/Infrastructure`
- **Device:** /dev/disk2 (disk3 container)
- **Status:** ✅ Wiped and Ready (2026-03-24)

**Capabilities:**
- 2TB shared storage pool
- Fast USB 3.0 transfer speeds
- APFS snapshots and cloning
- Cross-machine data exchange
- Portable storage node

**Use Cases:**
- Agent workspace storage
- Large dataset repository
- Model storage (LLMs, embeddings)
- Docker volume backing
- Inter-machine file transfer
- Backup staging area
- Log archival
- Database storage

**Configuration:**
- **Owner:** user
- **Permissions:** Full read/write access
- **No ACL restrictions** (wiped clean)
- **Encryption:** Not enabled (can be added)

**Performance:**
- **Read Speed:** ~400-500 MB/s (USB 3.0)
- **Write Speed:** ~300-400 MB/s (USB 3.0)

---

## NETWORK TOPOLOGY

```
┌─────────────────────────────────────────────────────────────────┐
│                        Home Network                             │
│                                                                 │
│  Desktop/Laptop Nodes:                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │
│  │  Node6   │  │  Node2 ⭐│  │  Node3   │  (macOS)            │
│  │  M1 Max  │  │ Intel i9 │  │ Intel    │                     │
│  │  32GB    │  │  64GB    │  │  16GB    │                     │
│  └──────────┘  └──────────┘  └──────────┘                     │
│                                                                 │
│  ┌──────────┐  ┌──────────┐                                    │
│  │  Node4   │  │  Node5   │  (Linux)                          │
│  │ Arch 12GB│  │Xubuntu12G│                                    │
│  └──────────┘  └──────────┘                                    │
│                                                                 │
│  Mobile Devices:                                                │
│  📱 iPhone 16 (128GB) - Primary mobile interface               │
│  📱 iPhone 11 Pro (64GB) - Backup/test device                  │
│                                                                 │
│  Storage:                                                       │
│  🏢 Storage1 (2TB USB) - Portable, connects to any node        │
└─────────────────────────────────────────────────────────────────┘
```

---

## FLEET SUMMARY

### Compute Nodes

| Machine      | CPU          | RAM   | Storage    | Role                  | Status      |
|--------------|--------------|-------|------------|-----------------------|-------------|
| Node6        | M1 Max       | 32GB  | 1TB NVMe (584GB free) | Mobile Heavy Node | Active |
| Node2 ⭐     | i9-9980HK    | 64GB  | 1TB NVMe + 2TB USB | Primary Compute | Active      |
| Node3        | i7 @ 2.8GHz  | 16GB  | 500GB SSD (358GB free) | Security/Utility | Active |
| Node4        | i7-6600U     | 16GB  | 49GB       | Lightweight Worker    | Active      |
| Node5        | i7-3537U     | 12GB  | 1TB HDD    | Storage/Backup        | Active      |

### Mobile Devices

| Device         | CPU          | RAM   | Storage    | Role                  | Status      |
|----------------|--------------|-------|------------|-----------------------|-------------|
| iPhone 16      | A18 Bionic   | ~8GB  | 128GB      | Primary Mobile        | Active      |
| iPhone 11 Pro  | A13 Bionic   | 4GB   | 64GB       | Backup/Test           | Active      |

### Storage Hardware

| Device       | Type         | —     | Capacity   | Role                  | Status      |
|--------------|--------------|-------|------------|-----------------------|-------------|
| **Storage1** | (USB Drive)  | —     | **2TB**    | **Shared Storage**    | **Ready**   |

**Total Fleet Resources:**
- **Desktop/Laptop RAM:** 32GB + 64GB + 16GB + 16GB + 12GB = **140GB**
- **Mobile RAM:** ~8GB + 4GB = **~12GB**
- **Total RAM:** **~152GB**
- **Desktop/Laptop SSD:** 1TB + 1TB + 500GB + 50GB = **2.55TB**
- **Desktop/Laptop HDD:** 1TB
- **Mobile Storage:** 128GB + 64GB = **192GB**
- **Shared Storage:** 2TB (Storage1)
- **Total Compute Nodes:** 5 desktop/laptop + 2 mobile = **7 nodes**

---

## FLEET ROLES & CAPABILITIES

### Heavy Compute Tier
- **Node2 (64GB)** - Primary orchestrator, main AI brain
- **Node6 (32GB)** - Mobile heavy compute, dev workstation

### Medium Compute Tier
- **Node3 (16GB)** - Security, utilities, storage support
- **Node4 (12GB)** - Flexible worker, edge deployment
- **Node5 (12GB)** - Storage node, light automation

### Storage Tier
- **Storage1 (2TB)** - Shared infrastructure storage
- **Node5 (1TB HDD)** - Secondary bulk storage
- **Individual SSDs** - Local fast storage on each node

---

## DEPLOYMENT RECOMMENDATIONS

### Agent Orchestration
- **Main orchestrator:** Node2 (64GB RAM, current machine)
- **Backup orchestrator:** Node6 (32GB RAM, mobile)

### Specialized Agents
- **Scraper agents:** Node4, Node3 (sandboxed, limited resources)
- **Research agents:** Node3, Node5 (moderate compute)
- **Storage agents:** Node5, Storage1 (large storage pool)

### Infrastructure Services
- **Ollama Server:** Node2 (main), Node6 (backup)
- **API Server:** Node2 (primary)
- **Telegram Bot:** Node2 (runtime)
- **OpenClaw Gateway:** Node2 (orchestrator)
- **Log Aggregation:** Node3
- **Backup Services:** Node5, Storage1

### Mobile Integration
- **Primary User Interface:** iPhone 16 (Telegram client, voice commands)
- **Backup Interface:** iPhone 11 Pro (secondary Telegram, testing)
- **Push Notifications:** Both iPhones (alerts, monitoring)
- **Remote Control:** iPhone 16 (SSH client, system control)
- **Voice Input:** iPhone 16 (Whisper transcription, voice commands)
- **Mobile Automation:** Both iPhones (Shortcuts, location triggers)
- **Testing Platform:** iPhone 11 Pro (dedicated test environment)

### Storage Strategy
- **Fast workspace:** Local SSDs (Node2, Node6, Node3)
- **Model storage:** Storage1 (2TB shared)
- **Bulk archival:** Node5 (1TB HDD)
- **Temporary data:** Local SSDs
- **Cross-machine exchange:** Storage1 (portable)

---

## FUTURE EXPANSION

### Planned Additions
- [ ] Network-attached storage (NAS) device
- [ ] Dedicated GPU server for heavy ML workloads
- [ ] Raspberry Pi cluster for edge compute
- [ ] Additional ThinkPad/cheap laptop as worker nodes
- [ ] iPad for mobile agent development and testing

### Potential Upgrades
- [ ] Node6: Expand to M1 Max with more RAM (if possible)
- [ ] Node2: Consider external GPU (eGPU) for ML acceleration
- [ ] Node3: Upgrade to newer macOS (if compatible)
- [ ] Node4: Expand storage with external SSD
- [ ] Node5: Upgrade to SSD for better performance
- [ ] iPhone 11 Pro: Upgrade iOS to latest supported version
- [ ] Consider iPhone SE as dedicated automation device

### Infrastructure Improvements
- [ ] VPN mesh network for secure inter-machine communication
- [ ] Docker Swarm or Kubernetes cluster across fleet
- [ ] Centralized logging and monitoring (Prometheus/Grafana)
- [ ] Automated backup rotation across Storage1 and Node5
- [ ] OpenClaw multi-agent deployment across fleet
- [ ] iOS Shortcuts automation for mobile agent triggers
- [ ] Mobile SSH client setup with dedicated keys
- [ ] iCloud integration for cross-device file sync

---

## NOTES

- **Current Machine:** Node2 (where Claude Code is running)
- **Primary User:** user (on Node2, Node6, Node3)
- **SSH Access:** Dedicated OpenClaw SSH key: `~/.ssh/agent_key`
- **Network:** All machines on same home network (WiFi/Ethernet/5G)
- **Storage Naming:** "Storage1" = Infrastructure shared storage (2TB USB)
- **Mobile Devices:** iPhone 16 (primary), iPhone 11 Pro (backup/test)
- **Primary Interface:** iPhone 16 via Telegram bot
- **Fleet Total:** 5 desktop/laptop nodes + 2 mobile devices + 1 shared storage = 8 hardware assets

---

## MAINTENANCE LOG

### 2026-03-27
- ✅ Node2 full specs confirmed: i9-9980HK, 64GB DDR4, AMD Radeon Pro 5500M 4GB, macOS 26.3.1
- ✅ Node4 (node4-hostname) full specs confirmed: i7-6600U, 16GB RAM, 27GB free, Tailscale active
- ✅ Node6 full specs confirmed: M1 Max, 10-core CPU, 24-core GPU Metal 4, 32GB LPDDR5, macOS 26.3.1
- ✅ Node3 full specs confirmed: i7 @ 2.8GHz quad-core, 16GB DDR3, AMD R9 M370X 2GB, macOS Monterey 12.7.6, IP 192.168.1.30

### 2026-03-26
- ✅ Node5 root partition expanded to 915GB (GParted live USB — previously 504GB)
- ✅ 859GB free on `/dev/sda5`

### 2026-03-24
- ✅ Storage1 (2TB One Touch) wiped and formatted as APFS
- ✅ Read/write tests passed
- ✅ Mounted at `/Volumes/Infrastructure`
- ✅ Ready for infrastructure use
