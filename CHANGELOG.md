# 📋 Changelog

All notable changes to **Bastion** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.4.0] - 2026-11-10

### Sovereign Help, UNIX Man Page, 4 Formats & 5 Operational Lenses (`SOVEREIGN-CHANGELOG-E2E-01`)

- **`[CLI/UX]` & `[Protocol]` Unified Sovereign CLI Architecture (`cli::cli_args`, `cli::cli_projector`, `cli::cli_help_man`)**:
  - **Root Cause Dissection**: Previously, the CLI command surface lacked unified output serialization across formats and operational perspectives, forcing operators into ad-hoc scripting for parsing reports. CSV exports lacked protection against formula injection attacks (`=`, `+`, `-`, `@`, `\t`, `\r`), violating `SOVEREIGN-SECURITY-01`. Offline environments lacked immediate access to comprehensive manuals without starting background daemons.
  - **Architectural & Mathematical Resolution**: Modularized the CLI crate under strict OJP bounds (`<= 300` lines per file, entry shell `<= 80` lines). Implemented zero-allocation `Projector` supporting 4 interchangeable formats (`tui`, `md`, `json`, `csv`) and 5 operational lenses (`1: grid`, `2: story`, `3: tree`, `4: spec`, `5: waterfall`). Hardened CSV outputs with automatic formula injection neutralization prefixing single-quotes (`'`). Added offline `bastion help [<topic>]` and a full UNIX-style manual page (`bastion man`) rendered through the operator's active `$PAGER` (`less -R`).
  - **Empirical Verification**: Verified via adversarial unit tests in `cli/tests/cli_tests.rs` asserting 100% green coverage across all lenses, formats, and formula-injection attack vectors.

### Bare-Metal Netlink (nl80211) & Dual-Radio Hardware Orchestration (`SOVEREIGN-CHANGELOG-E2E-01`)

- **`[Hardware/Kernel]` & `[Protocol]` Bare-Metal Linux Netlink (`nl80211`) & Kernel ioctl Driver Engine (`utils::net::nl80211`)**:
  - **Root Cause Dissection**: Historically, Wi-Fi hardware configuration (monitor mode switching, channel hopping, and link state transitions) relied on spawning external command binaries (`ip link`, `iw dev`, `iw set channel`, `nmcli`, `rfkill`). This introduced process-fork latency (~15ms per channel switch), shell dependency fragility, and vulnerability to PATH tampering in violation of `SOVEREIGN-RFC-BAREMETAL-01`.
  - **Architectural & Mathematical Resolution**: Engineered a zero-dependency, bare-metal Netlink and ioctl engine in `utils::net::nl80211`. Implemented direct Linux Generic Netlink (`AF_NETLINK` family `NETLINK_GENERIC`) controller family resolution (`GENL_ID_CTRL` = 16) for `nl80211` (`NL80211_CMD_SET_INTERFACE` = 6, `NL80211_CMD_SET_CHANNEL` = 65, `NL80211_ATTR_WIPHY_FREQ` = 38). Engineered `LinkControl` utilizing direct libc ioctls (`SIOCGIFFLAGS`, `SIOCSIFFLAGS`, `SIOCGIFINDEX`) for sub-microsecond link UP/DOWN and promiscuous mode toggling. Added multi-band center frequency translation across 2.4 GHz (1-14) and 5 GHz (UNII-1/UNII-3).
  - **Empirical Verification**: Verified via unit tests (`test_controller_initialization`, `test_channel_to_frequency_2ghz`, `test_channel_to_frequency_5ghz`, `test_loopback_ifindex_and_up`, `test_socket_open_and_family_lookup`) and empirical execution on standard 802.11 monitor-capable hardware.

- **`[Hardware/RF]` Intelligent Dual-Radio Auto-Pairing & Hardware Identification (`utils::net::nl80211::sysfs`, `strike_lab::interface_selector`)**:
  - **Root Cause Dissection**: Hardware radio selection previously required manual user input or blind array indexing, risking selection of internal motherboard Wi-Fi instead of high-gain external USB adapters.
  - **Architectural & Mathematical Resolution**: Implemented `SysfsDiscovery` directly traversing `/sys/class/net/` and USB device descriptor paths to detect hardware chipsets across vendor IDs and product descriptors. Implemented `auto_pair_dual_adapters()` prioritizing external dedicated radios over onboard PCIe Wi-Fi, automatically pre-configuring `[1] SNIFFER` for continuous zero-drop capture and `[2] INJECTOR` for targeted transmission.
  - **Empirical Verification**: Empirically verified in isolated test harness (`tests/live_wifi_tests.rs::test_live_sysfs_auto_pair_dual_adapters`), accurately classifying `wlan1` and `wlan2` as dedicated external adapters while segregating `wlan0` onboard Wi-Fi.

- **`[Crucible/Refactor]` Workspace-Wide Subprocess Eradication & Strict Line Bound Conformance (`utils`, `strike_lab`, `daemon`)**:
  - **Root Cause Dissection**: Monolithic files in `utils/src/net/sniffer.rs`, `strike_lab/src/bin/poc_strike.rs`, and `daemon/src/main.rs` exceeded the 300-line modularity bound and retained legacy subprocess calls (`Command::new("kill")`, `Command::new("iw")`).
  - **Architectural & Mathematical Resolution**: Purged all subprocess spawns from `sniffer.rs`, `adapter.rs`, `interfaces.rs`, `guard.rs`, and `target_scanner.rs`. Replaced `daemon/src/main.rs` process termination with direct `libc::kill` syscalls (SIGTERM/SIGKILL). Bounded all touched files strictly under 300 lines (`daemon/src/main.rs`: 296 lines, `sniffer.rs`: 198 lines, `poc_strike.rs`: 228 lines, `dashboard.rs`: 245 lines).
  - **Empirical Verification**: 100% green test suite across all 14 workspace crates (360+ tests passed).

---

## [1.3.0] - 2026-11-10

### Architectural Parity & Multi-Subsystem Fortification (`SOVEREIGN-CHANGELOG-E2E-01`)

- **`[Hardware/Kernel]` & `[State/SST]` CIDR Subnet Unrolling & Autonomous JSON Quarantine Feed Hot-Sync (`daemon::firewall::xdp_strategy`)**:
  - **Root Cause Dissection**: Previously, `XdpMapStrategy` parsed blocklist lines strictly using `line.parse::<IpAddr>()`, immediately failing on CIDR blocks (`192.0.2.0/24`) and unable to ingest structured JSON quarantine feeds emitted by upstream perimeter defenses (`phylax` and `propylea`).
  - **Architectural & Mathematical Resolution**: Upgraded `XdpMapStrategy` with `extract_ip_tokens` and `expand_cidr_or_ip`. Automatically detects plaintext CIDRs or structured JSON objects (`target`, `ip`, `cidr`, `quarantine`, `bans`). Enforces safe linear BPF map capacity guardrails (rejecting overly broad subnets `< /22` to prevent kernel memory exhaustion) while unrolling subnets into line-rate kernel `IP_BLOCKLIST` drops.
  - **Empirical Verification**: Verified via `daemon/src/firewall/tests/xdp_strategy_tests.rs` (`test_xdp_strategy_supports_cidr_subnet_unrolling`, `test_xdp_strategy_rejects_oversized_cidr_subnets`, and `test_xdp_strategy_ingests_json_quarantine_feeds`).

- **`[Sensor/Recon]` Zero-Copy Passive TCP/IP Stack & OS Fingerprinting Engine (`recon::fingerprint::passive`)**:
  - **Root Cause Dissection**: Prior OS reconnaissance lacked a zero-copy integration layer connecting raw wire packet bytes directly to heuristic heuristic matching, requiring downstream callers to manually unpack network layers.
  - **Architectural & Mathematical Resolution**: Engineered `recon::fingerprint::passive` exposing `infer_os_from_packet` and `infer_os_from_parameters` backed by `OnceLock<HeuristicMatcher>`. Evaluates initial TTL recovery (handling degraded routed hops), window sizes, DF bit flags, and TCP option ordering (`MSS,NOP,WS,SACKOK,TS`) to classify Linux, Windows 10/11, macOS Darwin, and embedded network routers with high confidence.
  - **Empirical Verification**: Verified via `recon/src/fingerprint/passive.rs` test suite (`test_passive_infer_linux_syn`, `test_passive_infer_windows11_syn`, `test_passive_infer_degraded_ttl_routed_packet`).

- **`[Network/Scanner]` Adaptive RTT Timing Engine & ANSI Control Byte Sanitizer (`utils::net::tcp_scanner`)**:
  - **Root Cause Dissection**: Port scanning used a fixed 750ms timeout per probe without latency learning, causing excessive delay on high-speed LANs and dropped probes on congested links. Furthermore, un-sanitized service banner grabbing risked terminal escape injection.
  - **Architectural & Mathematical Resolution**: Deconstructed the monolithic scanner into modular sub-modules (`connect.rs`, `syn.rs`, `mod.rs`). Implemented `AdaptiveRttEstimator` calculating an Exponential Weighted Moving Average (EWMA: 7/8 old + 1/8 sample) in microseconds with 3x RTT margin. Added `grab_banners` mode with `scrub_ansi_and_control` to scrub CSI/OSC escape codes and unprintable control bytes.
  - **Empirical Verification**: Verified via `utils/src/net/tests/tcp_scanner_tests.rs` (`test_scanner_adaptive_rtt_estimator` and `test_scanner_banner_grabbing_and_ansi_scrubbing`).

- **`[Protocol/Forge]` RFC 793, RFC 7323 & RFC 2018 TCP Options Serializer (`forge::packet::tcp_options`)**:
  - **Root Cause Dissection**: Packet construction lacked a dedicated, zero-allocation builder for complex TCP option negotiations, preventing advanced probe crafting.
  - **Architectural & Mathematical Resolution**: Created `TcpOptionsBuilder` supporting MSS, Window Scale, SACK Permitted, Timestamps, and NOPs, with automatic padding to 32-bit word alignment.
  - **Empirical Verification**: Verified via `forge/src/packet/tcp_options.rs` (`test_tcp_options_mss_and_sack_aligned` and `test_tcp_options_standard_syn_pack`).

- **`[Hardware/RF]` IEEE 802.11w-2009 Protected Management Frames (PMF) Inspector (`strike::wifi::pmf`)**:
  - **Root Cause Dissection**: Wireless interdiction tools blindly injected deauthentication frames against modern WPA3/802.11w networks where Management Frame Protection (MFP) cryptographically authenticates frames, resulting in wasted transmissions.
  - **Architectural & Mathematical Resolution**: Engineered `parse_rsn_pmf_status` parsing the 802.11 RSN Information Element (Tag 48) to extract MFPR (Bit 6) and MFPC (Bit 7), classifying target BSSIDs into `Required`, `Capable`, or `Disabled`.
  - **Empirical Verification**: Verified via `strike/src/wifi/pmf.rs` (`test_pmf_required_detection`, `test_pmf_capable_optional_detection`, and `test_pmf_disabled_or_empty_detection`).

---

## [1.2.0] - 2026-11-07

### Architectural Ground-Floor Rebuild (`SOVEREIGN-CHANGELOG-E2E-01`)

- **`[Hardware/Kernel]` & `[Protocol]` In-Kernel XDP L3/L4 Anti-Spoofing, Fragmentation Anomaly Guard, Volumetric UDP/ICMP Flood Limiter & OpenMetrics Exporter (`constants::flood_defense`, `bastion-ebpf`, `bastion-xdp`)**:
  - **Root Cause Dissection**: While `v1.1.0` neutralized TCP SYN backlog exhaustion via Katran-style ARX SYN Cookies, adversarial L3/L4 vectors—specifically LAND attacks (`saddr == daddr`), Martian/Bogon source spoofing (`0.0.0.0/8`, `127.0.0.0/8`, multicast/broadcast `>= 224.0.0.0`), IPv4 fragmentation anomalies (RFC 1858 tiny-fragment L4 header splitting, Teardrop overlapping fragment offsets, Ping-of-Death `offset * 8 + payload > 65535`, contradictory `DF+MF` bit combinations), and volumetric UDP amplification / ICMP Echo Request ping floods—could still traverse the XDP ingress hook before L4 TCP evaluation.
  - **Architectural & Mathematical Resolution**:
    1. Engineered `constants::flood_defense` (`#![no_std]`) providing zero-branching bitwise evaluators shared 1:1 between the in-kernel `bastion-ebpf` program and the userspace `XdpRing0Simulator`.
    2. Implemented `inspect_ipv4_spoofing(saddr_host, daddr_host)` enforcing sub-nanosecond drop rules against LAND (`saddr == daddr`), Unspecified (`0.0.0.0/8`), Loopback-on-Wire (`127.0.0.0/8`), and Multicast/Broadcast (`>= 224.0.0.0`) source addresses.
    3. Implemented `inspect_ipv4_fragmentation(frag_off_host, tot_len_host, ihl_bytes)` enforcing hardware-speed drop rules against contradictory `DF+MF` flags, RFC 1858 tiny initial fragments (`offset == 0 && MF == 1 && tot_len < 48`), Teardrop overlapping non-zero offsets (`0 < offset * 8 < 24`), and Ping-of-Death reassembly overflows (`offset * 8 + payload > 65535`).
    4. Implemented `evaluate_volumetric_flood` using a sliding 1-second decay window (`VOLUMETRIC_WINDOW_NS = 1_000_000_000`) to throttle per-source UDP floods (`DEFAULT_UDP_THRESHOLD = 512 pkt/s`) and ICMP Echo Request Type 8 floods (`DEFAULT_ICMP_THRESHOLD = 16 pkt/s`) while allowing normal ICMP Echo Replies (Type 0) and non-flood UDP datagrams at line rate.
    5. Expanded `XdpDefenseTelemetry` (`#[repr(C)]`) with `spoofed_dropped`, `frag_dropped`, `udp_flood_dropped`, and `icmp_flood_dropped`, and added zero-dependency OpenMetrics/Prometheus exposition (`render_openmetrics()`).
  - **Empirical Verification**: Verified via `xdp/tests/adversarial_xdp_syn_cookie_tests.rs` (`test_adversarial_bogon_martian_land_spoofing_and_fragment_anomalies` and `test_volumetric_udp_and_icmp_flood_mitigation_and_openmetrics`) across 50 total workspace tests.

---

## [1.1.0] - 2026-11-06

### Architectural Ground-Floor Rebuild (`SOVEREIGN-CHANGELOG-E2E-01`)

- **`[Hardware/Kernel]` & `[Protocol]` Katran-Style Stateless In-Kernel XDP SYN Cookie Reflector (`bastion-ebpf`, `constants::syn_cookie`)**:
  - **Root Cause Dissection**: Prior `bastion-ebpf` acted strictly as a static IP/Port drop filter (`IP_BLOCKLIST`, `IPV6_BLOCKLIST`, `PORT_BLOCKLIST`). Under volumetric spoofed TCP SYN floods, unblocked random source IPs bypassed XDP into the Linux kernel networking stack, allocating `sk_buff` structures and exhausting SYN backlog queues.Furthermore, passing `&XdpContext` directly across `#[inline(never)]` BPF functions triggers LLVM's argument promotion pass, unpacking context pointers into registers with `<<= 32` zero-extension instructions that the Linux BPF verifier rejects (`"pointer arithmetic with <<= operator prohibited"`).
  - **Architectural & Mathematical Resolution**:
    1. Engineered a zero-allocation `#![no_std]` Single Source of Truth (`constants::syn_cookie`) shared between ring-0 eBPF (`bastion-ebpf`), `bastion` (`core`), and `bastion-xdp`.
    2. Implemented 32-bit ARX cryptographic SYN Cookies packing a 2-bit 4-tier MSS index (`536, 1220, 1360, 1460` bytes—eliminating the 3-slot out-of-bounds gap in legacy implementations) into bits `31..=30` and a 30-bit ARX authentication tag (`2^30` brute-force resistance) into bits `29..=0`, backed by rotating 60-second dual-bucket 128-bit secrets (`SynCookieSecret`).
    3. Implemented `PacketCtxRef<'a>` (`SOVEREIGN-BPF-CTX-SHIELD-01`) to hold `&'a XdpContext` on the caller stack frame and `compute_syn_cookie_packed` (`#[inline(never)]` 4-register ABI), permanently preventing LLVM pointer promotion and BPF verifier state explosion.
    4. When per-source SYN velocity exceeds threshold, `bastion-ebpf` truncates excess TCP options via `bpf_xdp_adjust_tail`, swaps Ethernet MACs, IPv4 addresses, and TCP ports in-place, stamps the 30-bit cookie into `tcp.seq`, computes RFC 1071 checksums via `bpf_csum_diff`, and bounces the SYN-ACK directly out the NIC wire via `XDP_TX` with zero `sk_buff` allocations. Valid client ACKs grant a 60s fast-pass grace window (`VERIFIED_GRACE_NS`), while unverified challenges auto-recover after `30s` (`CHALLENGE_TIMEOUT_NS`).
  - **Empirical Verification**: Verified via `xdp/tests/adversarial_xdp_syn_cookie_tests.rs` and the Digital Twin Ring-0 Wire Simulator (`bastion_xdp::simulator::XdpRing0Simulator`) covering `2^30` ACK forgery floods, `u32::MAX` sequence wrap-around, `bpf_xdp_adjust_tail` option shrinking, TFO payload drop protection, and `O(1)` `Ipv4Trie` CIDR blocklist integration.

---

## [1.0.6] - 2026-08-14

### Added
- **Target-Isolated Report Directory**: Telemetry report exports (`.csv` and `.json`) write exclusively to `./reports/` subfolders (`reports/discovered_hosts_<target>.<ext>`), preventing root binary folder pollution.
- **Dynamic C2 Room Inheritance**: `DaemonConfig` now persists the authenticated user's private `room_id` returned during PIN pairing into `/root/.config/rmediatech/daemon.json`.

### Fixed
- **CLI Flag Filtering**: Filtered out `--with-defaults` for `discover` command in `remote.rs` to prevent argument parsing crashes.
- **Stderr Piping**: Enabled stderr output piping for helper subprocesses to record diagnostic logs.
- **Protocol-Agnostic URL Parsing**: Refactored `get_or_pair_token` in `pairing.rs` to derive HTTP/HTTPS base URLs dynamically from `--signaling-url`.

---

## [1.0.5] - 2026-08-14

### Fixed
- **Dynamic URL Derivation**: Removed legacy hardcoded `https://next.rmediatech.com` URLs from daemon pairing routines.

---

## [1.0.4] - 2026-08-13

### Added
- **Multi-Protocol Recon Suites**: Added UDP scouting, BLE scans, and passive IoT traffic monitoring capabilities.
- **Headless Scanning Mode**: Added `--no-tui` option to `ble_scan` binary for programmatic background execution.
