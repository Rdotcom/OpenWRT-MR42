# OpenWRT-MR42
# Flashing OpenWrt on Cisco Meraki MR42

## Network Topology

```
┌──────────────┐       ┌──────────────────┐       ┌──────────────┐
│    Laptop    │◄─────►│  Cisco Catalyst  │◄─────►│  Meraki MR42 │
│  192.168.1.  │ eth   │  3560 (switch)   │ eth   │  (unpowered  │
│   250/24     │       │  10M half-duplex │       │   at start)  │
└──────────────┘       └──────────────────┘       └──────────────┘
```

> **Why a switch?** Laptop Ethernet interfaces often fail to link up in time
> for the MR42's TFTP boot. The official OpenWrt documentation explicitly
> recommends a switch in this case.

---

## Prerequisites

### Hardware
- Laptop with an Ethernet port
- Cisco Catalyst 3560 (or any managed switch)
- Cisco Meraki MR42 (unpowered)
- 2× Ethernet cables (Cat5e or better)
- Paperclip / reset button tool

### Software (on laptop)
- **TFTP server** — e.g. `tftpd-hpa` (Linux), TFTPD64 (Windows)
- **Terminal / telnet client**
- **Python 3** (only if using UART method as fallback)

### Files (from [clayface/openwrt-cryptid](https://github.com/clayface/openwrt-cryptid))
| File | Purpose |
|------|---------|
| `mr42_u-boot.mbn` | U-Boot image to flash |
| `openwrt-ipq806x-generic-meraki_mr42-initramfs-fit-uImage.itb` | Temporary initramfs |
| `openwrt-ipq806x-generic-meraki_mr42-squashfs-sysupgrade-*.bin` | Final OpenWrt image |

Place `mr42_u-boot.mbn` and the initramfs `.itb` file in your TFTP root directory.

---

## Step 1 — Cisco 3560 Configuration

Set the relevant ports to **10 Mbps half-duplex** (required for reliable TFTP boot):

```cisco
enable
configure terminal
interface Fa0/2
 speed 10
 duplex half
 no shutdown
interface Fa0/4
 speed 10
 duplex half
 no shutdown
end
write memory
```

> Adjust your port numbers to match your cabling. Port 1 → laptop, Port 2 → MR42.
> Disable any other ports or VLANs to keep the network isolated.

---

## Step 2 — Laptop Configuration

### Set static IP
```bash
# Linux
sudo ip addr add 192.168.1.250/24 dev eth0
sudo ip link set eth0 up

# Windows (PowerShell)
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.250 -PrefixLength 24
```

### Start TFTP server
```bash
# Linux (tftpd-hpa)
sudo mkdir -p /srv/tftp
sudo cp mr42_u-boot.mbn openwrt-ipq806x-generic-meraki_mr42-initramfs-fit-uImage.itb /srv/tftp/
sudo systemctl restart tftpd-hpa

# Verify
tftp 192.168.1.250 -c get mr42_u-boot.mbn -m binary
```

---

## Step 3 — Enter Diagnostic Mode on MR42 (not working on my case)

1. **Do NOT** plug in MR42 power yet.
2. Connect Ethernet cable from **switch port 2** to MR42.
3. Hold the **reset button** while plugging in power.
4. After ~10 seconds, the **orange LED** will start flashing. 
5. **Release** the reset button.
6. **Press reset twice** (quickly).
7. LED should turn **blue** (or remain flashing — both are OK).

> If the LED goes rainbow instead of blue, the device may be locked by
> Meraki. Use the UART method instead. 

---

## Step 4 — Flash U-Boot via Telnet

Connect to the MR42:
```bash
telnet 192.168.1.1
```

### Option A: One-liner (if TFTP server is confirmed working)
```bash
/etc/update_uboot.sh 192.168.1.250 mr42_u-boot.mbn
```

### Option B: Manual (safer, verify checksum)
```bash
cd /tmp
tftp-hpa 192.168.1.250 -m binary -c get mr42_u-boot.mbn
md5sum mr42_u-boot.mbn   # verify against expected value

echo 1 > /sys/devices/platform/msm_nand/boot_layout
mtd erase /dev/mtd1
nandwrite -pam /dev/mtd1 mr42_u-boot.mbn
echo 0 > /sys/devices/platform/msm_nand/boot_layout
```

> **Watch the `nandwrite` output carefully.** You must see all 3 blocks
> written. An error here will likely brick the device. 

Once done, **power off** the MR42.  **Do NOT stop the TFTP server.** 

---

## Step 5 — Boot Initramfs

1. Hold the **reset button**. 
2. Power on the MR42.
3. **Release reset immediately.**
4. The device will TFTP-boot the initramfs image.
5. Wait ~2 minutes for a **solid white LED**.

> If TFTP fails, the switch port speed/duplex setting is the most common
> cause. Verify 10M half-duplex is active on both ports.

---

## Step 6 — Flash Permanent U-Boot (via UART)

> ⚠️ **This step requires UART access.** Do NOT connect over Ethernet.

Connect USB-TTL adapter to MR42 UART header:
| Pin | Signal |
|-----|--------|
| 1   | 3.3V — **DO NOT CONNECT** |
| 2   | TXD |
| 3   | RXD |
| 4   | GND |

Open serial terminal (115200 8N1), then:
```bash
insmod mtd-rw i_want_a_brick=1
mtd erase /dev/mtd8
nandwrite -pam /dev/mtd8 /root/mr42_u-boot.mbn
```

Expected output:
```
Writing data to block 0 at offset 0x0
Writing data to block 1 at offset 0x20000
Writing data to block 2 at offset 0x40000
```

---

## Step 7 — Sysupgrade

Reconnect over Ethernet (laptop → switch → MR42). The device is now at
`192.168.1.1`.

```bash
# Copy sysupgrade image to device
scp openwrt-ipq806x-generic-meraki_mr42-squashfs-sysupgrade-*.bin root@192.168.1.1:/tmp/

# Flash
ssh root@192.168.1.1 "sysupgrade -n /tmp/openwrt-ipq806x-generic-meraki_mr42-squashfs-sysupgrade-*.bin"
```

Or use **LuCI** at `http://192.168.1.1` → System → Backup / Flash Firmware.

---

## Step 8 — Cleanup (Optional)

Remove Meraki UBI volumes to reclaim space:
```bash
for i in diagnostic1 part.old storage part.safe; do
  ubirmvol /dev/ubi0 -N $i
done
```

> **NEVER** delete the `ART` partition (`mtd12`) — it contains Wi-Fi calibrations.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| TFTP request never arrives | Switch ports not at 10M half-duplex; verify with `show interface` |
| Orange LED never flashes | Device is Meraki-locked; use UART method |
| Rainbow LED after reset | Device is locked; UART required |
| `nandwrite` error | **Device likely bricked.** UART recovery needed. |
| Initramfs boots but no network | Check switch port status: `show interface GigabitEthernet0/2` |
| 5 GHz Wi-Fi limited to 40 MHz | Known `ath10k` limitation; not fixable in OpenWrt |

---

## References

- [OpenWrt MR42 forum thread](https://forum.openwrt.org/t/openwrt-on-meraki-mr42/32193)
- [clayface/openwrt-cryptid](https://github.com/clayface/openwrt-cryptid) — U-Boot & tools
- [OpenWrt ipq806x target](https://openwrt.org/toh/start) — device images
- [OpenWrt Custom Firmware](https://krishnendu.com/cisco-meraki-m42-wifi-router-openwrt-custom-firmware/) - intruction
