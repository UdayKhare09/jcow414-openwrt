# OpenWrt for Sercomm JioFiber JCOW414 (EcoNet EN7528DU)

This repository provides an open-source, upstream OpenWrt distribution with full GPON WAN, DSA Ethernet switching, and Dual-Band Wi-Fi support for the **Sercomm JioFiber JCOW414** optical network router.

---

## 1. Hardware Specifications

| Component | Specification |
| :--- | :--- |
| **SoC** | EcoNet / Airoha EN7528DU (Dual-core MIPS 24KEc, `mipsel_24kc` Little-Endian) |
| **RAM** | 512 MB DDR3 |
| **Flash** | 256 MB SPI NAND (`mtd0` TcBoot, `mtd1` romfile, `mtd2` Bank 1 / tclinux, `mtd5` Bank 2 / tclinux_slave) |
| **LAN Switch** | MediaTek MT7530 internal switch via DSA (`lan1`–`lan4` bridged into `br-lan`) |
| **Wi-Fi 2.4 GHz** | MediaTek MT7603E (802.11b/g/n, 2x2 MIMO) |
| **Wi-Fi 5 GHz** | MediaTek MT7615E / MT7663 (802.11a/n/ac, 4x4 MIMO, VHT80) |
| **Optical WAN** | Integrated EcoNet GPON MAC (`1fb60000.xmac`, netdev `pon0`), Airoha EN7571 BOSA controller |

---

## 2. GPON & OMCI Architecture

* **ITU-T G.984 GPON MAC:** Native hardware acceleration with dynamic Alloc-ID and GEM port provisioning directly from OLT grants.
* **ITU-T G.988 OMCI Stack:** In-kernel baseline OMCI stack with multi-packet MIB upload (`net/xpon/omci/agent.c`) and Radisys (`RSYS`) interoperability.
* **Carrier Datapath:** 802.1Q VLAN 1015 (`pon0.1015`), IPoE DHCP with mandatory Option 60 (Vendor Class Identifier) and Option 61 (Client Identifier). Option 12 (Hostname) is suppressed (`*`) to ensure compatibility with carrier BRAS gateways.

---

## 3. Out-Of-The-Box Defaults

On first boot after installation, the firmware automatically initializes with generic, secure defaults:

* **LAN Network:**
  * IP: `192.168.1.1`
  * Netmask: `255.255.255.0`
  * DHCP Server: Active (pool `192.168.1.100` – `192.168.1.250`)
* **Wi-Fi (Active by default):**
  * **2.4 GHz:** SSID `OpenWrt-JCOW414-2.4G` (Channel 6, HT20, WPA2-PSK: `openwrt123`)
  * **5 GHz:** SSID `OpenWrt-JCOW414-5G` (Channel 36, VHT80, WPA2-PSK: `openwrt123`)
* **WAN Interface:**
  * Device: `pon0.1015`
  * Protocol: DHCP (IPv4) + DHCPv6
  * Hostname: `*` (suppressed)
* **Identity:**
  * Factory Native Identity (derives hardware MAC from bootloader nvmem and hardware GPON serial from reservearea).

---

## 4. Subscriber Identity Configuration (One-Click Configurator)

The firmware includes a unified identity configuration tool (`/usr/sbin/jiofiber-config`):

### Mode A: Native Hardware Identity
If your JioFiber subscription is provisioned directly on this physical router:
```sh
jiofiber-config native
```
This automatically clears MAC overrides and uses the hardware's native burnt-in MAC and serial number.

### Mode B: Clone Subscribed Router
If you are running OpenWrt on a secondary router and wish to use the subscription from your primary router:
```sh
jiofiber-config clone <SUBSCRIBED_GPON_SN> <SUBSCRIBED_WAN_MAC> [PLOAM_PASS]
```

**Example:**
```sh
jiofiber-config clone SCOMXXXXXXXX AA:BB:CC:DD:EE:FF 00000001
```

This single command automatically:
1. Configures the xPON OMCI agent with the subscribed GPON serial and password.
2. Sets the WAN MAC override (`network.wan.macaddr`).
3. Formats and sets DHCP Option 60 (`network.wan.vendorid`).
4. Formats and sets DHCP Option 61 (`network.wan.clientid`).
5. Commits UCI and reloads networking and xPON services in one operation.

### Via Web Interface (LuCI)
* **xPON Profile:** Navigate to **xPON → OMCI → Profiles** and select `JioFiber - Sercomm JCOW414`.
* **WAN Interface:** Navigate to **Network → Interfaces → wan** to adjust MAC and DHCP options.

---

## 5. Building the Firmware

### Prerequisites (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install -y build-essential libncurses-dev gawk git subversion libssl-dev \
  gettext unzip zlib1g-dev file python3 libelf-dev
```

### Build Commands
```bash
# 1. Update and install feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 2. Load JCOW414 default config
cp jcow414.defconfig .config
make defconfig

# 3. Compile firmware
make -j$(nproc)
```

The resulting sysupgrade image will be generated at:
`bin/targets/airoha/en7528/openwrt-airoha-en7528-sercomm_jiofiber-jcow414-squashfs-sysupgrade.bin`

---

## 6. Flashing Instructions

> [!CAUTION]
> **Dual-Bank Safety Rule:** Always flash to **Bank 1 (`mtd2 = tclinux`) ONLY**. 
> **NEVER** flash, erase, or overwrite `mtd0` (bootloader), `mtd1` (romfile), or `mtd5` (`tclinux_slave` / factory stock Jio OS). Keeping Bank 2 intact ensures your router remains 100% unbrickable.

---

### Method A: Flashing from Factory Stock Jio OS using JFxploit (No Disassembly Needed)

You do **not** need to open the router casing or solder UART pins to flash OpenWrt. You can gain root access remotely over LAN using [JFxploit](https://github.com/JFC-Group/JFxploit):

#### Step 1: Connect to the Router
Connect your PC to any LAN port (`LAN1`–`LAN4`) with an Ethernet cable. The stock router web interface is at `192.168.29.1`.

#### Step 2: Run JFxploit on your PC
```bash
# Clone JFxploit
git clone https://github.com/JFC-Group/JFxploit.git
cd JFxploit

# Setup environment (uv recommended, or use pip)
uv sync

# Exploit router web interface and open root telnet
uv run jfxploit run 192.168.29.1
```
*(If you changed the default admin password in the Jio web GUI, pass `-u superadmin -P 'your_password'`)*.

#### Step 3: Host the OpenWrt Firmware on your PC
Download `openwrt-jcow414-squashfs-tclinux.trx` from the [Releases](https://github.com/UdayKhare09/jcow414-openwrt/releases) page. In your terminal, navigate to the folder containing the `.trx` file and start a temporary web server:
```bash
python3 -m http.server 8000
```
*(Find your PC's LAN IP, e.g. `192.168.29.150` via `ip a` or `ipconfig`)*.

#### Step 4: Flash via Telnet on the Router
Open a new terminal and connect to the exploited router's telnet shell:
```bash
telnet 192.168.29.1
```
Inside the router root shell, download and write the firmware:
```sh
cd /tmp

# Download the OpenWrt TRX image from your PC
wget http://192.168.29.xxx:8000/openwrt-jcow414-squashfs-tclinux.trx -O /tmp/fw.trx

# Write to Bank 1 (tclinux)
mtd -f write /tmp/fw.trx tclinux

# Switch active boot bank to Bank 1 and reboot
/usr/bin/sys bootflag swap
reboot
```

#### Step 5: Access OpenWrt
Wait ~60 seconds for the router to boot:
* **Web UI (LuCI):** `http://192.168.1.1` (no password by default)
* **Default Wi-Fi:** `OpenWrt-JCOW414-2.4G` and `OpenWrt-JCOW414-5G` (Password: `openwrt123`)

---

### Method B: Upgrading from an Existing OpenWrt Installation

If your router is already running OpenWrt:

#### Via LuCI Web Interface:
1. Navigate to **System → Backup / Flash Firmware**.
2. Under **Flash new firmware image**, upload `openwrt-jcow414-squashfs-sysupgrade.bin`.
3. Click **Upload** and proceed.

#### Via SSH Terminal:
```bash
# Copy sysupgrade image to router
scp openwrt-jcow414-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/

# Run sysupgrade
ssh root@192.168.1.1 "sysupgrade -v /tmp/openwrt-jcow414-squashfs-sysupgrade.bin"
```

---

### Method C: Safety Fallback — How to Revert to Factory Jio OS

Because Bank 2 (`mtd5 = tclinux_slave`) remains untouched:

* **From OpenWrt shell:**
  ```sh
  fw_setenv bootflag 1
  reboot
  ```
* **Or via TcBoot Serial Console (`bldr>`):**
  ```text
  bldr> memwl 80020000 31
  bldr> flash dfc0000 80020000 1
  bldr> go
  ```
The router will immediately boot back into the factory stock Jio OS!
