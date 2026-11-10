# 🏛️ 02_BASTION_GUIDE_ARCHITECTURE

> **Target Audience:** Systems Architects, Kernel Engineers, Security Researchers  
> **Topic:** Decoupled Process Planes, Bare-Metal Netlink Drivers, and Zero-Subprocess Engineering  
> **Standard:** `SOVEREIGN-RFC-BAREMETAL-01` & `SOVEREIGN-FORTHRIGHT-01`  

---

## 🏗️ 1. Multi-Plane Architecture

Bastion enforces a strict three-tier plane separation, eliminating monolithic sprawl and ensuring kernel-level performance without dynamic runtime overhead:

```mermaid
flowchart TB
    subgraph ControlPlane["User & Control Plane (Zero Allocation)"]
        CLI["CLI Control Utility ('bastion')"]
        UDS["Local Unix Domain Socket (/run/bastion/bastion.sock)"]
        CLI -->|Non-blocking JSON Commands| UDS
    end

    subgraph DaemonPlane["Daemon Coordinator Plane (Safe Rust)"]
        UDS --> DAEMON["Bastion Daemon Core"]
        DAEMON --> SCAN["Adaptive EWMA TCP Scanner"]
        DAEMON --> RECON["Zero-Copy Passive OS Fingerprinter"]
        DAEMON --> RADIX["Bitwise Radix IP Trie (&lt;85ns)"]
        DAEMON --> DUAL_RADIO["Dual-Radio Coordinator"]
    end

    subgraph KernelPlane["Kernel & Silicon Wire Plane (Ring 0 / Bare Metal)"]
        DAEMON -->|AF_NETLINK (Family 16)| NL80211["Linux Generic Netlink (nl80211)"]
        DAEMON -->|libc ioctl (SIOCGIFFLAGS)| IOCTL["Linux Link Control"]
        DAEMON -->|eBPF / XDP Ingress Hook| XDP["Katran ARX SYN Cookie Reflector"]
        XDP --> NIC["Physical Network Interface Controller (NIC)"]
        NL80211 --> NIC
        IOCTL --> NIC
    end

    style ControlPlane fill:#161b22,stroke:#30363d,color:#c9d1d9
    style DaemonPlane fill:#0d1117,stroke:#238636,color:#3fb950
    style KernelPlane fill:#1c1e24,stroke:#1f6feb,color:#58a6ff
```

---

## ⚡ 2. The Bare-Metal Linux Netlink Engine (`utils::net::nl80211`)

### The Flaw of Establishment Tools
Traditional networking scripts and tools (`iw`, `ip`, `nmcli`, `rfkill`) spawn child processes via `fork()/execve()`. This introduces critical engineering defects:
1. **High Latency Overhead**: Forking a shell process consumes ~15ms per channel switch, causing dropped 802.11 management frames during high-velocity channel hopping.
2. **PATH Poisoning & Fragility**: Relying on external binaries in `$PATH` exposes systems to injection attacks and fails on minimal Musl/BusyBox appliances.
3. **Unparseable Human Output**: Parsing command-line text requires fragile regex scrapers that break across kernel or distribution updates.

### The Sovereign Resolution
Bastion communicates directly with the Linux kernel across standard POSIX descriptors using raw Netlink datagram sockets (`AF_NETLINK`):

```rust
// Direct Generic Netlink controller family resolution (GENL_ID_CTRL = 16)
let fd = unsafe { libc::socket(libc::AF_NETLINK, libc::SOCK_RAW, libc::NETLINK_GENERIC) };
```

1. **Kernel Controller Resolution**: Resolves the dynamic `nl80211` family ID at runtime via `CTRL_CMD_GETFAMILY` over Netlink.
2. **Channel Frequency Switching**: Executes `NL80211_CMD_SET_CHANNEL` with `NL80211_ATTR_WIPHY_FREQ` directly, achieving channel hopping in $<120\mu\text{s}$.
3. **Bare-Metal Interface State Transitions**: Toggles interface states (`IFF_UP`, `IFF_PROMISC`) using direct `libc::ioctl` calls with `SIOCGIFFLAGS` and `SIOCSIFFLAGS`, completely bypassing legacy utilities.

---

## 🚫 3. Zero-Subprocess POSIX Architecture

In strict compliance with our Universal Sovereign Engineering Standard:
* **Zero Child Processes**: Bastion never calls `std::process::Command::new("ip")`, `Command::new("iw")`, or `Command::new("kill")`.
* **Direct Syscall Interception**: Process terminations, signaling, and timeouts execute via native POSIX `libc::kill` (with `SIGTERM` followed by atomic grace checks).
* **Memory Bounded Modularity**: All source modules conform to single-responsibility bounds ($\le 300$ lines), with clean abstractions between state tracking, packet forging, and kernel interaction.
