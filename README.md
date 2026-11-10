# 🛡️ Bastion (`bastion-dist`) — Official Distribution & Verification Hub

> **Microsecond Bitwise L3/L4/L7 Firewall, In-Kernel XDP Packet Shield & Bare-Metal nl80211 Netlink Wireless Controller**  
> *Zero cloud dependencies • Zero unvetted subprocesses • Pure Musl static standalone binary • 100% Adversarial TDD Tested*

[![Release](https://img.shields.io/badge/Release-v1.4.0-brightgreen.svg)](https://github.com/rmediatech/bastion-dist/releases/tag/v1.4.0)
[![Downloads](https://img.shields.io/badge/Downloads-rmediatech.com%2Fdownloads-blue.svg)](https://rmediatech.com/downloads)
[![TDD Tests](https://img.shields.io/badge/TDD%20Tests-360%2B%20Passed%20(100%25)-brightgreen.svg)](#)
[![Platform](https://img.shields.io/badge/Platform-Linux%20(Musl%20Static)-blue.svg)](#)
[![Arch](https://img.shields.io/badge/Arch-x86__64%20%7C%20aarch64-blueviolet.svg)](#)
[![Provenance](https://img.shields.io/badge/Provenance-GPG%20Signed%20(SOVEREIGN--SIGN--01)-blueviolet.svg)](SOVEREIGN_RELEASE_KEY.asc)
[![Security Standard](https://img.shields.io/badge/security-AGY--RULE--SOVEREIGN-green.svg)](https://rmediatech.com)
[![License: MIT OR Apache-2.0](https://img.shields.io/badge/license-MIT%2FApache--2.0-blue.svg)](LICENSE)

---

## 🏛️ Overview

**Bastion** is an enterprise-grade, zero-cloud Layer-3/4/7 firewall and bare-metal wireless controller written in 100% pure safe Rust. Engineered for complete operational sovereignty, Bastion delivers microsecond packet filtering, anti-spoofing defense, and dual-radio RF monitoring without relying on third-party cloud telemetry, external orchestration frameworks, or legacy shell wrappers (`ip`, `iw`, `nmcli`, `rfkill`).

### Core Capabilities
* **Bare-Metal Linux Netlink Driver (`nl80211` / `rtnetlink`)**: Communicates directly with the Linux kernel via RFC-compliant Generic Netlink sockets (`AF_NETLINK`) and standard `libc` ioctls (`SIOCGIFFLAGS`, `SIOCSIFFLAGS`, `SIOCGIFINDEX`), eliminating subprocess execution overhead.
* **Dual-Radio Hardware Auto-Pairing**: Automatically scans the physical USB bus topology via `/sys/class/net`, discovers promiscuous-capable wireless chipsets, and auto-pairs dual adapters into coordinated `[1] SNIFFER` and `[2] INJECTOR` roles.
* **Microsecond Bitwise Radix Trie & XDP Shield**: Intercepts TCP SYN floods, fragmentation anomalies, and spoofed packets at wire speed with sub-microsecond latency.
* **Dynamic CIDR Quarantine**: Evaluates incoming flows against multi-tiered IP allowlists and CIDR quarantine blocks.
* **Single Standalone Static Binary**: Statically linked against Musl libc with zero dynamic C/FFI dependencies. Starts in `<10ms` with $<15\text{MB}$ memory footprint.

---

## 📦 Binary Verification & Download

Verified standalone binaries are distributed directly through the **[RMediaTech Sovereign Binary Vault](https://rmediatech.com/downloads)**:

| Platform | Architecture | Checksum (SHA-256) | Vault Download |
| :--- | :--- | :--- | :--- |
| **Linux (Static Musl)** | `x86_64` | `df0e4a4e6dd2396e39ae23cdca6ddbb387311d9d80b5dd552fb14e53b9f0b90d` | [Download (2.0 MB)](https://rmediatech.com/bin/bastion) |

### 1-Click Stream Download & Installation
```bash
# 1-Click Stream Install
curl -sSL https://rmediatech.com/install/bastion | sh

# Or Direct Binary Download
curl -sSL https://rmediatech.com/bin/bastion -o bastion
chmod +x bastion
```

### Cryptographic Provenance Verification (`SOVEREIGN-SIGN-01`)
All releases are cryptographically sealed by `Rick Walker (Sovereign Release Signing Key) <rick@rmediatech.com>` (Key ID: `4E428019A109578B`):

```bash
# Import Public Key
curl -sSL https://rmediatech.com/SOVEREIGN_RELEASE_KEY.asc | gpg --import

# Verify Detached OpenPGP Signature
curl -sSL https://rmediatech.com/signatures/bastion -o bastion.asc
gpg --verify bastion.asc bastion
```

---

## 🚀 Quickstart

```bash
# View active daemon status and interface telemetry
bastion status --json

# Run local subnet discovery sweep
bastion sweep --target 192.0.2.0/24

# Preview active quarantine rules
bastion policy preview
```

---

## 🛡️ License & Provenance

Dual-licensed under MIT OR Apache-2.0. Cryptographically sealed by Rick Walker (`4E428019A109578B`).
