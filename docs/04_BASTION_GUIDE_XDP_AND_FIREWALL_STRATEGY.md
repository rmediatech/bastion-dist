# 🛡️ 04_BASTION_GUIDE_XDP_AND_FIREWALL_STRATEGY

> **Target Audience:** Systems Architects, Network Defense Engineers, eBPF Practitioners  
> **Topic:** Wire-Speed Packet Interception, Katran ARX SYN Cookies, and Anti-Spoofing  
> **Standard:** `AGY-RULE-SOVEREIGN-SECURITY-01` & `SOVEREIGN-RFC-BAREMETAL-01`  

---

## ⚡ 1. The Power of eBPF / XDP (eXpress Data Path)

Traditional Linux firewall tools (`iptables`, `nftables`) operate high up in the networking stack. Before a packet reaches their rules, the kernel must:
1. Allocate an `sk_buff` socket buffer structure in memory.
2. Initialize driver ring descriptors and parse L2/L3/L4 headers.
3. Traverse multiple netfilter table hooks (`PREROUTING`, `INPUT`).

Under a multi-gigabit TCP SYN flood or volumetric UDP attack, the CPU spends 100% of its cycles allocating and freeing `sk_buff` structs, causing total system lockup (SYN backlog exhaustion).

**Bastion's In-Kernel XDP Ingress Hook**:
* Intercepts packets directly at the network interface card (NIC) driver layer before the kernel allocates `sk_buff`.
* Evaluates rules in $<25\text{ns}$ per packet.
* Drops hostile packets via `XDP_DROP` or bounces SYN-ACK challenges via `XDP_TX` at line rate.

---

## 🍪 2. Katran-Style ARX Cryptographic SYN Cookie Reflector

To defeat distributed TCP SYN floods without consuming server memory, Bastion implements in-kernel ARX SYN Cookies:

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| MSS Index (2) |         ARX Authentication Tag (30)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Mathematical Architecture:
* **Bits 31..=30 (2-Bit MSS Table)**: Encodes 4 standard TCP Maximum Segment Sizes (`536, 1220, 1360, 1460` bytes), eliminating the out-of-bounds gap of legacy implementations.
* **Bits 29..=0 (30-Bit ARX Tag)**: Computes a 30-bit cryptographic authentication tag ($2^{30}$ brute-force resistance) using rotating dual-bucket 128-bit secrets (`SynCookieSecret`).
* **Line-Rate Reflection (`XDP_TX`)**: When per-source SYN velocity exceeds threshold, the eBPF program:
  1. Truncates TCP options in-place via `bpf_xdp_adjust_tail`.
  2. Swaps Ethernet MACs, IPv4 addresses, and TCP ports.
  3. Writes the calculated 32-bit cookie into `tcp.seq`.
  4. Recomputes RFC 1071 checksums via `bpf_csum_diff`.
  5. Bounces the SYN-ACK directly out the wire via `XDP_TX` with **zero memory allocations**.
* **Grace Verification**: Valid client ACKs matching the rotating secret grant a 60-second fast-pass grace window (`VERIFIED_GRACE_NS`).

---

## 🚫 3. Wire-Level Anti-Spoofing & Anomaly Defense

Before any packet reaches TCP evaluation, Bastion applies zero-branching bitwise hardware filters:

### A. Bogon, Martian & LAND Attack Defense
* **LAND Attack (`saddr == daddr`)**: Dropped instantly.
* **Bogon / Martian Source Addresses**: Dropped at wire speed (`0.0.0.0/8`, `127.0.0.0/8`, Multicast/Broadcast `≥ 224.0.0.0`).

### B. IPv4 Fragmentation Anomaly Neutralization
* **Contradictory `DF + MF` Flags**: Dropped.
* **RFC 1858 Tiny Initial Fragments**: Initial fragments with `offset == 0 && MF == 1` but total length $< 48$ bytes (hiding L4 port headers) are dropped immediately.
* **Teardrop Overlapping Offsets**: Non-zero fragment offsets with $0 < \text{offset} \times 8 < 24$ dropped.
* **Ping-of-Death Reassembly Overflow**: Fragments where $\text{offset} \times 8 + \text{payload} > 65535$ dropped.

### C. Volumetric UDP & ICMP Flood Limiting
* Enforces a sliding 1-second decay window (`VOLUMETRIC_WINDOW_NS = 1_000_000_000`).
* Throttles per-source UDP datagrams (`DEFAULT_UDP_THRESHOLD = 512 pkt/s`) and ICMP Echo Request Type 8 floods (`DEFAULT_ICMP_THRESHOLD = 16 pkt/s`) while permitting normal traffic and ICMP Echo Replies (Type 0).
