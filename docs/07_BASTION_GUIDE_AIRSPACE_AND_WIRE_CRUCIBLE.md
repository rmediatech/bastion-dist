# 📡 Bastion Guide: Airspace Reconnaissance, RFC EAPOL Dissection & Wire Crucibles

This guide covers the **Bastion Airspace** subsystem, detailing the unified CLI command surface, bare-metal IEEE 802.11 / IEEE 802.1X zero-copy dissectors, 802.11w Protected Management Frames (PMF) security auditing, and the adversarial wire-level test crucibles.

---

## 🎛️ 1. Airspace CLI Command Surface

The `bastion airspace` command tree provides sovereign, offline RF reconnaissance and wireless security evaluation without reliance on bloated external suites (`aircrack-ng`, `iw`, `kismet`).

```bash
# 1. Interactive overview, command syntax & hardware auto-discovery
bastion airspace

# 2. Inspect discovered wireless hardware adapters, bus topologies & operational roles
bastion airspace adapters
bastion airspace adapters -f json

# 3. Multi-channel hopping RF survey (1, 6, 11, 36, 40, 44, 48, 149, 153...)
sudo bastion airspace scan --duration 10

# 4. High-fidelity single-channel locked capture (pinned to AP channel)
sudo bastion airspace scan -c 36 --duration 15 -f tui
sudo bastion airspace scan -c 6 --duration 5 -f md

# 5. Audit 802.11w Protected Management Frames (PMF) & deauth vulnerability
sudo bastion airspace audit 78:67:0E:BA:7E:76 -c 36
sudo bastion airspace audit 78:67:0E:BA:7E:76 -f json

# 6. Launch the sovereign real-time Ratatui radar dashboard matrix
sudo bastion airspace dashboard
```

---

## 🔬 2. Wire-Level RFC 802.11 & 802.1X Dissectors

Legacy tools typically employ naive substring searches or unverified sliding windows (`frame.windows(8)`) looking for LLC/SNAP bytes, introducing false positives from application payload collisions and failing on variable-length headers. Bastion implements zero-copy dissectors directly derived from authoritative IEEE specifications:

### IEEE 802.11-2020 MAC Header Dissection (`airspace::wifi::mac_frame`)
* **Frame Control Decomposition (§9.2.4)**: Zero-copy parsing of Protocol Version, Frame Type (Management, Control, Data), and Subtypes.
* **ToDS / FromDS Address Multiplexing**: Resolves dynamic 802.11 address field roles:
  * `ToDS=0, FromDS=0`: Direct / IBSS (`Addr1=DA, Addr2=SA, Addr3=BSSID`).
  * `ToDS=1, FromDS=0`: Station $\to$ AP (`Addr1=BSSID, Addr2=STA, Addr3=DA`).
  * `ToDS=0, FromDS=1`: AP $\to$ Station (`Addr1=STA, Addr2=BSSID, Addr3=SA`).
  * `ToDS=1, FromDS=1`: WDS Bridge (`Addr1=RA, Addr2=TA, Addr3=DA, Addr4=SA`).
* **Variable Header Sizing**: Dynamically computes 802.11 header offsets across 4-address WDS frames (`+6B`), QoS Control headers (`+2B` when subtype bit 3 is set), and HT Control fields (`+4B` when order bit is set).
* **RFC 1042 LLC/SNAP Encapsulation**: Validates the 8-byte SNAP header (`0xAA, 0xAA, 0x03, 0x00, 0x00, 0x00`) and extracts the underlying EtherType (`0x888E` for EAPOL).

### IEEE 802.1X-2020 & 802.11i EAPOL Dissector (`airspace::wifi::eapol`)
* **802.1X Packet Validation**: Enforces Version $\ge 1$, Packet Type 3 (EAPOL-Key), and verifies body bounds against payload buffers.
* **EAPOL-Key Bitmask Parsing**: Dissects 95-byte base frames, resolving Descriptor Type (RSN vs legacy WPA), Key Descriptor Version (RC4 vs AES-128-CMAC), Key Type (Pairwise vs Group), Key ACK, Key MIC, and Secure flags.
* **4-Way Message Classification**:
  * **Msg 1**: `Pairwise=1, Key ACK=1, Key MIC=0`. Extracts 32-byte `ANonce` and inspects Key Data for PMKIDs.
  * **Msg 2**: `Pairwise=1, Key ACK=0, Key MIC=1, Secure=0`. Extracts 32-byte `SNonce` and 16-byte `MIC`.
  * **Msg 3**: `Pairwise=1, Key ACK=1, Key MIC=1`. Confirms pairwise key installation.
  * **Msg 4**: `Pairwise=1, Key ACK=0, Key MIC=1, Secure=1`. Final handshake confirmation.
* **Zero-Client PMKID Extraction**: Navigates Vendor-Specific Key Data Elements (KDE: `Type 0xDD`, `OUI 00-0F-AC`, `DataType 0x04`) to extract the 16-byte Pairwise Master Key Identifier (PMKID) directly from Msg 1 without awaiting a connecting client.
* **Replay Counter Auditing (`HandshakeTracker`)**: Asserts mathematical Replay Counter synchronization between Msg 1 and Msg 2, discarding spoofed or replayed handshake frames.

---

## 🛡️ 3. 802.11w Protected Management Frames (PMF) Audit

Bastion inspects the RSN Information Element (Tag 48) in captured Beacons and Probe Responses to verify Protected Management Frames compliance:

| PMF Status | MFPR (Bit 6) | MFPC (Bit 7) | Deauth Vulnerability | Security Posture |
| :--- | :---: | :---: | :---: | :--- |
| **Required** | `1` | `1` | **IMMUNE** | Mandatory 802.11w / WPA3-Personal. All management frames cryptographically authenticated via BIP-CMAC-128. Rogue deauth frames discarded by hardware MAC. |
| **Capable** | `0` | `1` | **VULNERABLE** | Optional PMF. Modern clients negotiate PMF, but legacy clients fall back to unprotected management frames, susceptible to targeted deauth downgrade. |
| **Disabled** | `0` | `0` | **CRITICAL** | Legacy WPA2-PSK without management protection. Susceptible to standard unauthenticated deauthentication injection. |

---

## 🧪 4. Adversarial Wire-Level RFC Crucibles

The `airspace` crate implements rigorous adversarial self-attack test batteries enforcing zero-panic boundaries and specification compliance:

1. **`adversarial_wire_fuzz_tests.rs`**:
   * *Truncated Frame Containment*: Asserts zero panics when slicing sub-24B frames.
   * *IE Allocation Bombs*: Validates bounds protection against 65,535 pairwise suite counts.
   * *ANSI Escape Code Scrubbing*: Verifies that malicious terminal control sequences embedded in SSIDs are neutralized before display.
   * *10,000 Mutated Frames*: 10,000 bit-flipped and truncated beacon frames fuzzed with zero panics.
2. **`adversarial_eapol_rfc_crucible.rs`**:
   * *Truncated Headers*: Verifies sub-24B MAC and sub-99B EAPOL frames return `None` safely.
   * *Key Data Length Bomb*: Rejects frames where claimed `u16` Key Data Length exceeds physical frame bounds.
   * *PMKID KDE Extraction*: Asserts bit-perfect extraction of 16-byte PMKID from Vendor Specific KDEs.
   * *Replay Counter Audit*: Verifies `HandshakeTracker` rejects spoofed Msg 2 frames with desynchronized replay counters.
   * *Address Multiplexing*: Asserts correct resolution across IBSS, Station-to-AP, AP-to-Station, and WDS topologies.
   * *10,000 Mutated EAPOL Fuzz Battery*: 10,000 mutated EAPOL frames parsed with 100% zero-panic guarantee.
3. **`adversarial_airspace_lab_crucible.rs`**:
   * *Device Flood Pointer Stability*: Selection pointer remains anchored during 500-device high-velocity surges.
   * *Corrupted Ingestion*: Handles non-numeric NaN signals and missing channel tags gracefully.
   * *Dual-Radio Allocation Rollback*: Reverts hardware state safely if secondary interface initialization fails.
