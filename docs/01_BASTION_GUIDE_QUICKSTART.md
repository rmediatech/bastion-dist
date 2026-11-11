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

Bastion's CLI interface communicates directly with the background daemon over local Unix Domain Sockets (`/tmp/bastion_master.sock`), offering full self-documenting help, 4 output formats, and 5 operational lenses.

### Step 1: Query Local Interface & Driver Status
Inspect active network hardware, Netlink controller bindings, and XDP packet hook status across your preferred lens:

```bash
# Query compact card view (Default)
bastion status -f tui -l 1

# Export structured JSON for upstream SIEM / Prometheus agents
bastion status -f json

# View waterfall latency & drop distribution chart
bastion status -f tui -l waterfall
```

---

### Step 2: Interactive Offline Help & Manual
Explore built-in deep-dive documentation without needing an active internet connection or socket:

```bash
# Topic index
bastion help

# Deep-dive into operational lenses
bastion help lenses

# Full UNIX manual page (renders through your active $PAGER)
bastion man
```

---

### Step 3: Atomic L3/L4 Mitigation (Block & Pardon)
Instantly push an adversarial IP or CIDR into in-kernel wire-speed packet drop:

```bash
# Atomic IP drop
bastion block --ip 198.51.100.42

# Pardon and unblock
bastion unblock --ip 198.51.100.42
```

The daemon updates entries in the in-kernel `IP_BLOCKLIST` eBPF map in $<15\mu\text{s}$, dropping incoming packets at wire speed before Linux allocates socket buffers.

---

### Step 4: Policy Presets & Reconnaissance
Fetch fixture-backed host policy recommendations and run bounded stateless recon:

```bash
# Fetch policy recommendations formatted for spreadsheet audit (Anti-Formula Injection protected)
bastion policy-preset -f csv

# Execute bounded stateless SYN recon scan
bastion recon-scan --target 198.51.100.1
```

---

## 📖 Related Documentation
* [02. Architecture & Netlink Engine](02_BASTION_GUIDE_ARCHITECTURE.md)
* [03. Dual-Radio Hardware Orchestration](03_BASTION_GUIDE_DUAL_RADIO_HARDWARE.md)
* [04. XDP Packet Shield & SYN Cookies](04_BASTION_GUIDE_XDP_AND_FIREWALL_STRATEGY.md)
* [05. Tactical Scanner & Passive Recon](05_BASTION_GUIDE_TACTICAL_SCANNER_AND_RECON.md)
* [06. Crucible Empirical Security Audit](06_BASTION_GUIDE_CRUCIBLE_EMPIRICAL_AUDIT.md)
