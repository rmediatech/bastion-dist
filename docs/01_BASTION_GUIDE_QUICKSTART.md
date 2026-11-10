# 🚀 01_BASTION_GUIDE_QUICKSTART

> **Target Audience:** Systems Engineers, Network Security Operators, Infrastructure Architects  
> **Topic:** 5-Minute Installation, Standalone Verification, and Initial Network Defense  
> **Standard:** `SOVEREIGN-RFC-BAREMETAL-01` & `SOVEREIGN-SIGN-01`  

---

## ⚡ 1. One-Click Stream Installation

Bastion is distributed as a single, self-contained standalone executable statically linked against Musl libc. It requires zero runtime dependencies, package managers, or shared libraries.

### A. 1-Click Stream Install (Linux)
```bash
curl -sSL https://rmediatech.com/install/bastion | sh
```

### B. Direct Standalone Binary Download
```bash
# Download verified Linux x86_64 binary
curl -sSL https://rmediatech.com/bin/bastion -o bastion
chmod +x bastion
sudo mv bastion /usr/local/bin/

# Verify binary execution & version
bastion --version
# Output: bastion 1.4.0
```

---

## 🔒 2. Cryptographic Provenance Verification (`SOVEREIGN-SIGN-01`)

Every release binary of Bastion is cryptographically sealed with the official RMediaTech Sovereign Release Signing Key:
* **Key ID**: `4E428019A109578B`
* **Signer**: `Rick Walker (Sovereign Release Signing Key) <rick@rmediatech.com>`

```bash
# 1. Import Public Release Key
curl -sSL https://rmediatech.com/SOVEREIGN_RELEASE_KEY.asc | gpg --import

# 2. Download Detached OpenPGP Signature
curl -sSL https://rmediatech.com/signatures/bastion -o bastion.asc

# 3. Verify Cryptographic Authenticity
gpg --verify bastion.asc /usr/local/bin/bastion
```

**Expected Real-World Output:**
```text
gpg: Signature made Tue 10 Nov 2026 09:54:00 AM EST
gpg:                using EDDSA key 6BC8DA5E7F1B8B49BE5CD70A4E428019A109578B
gpg: Good signature from "Rick Walker (Sovereign Release Signing Key) <rick@rmediatech.com>" [ultimate]
```

---

## 🧭 3. Fast Operational Walkthrough

Bastion's CLI interface communicates directly with the background daemon over local Unix Domain Sockets (`/run/bastion/bastion.sock`) or executes standalone network discovery.

### Step 1: Query Local Interface & Driver Status
Inspect active network hardware, Netlink controller bindings, and XDP packet hook status:

```bash
bastion status --json
```

**Expected Real-World Output:**
```json
{
  "version": "1.4.0",
  "kernel_netlink": "nl80211_active",
  "xdp_hook": "XDP_DRV",
  "active_interfaces": [
    {
      "name": "eth0",
      "ifindex": 2,
      "state": "UP",
      "promisc": false
    },
    {
      "name": "wlan1",
      "ifindex": 3,
      "state": "UP",
      "mode": "SNIFFER"
    }
  ],
  "quarantine_count": 0,
  "syn_cookies_active": true
}
```

---

### Step 2: High-Speed Subnet Reconnaissance (`sweep`)
Execute an asynchronous local subnet sweep across RFC 5737 / RFC 1918 address space:

```bash
bastion sweep --target 192.0.2.0/24
```

**Expected Real-World Output:**
```text
[*] Initializing Subnet Sweep across 192.0.2.0/24...
================================================================================
  BASTION LAYER-2/LAYER-3 ACTIVE DISCOVERY
================================================================================
  IP ADDRESS       MAC ADDRESS        RTT      STATUS    IDENTIFIER
--------------------------------------------------------------------------------
  192.0.2.1        52:54:00:12:34:01  0.24ms   ACTIVE    Carrier Gateway AP
  192.0.2.10       52:54:00:12:34:AA  0.08ms   ACTIVE    ★ THIS HOST (bastion-node)
  192.0.2.20       52:54:00:12:34:20  1.12ms   ACTIVE    Workstation Node
  192.0.2.30       52:54:00:12:34:30  0.89ms   ACTIVE    Media Receiver
================================================================================
Discovered 4 active stations in 184ms. Bitwise Radix Trie synchronized.
```

---

### Step 3: Inspect Active Firewall & XDP Quarantine Policy
Preview active CIDR quarantine rules and in-kernel BPF map entries:

```bash
bastion policy preview
```

**Expected Real-World Output:**
```text
================================================================================
  BASTION KERNEL XDP POLICY & QUARANTINE LEDGER
================================================================================
  INGRESS HOOK:      XDP (eBPF Native Wire Filter)
  SYN COOKIE MODE:   KATRAN ARX 30-BIT (ROTATING DUAL BUCKET)
  VOLUMETRIC LIMITS: UDP 512 pkt/s | ICMP Type-8 16 pkt/s
  BOGON / LAND:      ENFORCED (Sub-nanosecond hardware drop)
--------------------------------------------------------------------------------
  ACTIVE QUARANTINES: 0 rules active
================================================================================
```

---

### Step 4: Add Dynamic CIDR Quarantine
Instantly push an adversarial subnet into in-kernel wire-speed packet drop:

```bash
sudo bastion policy block --cidr 198.51.100.0/24 --reason "Repeated SYN flood attempt"
```

The daemon expands the CIDR block within bounded limits ($\le /22$) and inserts entries into the in-kernel `IP_BLOCKLIST` map in $<15\mu\text{s}$, dropping incoming packets at wire speed before Linux allocates socket buffers.

---

## 📖 Related Documentation
* [02. Architecture & Netlink Engine](02_BASTION_GUIDE_ARCHITECTURE.md)
* [03. Dual-Radio Hardware Orchestration](03_BASTION_GUIDE_DUAL_RADIO_HARDWARE.md)
* [04. XDP Packet Shield & SYN Cookies](04_BASTION_GUIDE_XDP_AND_FIREWALL_STRATEGY.md)
* [05. Tactical Scanner & Passive Recon](05_BASTION_GUIDE_TACTICAL_SCANNER_AND_RECON.md)
* [06. Crucible Empirical Security Audit](06_BASTION_GUIDE_CRUCIBLE_EMPIRICAL_AUDIT.md)
