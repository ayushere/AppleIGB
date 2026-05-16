# AppleIGB — Tahoe-Compatible Fork

Intel onboard LAN driver for macOS, patched to fix kernel panics on **macOS 26.x (Tahoe)**.

This fork fixes four bugs in the original Apple-specific driver code that caused repeated kernel panics on macOS Tahoe 26.x due to stricter mbuf validity checks, Probabilistic GZAlloc (PGZ) guard pages, and a changed output queue threading model introduced in that release.

---

## Supported Devices

| Family | PCI IDs |
|---|---|
| 82575 | 0x10A7, 0x10A9, 0x10D6 |
| 82576 | 0x10C9, 0x10E6, 0x10E7, 0x10E8, 0x1526, 0x150A, 0x1518, 0x150D |
| 82580 | 0x150E, 0x150F, 0x1510, 0x1511, 0x1516, 0x1527 |
| dh89xxcc | 0x0438, 0x034A, 0x043C, 0x0440 |
| **I210 / I211** | 0x1533, 0x1534, 0x1535, 0x1536, 0x1537, 0x1538, 0x1539, 0x157B, 0x157C |
| I350 | 0x1521, 0x1522, 0x1523, 0x1524, 0x1546 |
| I354 | 0x1F40, 0x1F41, 0x1F45 |

---

## What Was Fixed

### Bug 1 — TX path mbuf out-of-bounds (panics on Tahoe 26.4 and 26.5)

**Affected functions:** `tcp6_hdr()`, `igb_tx_csum()`

Both functions walked IPv6 extension headers by doing `ip6++`, which advances by `sizeof(struct ip6_hdr) = 40 bytes`. Real IPv6 extension headers have variable lengths with the format `{next_hdr, hdr_ext_len, ...}` where the actual byte length is `(hdr_ext_len + 1) * 8`. The incorrect stride caused the pointer to walk past the end of the mbuf's data region.

On macOS Tahoe, Probabilistic GZAlloc places a guard page immediately after each 512-byte mbuf zone element. The errant read hit that guard page and triggered a kernel panic:

```
panic: Kernel trap at ..., type=14 (page fault)
Zone: mbuf, Kind: out-of-bounds (high confidence), Access: 29 byte(s) past
Kernel Extensions in backtrace: com.amdosx.driver.AppleIGB(5.11.4)
```

**Fix:** Replaced the `ip6++` loop with proper bounds-checked extension header traversal using the correct `(cursor[1] + 1) * 8` stride.

### Bug 2 — RX path silent mbuf corruption (panic: `mbuf len -3`)

**Affected function:** `igb_add_rx_frag()`

`mbuf_copyback()` was called with `MBUF_WAITOK` in the RX interrupt context (incorrect — sleeping is not allowed in interrupt context). When the call failed, the function logged an error but returned `true` (success) anyway, handing a partially-corrupted mbuf up to the network stack. Tahoe's stricter mbuf validity check then caught the corrupted length field and panicked:

```
panic: Failed mbuf validity check: mbuf len -3 type 1 flags 0x2 rcvif en6
```

**Fix:** Changed to `MBUF_DONTWAIT`, and on failure the mbuf is freed and `NULL` is returned to the caller so the descriptor is dropped cleanly.

---

## Installation (OpenCore)

### Prerequisites

- OpenCore bootloader
- SIP disabled (required for third-party kexts)
- macOS Tahoe 26.x (for older macOS use the [original upstream](https://github.com/Shaneee/AppleIGB))

### Steps

1. **Download** `AppleIGB.kext` from the [Releases](../../releases/latest) page.

2. **Mount your EFI partition:**
   ```bash
   sudo diskutil mount disk0s1
   ```
   *(replace `disk0s1` with your EFI partition — check with `diskutil list`)*

3. **Back up your existing kext:**
   ```bash
   cp -r /Volumes/EFI/EFI/OC/Kexts/AppleIGB.kext ~/Desktop/AppleIGB.kext.bak
   ```

4. **Replace with the patched kext:**
   ```bash
   sudo rm -rf /Volumes/EFI/EFI/OC/Kexts/AppleIGB.kext
   sudo cp -r ~/Downloads/AppleIGB.kext /Volumes/EFI/EFI/OC/Kexts/
   sudo chown -R 0:0 /Volumes/EFI/EFI/OC/Kexts/AppleIGB.kext
   ```

5. **Unmount EFI and reboot:**
   ```bash
   sudo diskutil unmount /Volumes/EFI
   sudo reboot
   ```

No `config.plist` changes are needed — the bundle identifier (`com.amdosx.driver.AppleIGB`) and kext name are unchanged.

### Verify it loaded

After reboot:
```bash
kextstat | grep AppleIGB
```
You should see `com.amdosx.driver.AppleIGB (5.11.5)`.

---

## Building from Source

Requires Xcode 14 or later. macOS Tahoe SDK (Xcode 26+) recommended.

```bash
git clone https://github.com/ayushere/AppleIGB.git
cd AppleIGB
xcodebuild -scheme AppleIGB -configuration Release CODE_SIGN_IDENTITY="-"
```

Output: `~/Library/Developer/Xcode/DerivedData/AppleIGB-.../Build/Products/Release/AppleIGB.kext`

---

## Changelog

### v5.11.6 — 2026-05-16
- Fixed TX ring use-after-free race between `outputPacket()` and `igb_clean_tx_irq()`
- Root cause: on Tahoe 26.x, `IOBasicOutputQueue` delivers `outputPacket` from a separate high-priority thread instead of the workloop; `igb_clean_tx_irq` runs on the workloop — without a lock they race on the TX ring, freeing an mbuf while `outputPacket` still holds it
- Fix: added `IOSimpleLock` (txLock) protecting the TX buffer write-map section in `outputPacket` and the buffer-free section in `igb_clean_tx_irq`
- Same pattern as the locking fix applied in IntelMausiEthernet for the same Tahoe threading change

### v5.11.5 — 2026-05-15
- Fixed `tcp6_hdr()`: replaced unbounded `ip6++` traversal with correct bounds-checked IPv6 extension header walking using `(cursor[1] + 1) * 8` stride
- Fixed `igb_tx_csum()`: same IPv6 extension header traversal fix; added early return on malformed packet
- Fixed `igb_add_rx_frag()`: changed `MBUF_WAITOK` → `MBUF_DONTWAIT`; on `mbuf_copyback` failure, now frees the mbuf and returns `NULL` instead of passing a corrupted mbuf to the stack
- Built against macOS 26.5 SDK (Xcode 26.5)

### v5.11.4 — 2022-08-27 *(upstream — Shaneee/donatengit)*
- Updated to Intel IGB Linux driver source v5.11.4
- Adopted IntelMausi link management and code structure
- Added EEE mode toggle option
- Explicit TX queue stall on busy transmit

---

## Credits

- **Intel** — original `igb` Linux kernel driver source (v5.11.4)
- **hnak** ([InsanelyMac](https://www.insanelymac.com/forum/topic/273073-appleigbkext/)) — original macOS port of the IGB driver
- **Shanee** ([Shaneee/AppleIGB](https://github.com/Shaneee/AppleIGB)) — update to IGB v5.11.4, macOS kext structure
- **donatengit** ([donatengit/AppleIGB](https://github.com/donatengit/AppleIGB)) — IntelMausi code structure merge, EEE support, stability improvements
- **ayushere** — macOS Tahoe 26.x compatibility fixes (v5.11.5)

---

## Known Issues (inherited from upstream)

- Instability and connection drops reported on X570 and TRX40 motherboards — use SmallTree or USB Ethernet on those platforms
- Does not use MSI-X interrupts

---

## License

The Intel IGB driver source is released under the GNU General Public License v2. This project inherits that license.
