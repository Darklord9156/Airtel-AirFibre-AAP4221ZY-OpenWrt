# OpenWrt on Airtel AirFiber AAP4221ZY

A field-tested guide to unlocking the stock firmware shell, enabling the serial console, booting OpenWrt through TFTP, and installing OpenWrt permanently on the Airtel AirFiber **AAP4221ZY** indoor router. The resulting setup can be used as a standalone router without the matching outdoor unit.

> **Tested hardware:** Airtel AirFiber AAP4221ZY (OpenWrt identifies the board as `airtel,aap4221zy`; upstream support references the Zyxel EX3310-T0)  
> **Tested OpenWrt build:** `25.12.6`, revision `r33311-981c978eff`  
> **Target:** `mediatek/filogic`

This README records the steps that worked on one device. It is not a guarantee for every hardware or stock-firmware revision. Read the upstream device-support notes before attempting installation. Flashing carries a risk of bricking the router.

## What was completed

- Obtained a root-capable shell on the stock firmware by exporting, decrypting, editing and restoring the vendor configuration backup.
- Enabled UART bootloader output using `fw_setenv EngDebugFlag 0x01`.
- Entered the MediaTek U-Boot console from the Zyxel `ZHAL>` shell.
- Booted the device-specific OpenWrt initramfs image into RAM over TFTP.
- Verified the permanent sysupgrade image by SHA-256 and with OpenWrt's test-only sysupgrade check.
- Installed OpenWrt to NAND with `sysupgrade -n` and rebooted successfully from flash.
- Confirmed the SquashFS root and writable UBIFS overlay.
- Accessed LuCI and configured a working 2.4 GHz access point.
- Reassigned LAN2 as the PPPoE WAN interface, leaving the PoE-capable dedicated WAN port unused.
- Resolved an ISP MAC-binding rejection by cloning the previous router's WAN MAC onto LAN2. The PPPoE interface then reported `up: true` and received an IPv4 address, default route and DNS servers.

The 5 GHz radio was detected but a 5 GHz access point was not configured during this run. Custom LED behaviour was not configured.

## Hardware and safety notes

The AAP4221ZY uses a MediaTek Filogic platform with two LAN ports and a dedicated WAN port. The dedicated WAN port provides PoE for the matching outdoor unit. In this standalone-router setup, the outdoor unit was not used; the dedicated WAN port was left disconnected and LAN2 was repurposed as WAN.

**Do not connect ordinary Ethernet equipment to the dedicated WAN port unless you have independently confirmed that its PoE output is disabled.** The device-specific OpenWrt notes identify the WAN port's PoE control as GPIO 10. Using LAN2 as WAN avoids that port entirely.

Never erase or write raw flash partitions manually as part of this procedure. In particular, preserve factory calibration data. Use the device-specific OpenWrt sysupgrade process.

Read these references before starting:

- [OpenWrt device support and installation instructions — PR #22809](https://github.com/openwrt/openwrt/pull/22809)
- [Official OpenWrt 25.12.6 MediaTek Filogic downloads](https://downloads.openwrt.org/releases/25.12.6/targets/mediatek/filogic/)
- [OpenWrt AAP4221ZY image definition](https://github.com/openwrt/openwrt/blob/v25.12.6/target/linux/mediatek/image/filogic.mk)

## 1. Requirements

- Airtel AirFiber AAP4221ZY router.
- Access to the stock firmware's local administration page and an authorised administrator account.
- Windows PC with Ethernet.
- Python and OpenSSL for the local configuration-backup helper workflow.
- A 3.3 V USB-to-TTL UART adapter.
- PuTTY or another serial terminal.
- Tftpd64 or another TFTP server.
- Ethernet cable for a direct PC-to-router LAN connection.
- The correct AAP4221ZY OpenWrt initramfs and sysupgrade images from the official release directory.

### UART serial settings

| Parameter | Value |
|---|---|
| Electrical level | 3.3 V TTL |
| Baud rate | 115200 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

Use the board's documented GND, TX and RX connections. **Do not connect the UART adapter's VCC pin** to the router.

## 2. Obtain root-capable access on the stock firmware

This was the prerequisite that allowed the UART console to be enabled. The method used the stock router's configuration export/restore workflow; it did not involve opening the router or writing raw flash.

### 2.1 Export a stock configuration backup

1. Connect a PC to the router's LAN and sign in to the stock web administration interface with an authorised administrator account.
2. Use the configuration backup/restore utility to request a configuration export and download the resulting encrypted backup (`config-current.cfg`). The stock interface's backup handler creates the file exposed as `romfile.cfg`.
3. Keep the original encrypted backup unchanged as a recovery copy.
4. Confirm that the downloaded file is an encrypted OpenSSL-salted backup (the helper used in this workflow checks for the `Salted__` header).

The local helper used for this work was `zyxel_backup_restore.py`. Its download operation handles the stock web session, requests the backup, downloads the encrypted configuration and checks the file format. Example invocation (replace placeholders locally; never commit credentials):

```powershell
python zyxel_backup_restore.py download -u <STOCK_ADMIN_USER> -p <STOCK_ADMIN_PASSWORD> -o config-current.cfg
```

### 2.2 Decrypt the backup and inspect the XML

Decrypt the downloaded configuration to XML using the helper and the encryption parameters that match the stock firmware build:

```powershell
python zyxel_backup_restore.py decrypt -i config-current.cfg -o config-current.xml
```

The configuration format observed in this workflow was an OpenSSL salted AES-256-CBC file containing XML. The stock firmware uses a firmware-embedded configuration encryption key and legacy MD5-based key derivation. **The key is intentionally not reproduced in this public README.** Do not publish the key, encrypted backups, decrypted XML, or router configuration files; they can expose credentials and other device-specific information. A backup from another firmware revision may use different parameters.

### 2.3 Modify only the access-control fields

The working PowerShell preparation script was `prepare_aap4221zy_unlock.ps1`. It expected `config-current.cfg` and `config-current.xml` in the working directory, preserved original copies, parsed the XML, made only the required access changes, re-encrypted the edited XML and performed a decrypt/encrypt round-trip verification.

The changes were:

| XML area | Setting | Value applied |
|---|---|---|
| `LoginPrivilegeMgmt` | `UserLevel` | Changed from `1` to `0` |
| `SSH` | `Type` | `all` |
| `SSH` | `Interface` | `LAN` |
| `TELNET` | `Type` | `all` |
| `TELNET` | `Interface` | `LAN` |

The helper produced `config-modified.xml` and the encrypted `config-modified.cfg`. It validated that the expected XML nodes had the intended values and checked that re-encrypting/decrypting the result preserved the XML bytes.

For reference, the script's expected input/output files were:

```text
config-current.cfg
config-current.xml
config-current.original.cfg
config-current.original.xml
config-modified.xml
config-modified.cfg
config-modified-roundtrip.xml
```

Do not upload an edited file unless the XML validation and encryption round-trip both pass. Preserve the original backup so the stock configuration can be restored if required.

### 2.4 Restore the edited configuration and verify shell access

Restore the re-encrypted `config-modified.cfg` through the stock configuration restore function. The router validates the uploaded encrypted file, applies the configuration and reboots. The local backup/restore helper also supports a restore operation; its example syntax is:

```powershell
python zyxel_backup_restore.py restore -i config-modified.cfg -u <STOCK_ADMIN_USER> -p <STOCK_ADMIN_PASSWORD>
```

After the router has rebooted, reconnect from the trusted LAN using SSH. Telnet was also enabled on the LAN as a fallback during this process; **Telnet is unencrypted and should only be used temporarily on a trusted, isolated LAN**. Prefer SSH whenever it is available. Verify that the shell is running with root privileges before continuing:

```sh
id
```

Proceed only when the output confirms `uid=0`.

> **Security note:** This stock-firmware unlock path enables management shell services and changes a privilege field. It is intended only for a device you own or are authorised to administer. Do not expose SSH or Telnet to the Internet. Avoid leaving Telnet enabled on stock firmware after SSH access has been confirmed.

## 3. Enable UART output from the stock root shell

From the verified stock root shell, set the bootloader debug flag:

```sh
fw_setenv EngDebugFlag 0x01
```

Verify it:

```sh
fw_printenv EngDebugFlag
```

The expected value is:

```text
EngDebugFlag=0x01
```

Reboot the router. This enables the serial boot output needed for the next stage.

## 4. Connect UART and enter U-Boot

Open PuTTY using the UART settings above and power on/reboot the router. The Zyxel bootloader prompt appears as:

```text
ZHAL>
```

Enter `ATGU` twice, waiting for the prompt to return between commands:

```text
ZHAL> ATGU
```

After the first command returns to `ZHAL>`, enter it again:

```text
ZHAL> ATGU
```

The second command enters the MediaTek U-Boot shell:

```text
MT7981>
```

The AAP4221ZY boot chain is BL2 → U-Boot → Zyxel zloader. The device-specific OpenWrt instructions document the two-`ATGU` sequence.

## 5. Prepare TFTP on the Windows PC

Set the PC's Ethernet adapter connected to the router to a static IPv4 address:

| Setting | Value |
|---|---|
| PC Ethernet IP | `192.168.1.2` |
| Subnet mask | `255.255.255.0` |
| Default gateway | Leave blank |
| DNS | Leave blank |

In Tftpd64:

1. Set **Current Directory** to the folder containing the initramfs image.
2. Set **Server interfaces** to the Ethernet adapter with address `192.168.1.2`.
3. Leave Tftpd64 running while the router requests the image.

The filename used in this run was:

```text
openwrt-mediatek-filogic-airtel_aap4221zy-initramfs-kernel.bin
```

Use the exact local filename shown in the TFTP root directory when running the U-Boot command.

## 6. Boot the temporary OpenWrt image from RAM

At the `MT7981>` prompt, set the router and TFTP server addresses:

```text
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.2
```

Load the initramfs image into RAM:

```text
tftpboot 0x46000000 openwrt-mediatek-filogic-airtel_aap4221zy-initramfs-kernel.bin
```

The transfer completed successfully and reported `9,069,932` bytes transferred. Boot the image:

```text
bootm 0x46000000
```

OpenWrt started from RAM and provided a root shell. This step did not permanently flash the router.

Set a temporary root password:

```sh
passwd
```

Confirm the running build and board:

```sh
cat /etc/openwrt_release
cat /tmp/sysinfo/board_name
```

The tested build reported OpenWrt `25.12.6`, revision `r33311-981c978eff`, target `mediatek/filogic`, and board `airtel,aap4221zy`. The partition layout was inspected with:

```sh
cat /proc/mtd
```

## 7. Download and verify the permanent sysupgrade image

The sysupgrade image used in this run was:

```text
openwrt-25.12.6-mediatek-filogic-airtel_aap4221zy-squashfs-sysupgrade.bin
```

The SHA-256 computed on Windows matched the checksum for the official OpenWrt 25.12.6 image:

```text
020385fad2f82b6d60a017519044e311512d26643e05ab3dbe89c1ef9d6fa48c
```

Verify the file on Windows PowerShell (adjust the local folder/file name if needed):

```powershell
Get-FileHash "<PATH_TO_SYSUPGRADE_IMAGE>" -Algorithm SHA256
```

Always compare the result with the checksum published alongside the official image. Do not rely on the filename alone.

## 8. Copy and validate the sysupgrade image

The temporary initramfs environment did not provide an SFTP server, so Windows OpenSSH SCP was run in legacy SCP mode using `-O`:

```powershell
scp -O "<PATH_TO_SYSUPGRADE_IMAGE>" root@192.168.1.1:/tmp/
```

On the OpenWrt shell, calculate the copied image's SHA-256:

```sh
sha256sum /tmp/openwrt-25.12.6-mediatek-filogic-airtel_aap4221zy-squashfs-sysupgrade.bin
```

The copied file's hash matched the Windows file and the official checksum.

Run the test-only sysupgrade validation:

```sh
sysupgrade -T /tmp/openwrt-25.12.6-mediatek-filogic-airtel_aap4221zy-squashfs-sysupgrade.bin
echo $?
```

The test completed with exit code `0`.

## 9. Permanently install OpenWrt

After verifying the board name, image checksum and test-only result, install the image:

```sh
sysupgrade -n /tmp/openwrt-25.12.6-mediatek-filogic-airtel_aap4221zy-squashfs-sysupgrade.bin
```

`-n` means **do not preserve the previous configuration**. The router therefore needed its root password and network configuration set again after installation.

Do not interrupt power during flashing or reboot. The device-specific sysupgrade procedure creates the `zyfwinfo` UBI metadata volume required by the Zyxel bootloader.

## 10. Verify the permanent installation

After reboot, the serial boot log showed the bootloader reading `zyfwinfo` from the OpenWrt UBI bank and continuing the boot process. OpenWrt returned on the serial console.

Verify the root and overlay filesystems:

```sh
mount
df -h
```

The successful output included:

```text
/dev/root on /rom type squashfs
/dev/ubi0_3 on /overlay type ubifs
```

From Windows, verify that the router responds after reboot:

```powershell
ping 192.168.1.1
```

The test returned four replies with 0% packet loss. Pressing Enter in the serial console returned to the OpenWrt root prompt.

Set the permanent OpenWrt root password again because the installation used `sysupgrade -n`:

```sh
passwd
```

## 11. Access LuCI

LuCI and `uhttpd` were already included in the installed image. The web server was listening on ports 80 and 443.

Open the web interface from a PC connected to the router's LAN:

```text
http://192.168.1.1
```

Log in as `root` using the password set after the permanent flash.

## 12. Configure the 2.4 GHz Wi-Fi access point

In LuCI, open **Network → Wireless** and edit the 2.4 GHz radio (`radio0`). The working configuration was:

| Setting | Value |
|---|---|
| Mode | Access Point |
| Radio mode | AX |
| Channel | 1 |
| Channel width | 20 MHz |
| SSID | Your chosen SSID (not included in this public guide) |
| Network | `lan` |
| Encryption | WPA2-PSK |
| SSID broadcast | Enabled |

The wireless interface was enabled, changes were saved/applied, and a phone successfully connected to the access point.

The 5 GHz radio (`radio1`) was detected by OpenWrt, but a 5 GHz access point was not configured during this run.

## 13. Use LAN2 as the PPPoE WAN port

### Why LAN2 was used

The dedicated WAN port is PoE-capable and is intended to power the matching outdoor unit. Since this router was configured for standalone use, the dedicated WAN port was left unused. LAN2 was removed from the LAN bridge and repurposed as the PPPoE WAN interface. LAN1 remained available for local management.

Before changing the port assignment, keep the management PC connected through LAN1 or Wi-Fi. Do not reassign the port carrying your current management session.

### Reassign LAN2

The following configuration worked:

```sh
uci del_list network.@device[0].ports='lan2'
uci set network.wan.device='lan2'
uci set network.wan.proto='pppoe'
uci set network.wan.ipv6='0'
uci set network.wan6.device='lan2'
uci set network.wan6.disabled='1'
uci commit network
/etc/init.d/network reload
```

### Configure PPPoE credentials

In LuCI, open **Network → Interfaces → WAN → Edit**, set the protocol to **PPPoE**, and enter your own ISP-supplied username and password. Leave optional Access Concentrator, AC MAC Address and Service Name fields empty unless the ISP provides values. Do not put credentials in this repository.

### Resolve MAC binding, if the ISP reports it

The first PPPoE attempt returned an ISP-side rejection stating:

```text
Incorrect Calling-Station-Id (MAC_Address)
```

The previous router's WAN MAC was then cloned onto the OpenWrt LAN2 device. The PPPoE session subsequently connected. The old router must be powered off while its WAN MAC is being reused; never operate two devices with the same MAC on the same network at the same time.

Use the WAN MAC from your own previous router. Do not copy another device's address. The working UCI pattern was:

```sh
uci set network.lan2_dev='device'
uci set network.lan2_dev.name='lan2'
uci set network.lan2_dev.macaddr='<YOUR_PREVIOUS_ROUTER_WAN_MAC>'
uci commit network
/etc/init.d/network reload
```

Verify the resulting MAC:

```sh
ip link show lan2
```

The `link/ether` address should match the address being cloned.

### Verify PPPoE

```sh
ifstatus wan
```

The successful session reported:

- `"up": true`
- `"proto": "pppoe"`
- Device: `lan2`
- An assigned IPv4 address
- A default route and DNS servers

This confirms PPPoE session establishment. A separate external speed test is not documented here.

## Final working layout

| Connection | Role |
|---|---|
| LAN1 | Local LAN / management |
| LAN2 | PPPoE WAN |
| Dedicated WAN | Unused; PoE-capable port for the outdoor unit |
| 2.4 GHz Wi-Fi | Access point on `lan` |
| 5 GHz Wi-Fi | Radio detected; AP configuration not completed |
| LuCI | `http://192.168.1.1` |
| Firmware | OpenWrt `25.12.6` (`r33311-981c978eff`) |

## Security and publishing checklist

- Never commit router admin/root passwords, ISP credentials, serial numbers, unique MAC addresses, private keys, configuration backups, decrypted XML, or session files.
- Keep `config-current.cfg`, `config-current.xml`, `config-modified.cfg`, local `.env` files and any firmware-specific backup keys out of Git.
- The stock configuration backup format observed here uses legacy AES-CBC/MD5-based processing and a firmware-embedded key. Treat the format as weak: the decrypted XML can contain plaintext credentials and other sensitive configuration. Do not publish the key or the backup files.
- Telnet is plaintext. Use it only as a temporary fallback on a trusted LAN; prefer SSH and do not expose either service to the Internet.
- Do not publish the actual MAC address cloned from a previous router. Use a placeholder in public examples.
- Keep local copies of the initramfs and sysupgrade images for recovery. Link to official OpenWrt downloads instead of committing firmware binaries unless you have checked the applicable distribution terms.
- Re-check the official device instructions and release checksums before repeating the procedure on another unit.

## References

1. [OpenWrt support and installation procedure for Airtel AAP4221ZY — PR #22809](https://github.com/openwrt/openwrt/pull/22809)
2. [Official OpenWrt 25.12.6 MediaTek Filogic image directory](https://downloads.openwrt.org/releases/25.12.6/targets/mediatek/filogic/)
3. [OpenWrt AAP4221ZY device definition](https://github.com/openwrt/openwrt/blob/v25.12.6/target/linux/mediatek/image/filogic.mk)
4. [OpenWrt sysupgrade documentation](https://openwrt.org/docs/guide-user/installation/generic.sysupgrade)
