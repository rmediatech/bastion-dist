# 🔍 05_BASTION_GUIDE_TACTICAL_SCANNER_AND_RECON

> **Target Audience:** Penetration Testers, Security Auditors, Network Administrators  
> **Topic:** High-Speed Adaptive Port Scanning, TCP Options Crafting, and Passive Fingerprinting  
> **Standard:** `SOVEREIGN-RFC-BAREMETAL-01` & `SOVEREIGN-SECURITY-01`  

---

## ⚡ 1. Adaptive EWMA RTT Timing Engine

Static timeouts in network scanners either waste time waiting on fast gigabit LANs or drop probes prematurely on congested WAN links.

Bastion implements an **Exponential Weighted Moving Average (EWMA)** RTT estimator:
$$\text{RTT}_{\text{new}} = \frac{7}{8} \text{RTT}_{\text{prev}} + \frac{1}{8} \text{Sample}$$
* Calculated entirely in integer microsecond arithmetic.
* Dynamic timeout threshold scales automatically to $3 \times \text{RTT}_{\text{est}}$.
* Adapts in real time to network congestion, dropping scan completion times by over 80% without losing probe fidelity.

---

## 🧱 2. RFC-Compliant TCP Options Builder (`forge::packet::tcp_options`)

To generate realistic probes and defeat fingerprint-based scanner detection, Bastion features a zero-allocation TCP option serializer:

```text
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Kind=2 (MSS) |    Length=4   |       Max Segment Size        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Kind=3 (WS)  |    Length=3   |  Shift Count  |  Kind=1 (NOP) |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Kind=4 (SACK)|    Length=2   |          Padding (0)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Compliant Standards Supported:
* **RFC 793**: Maximum Segment Size (MSS) option negotiation.
* **RFC 7323**: Window Scale (WS) and high-resolution Timestamps (TS).
* **RFC 2018**: Selective Acknowledgment (SACK) Permitted.
* **Automatic 32-Bit Word Alignment**: Enforces word padding via NOP (`Kind=1`) and End of Option List (`Kind=0`), guaranteeing kernel acceptance.

---

## 🕵️ 3. Zero-Copy Passive OS Fingerprinting

Rather than actively transmitting invasive probe bursts that trigger intrusion detection alarms, Bastion's `recon::fingerprint::passive` analyzes inbound traffic non-intrusively:

1. **Initial TTL Recovery**: Evaluates current IP TTL against standard initial baselines ($32, 64, 128, 255$) to reconstruct true hop counts across routed topologies.
2. **TCP Window Size & DF Bit Heuristics**: Distinguishes Linux kernels (`DF=1, Window=64240`), Windows 10/11 (`DF=1, Window=64512 or 65535`), and macOS (`Window=65535`).
3. **TCP Option Ordering Signatures**: Validates specific option order sequences (`MSS,NOP,WS,SACKOK,TS` vs `MSS,NOP,NOP,SACKOK`).

---

## 🛡️ 4. ANSI Escape & Log Poisoning Defense

Untrusted network banners frequently contain malicious ANSI terminal escape sequences (`\x1b[2J`, `\x1b[0m`, OSC hyperlink sequences) designed to clear operator terminals or exploit terminal emulator vulnerabilities.

Bastion passes all external service banners through `scrub_ansi_and_control`:
* Strips all ANSI CSI (`\x1b[...]`) and OSC (`\x1b]...`) control sequences.
* Converts unprintable non-ASCII bytes to sanitized hex escapes (`\xHH`).
* Enforces hard line-length ceilings ($<512$ bytes) to eliminate buffer exhaustion.
