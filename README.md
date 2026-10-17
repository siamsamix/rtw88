# ⚠️ Fork note: RTL8821CU USB support (RFE type 7)

This fork carries a **one-line addition** to `rtw8821c.c` so that RTL8821CU
USB adapters that report **RFE type 7** in their efuse work. Without it,
`rtw88`'s 8821c probe aborts and the adapter never gets a network interface.

This section is a fork-local note. The upstream README starts below the
horizontal rule at the end of this section.

---

## TL;DR

```diff
--- a/rtw8821c.c
+++ b/rtw8821c.c
@@ -1951,6 +1951,9 @@ static const struct rtw_rfe_def rtw8821c_rfe_defs[] = {
 	[2] = RTW_DEF_RFE_EXT(8821c, 0, 0, 0, 2),
 	[4] = RTW_DEF_RFE_EXT(8821c, 0, 0, 0, 2),
 	[6] = RTW_DEF_RFE(8821c, 0, 0, 0),
+	/* some RTL8821CU USB adapters report RFE 7; coex code groups it with
+	 * RFE 2 (rfe_btg is set for 2/4/7), so reuse the RFE 2 definition */
+	[7] = RTW_DEF_RFE_EXT(8821c, 0, 0, 0, 2),
 };
```

Build it with DKMS (`dkms.conf` is already in the tree) and blacklist the
in-tree `rtw88_8821cu` module, because both register the same USB IDs and the
in-tree one cannot handle RFE 7:

```sh
git clone https://github.com/siamsamix/rtw88.git && cd rtw88
sudo cp -r . /usr/src/rtw88-0.6          # name must match PACKAGE_NAME-VERSION in dkms.conf
sudo dkms add rtw88/0.6
sudo dkms install rtw88/0.6 -k "$(uname -r)"
echo 'blacklist rtw88_8821cu' | sudo tee /etc/modprobe.d/rtw88-usb-dongles.conf
sudo modprobe rtw_8821cu
ip -br link                              # a new wlanX interface should appear
```

## Affected hardware

- Realtek **RTL8821CU** USB adapters (AC600 class, dual-band) whose efuse
  reports RFE type 7. Observed on USB ID `0bda:c811`; the same chip is also
  sold as `7392:c811` (Edimax) and under various generic brands.
- `lsusb` shows the adapter, but no driver binds and no `wlanX` is created.

## Symptoms in `dmesg`

```
rtw88_8821cu 2-2.1:1.0: Firmware version 24.11.0, H2C version 12
rtw88_8821cu 2-2.1:1.0: rfe 7 isn't supported
rtw88_8821cu 2-2.1:1.0: failed to setup chip efuse info
rtw88_8821cu 2-2.1:1.0: failed to setup chip information
```

The rfkill switch, firmware package and USB mode choices are all fine — the
driver is simply rejecting the chip revision before it can continue.

## Why the alternatives don't help

- **Stock in-tree `rtw88` (any recent kernel) and upstream lwfinger `rtw88`
  without this patch**: `rtw8821c_rfe_defs[]` only has entries for RFE 0, 2, 4
  and 6, so the RFE 7 lookup fails exactly as above. The fork exists to add
  that entry.
- **Realtek vendor driver** (`morrownr/8821cu-20210916`, AUR
  `rtl8821cu-morrownr-dkms-git`): does not compile on kernels that removed
  `struct pppoe_tag` from `include/linux/if_pppox.h` (observed on 7.2):

  ```
  core/rtw_br_ext.c:1154:24: error: no member named 'tag_data' in 'struct pppoe_tag'
  ```

  Upstream master of that driver still uses the removed struct, and the AUR
  package maintainer now recommends the in-kernel driver instead. It is not a
  usable fallback on current kernels.

## Why RFE 7 = RFE 2

The driver already treats RFE 7 as equivalent to RFE 2 elsewhere:

- `rtw8821c_read_efuse()` sets `hal->rfe_btg = true` for RFE values
  2/4/7/10/12/15 (the BTG / "1-Ant, Main" front-end family).
- `rtw8821c_coex_cfg_rfe_type()` groups cases 2, 7, 10 and 15 together.

So RFE 7 reuses the RFE 2 definition, `RTW_DEF_RFE_EXT(8821c, 0, 0, 0, 2)`,
which is how the chip is already handled in every other code path. This is a
best-effort mapping of the TX power limit table; if you have Realtek
documentation or efuse dumps for RFE 7, corrections are welcome.

## Notes on module names

- This fork's modules are named `rtw_8821cu`, `rtw_core`, `rtw_usb`, …
  The in-tree kernel modules are named `rtw88_8821cu`, `rtw88_core`, …
- DKMS installs into `/usr/lib/modules/$(uname -r)/updates/dkms/`, which
  `depmod` ranks above the in-tree `kernel/` tree, so the fork's module wins
  the modalias match.
- Blacklisting `rtw88_8821cu` is still recommended: the in-tree module can
  never work with RFE 7 hardware and would otherwise probe the device first
  on some setups.

## Verify it works

```sh
sudo dmesg | grep -i rtw_8821cu     # expect "Firmware version ..." and no "rfe 7 isn't supported"
ip -br link                         # new wlanX present
iw phy | grep -A1 'Band 2'          # 5 GHz band with VHT capabilities
sudo iw dev wlanX scan | grep -E 'freq:|SSID:'
```

After associating, `iw dev wlanX link` should show a VHT rate (up to
390 Mbps of MAC throughput on this AC600-class chip: 80 MHz, 1 spatial
stream). Required firmware (`rtw8821c_fw.bin`, package `linux-firmware`) is
the same one the in-tree driver uses — no extra firmware needed.

## Kernel updates

DKMS recompiles this module automatically on kernel upgrades via its install
hook. If a future kernel changes an API the driver uses, the build will fail
in the DKMS log; the fix is to rebase the fork on the latest upstream (or
pull upstream into it), re-apply the three added lines above to
`/usr/src/rtw88-0.6/rtw8821c.c`, and run:

```sh
sudo dkms install rtw88/0.6 -k "$(uname -r)"
```

## Undo / uninstall

```sh
sudo modprobe -r rtw_8821cu
sudo dkms remove rtw88/0.6 --all
sudo rm -rf /usr/src/rtw88-0.6 /etc/modprobe.d/rtw88-usb-dongles.conf
```

## Tested environment

- CachyOS (Arch-based), kernel `7.2.8-1-cachyos`, dkms 3.4.3,
  `linux-firmware` 20260916.
- Hardware: generic RTL8821CU dongle, USB ID `0bda:c811`, efuse RFE 7.
- Based on upstream commit `a56bcd2` ("8821a/8812a: Clear BIT_BT_PTA_EN only
  for USB", 2026-05-21).
- Verified: probe succeeds, firmware 24.11.0 loads, `wlan1` is created, the
  radio scans both 2.4 GHz and 5 GHz, VHT capabilities advertise 390 Mbps RX
  and TX, and the adapter associates with an 802.11ac access point at 5 GHz.
- Other 8821CU revisions (RFE 0/2/4/6) are unaffected by this patch and keep
  using their existing definitions.


---
# rtw88 downstream 🐧
### This is a downstream repo with a primary purpose of supporting development, testing and maintenance for the Realtek rtw88 series of WiFi 5 drivers in the Linux kernel.

The most recent addition to this repo is the driver for the RTL8814AU chipset. Testing is needed so if you have an adapter based on the RTL8814AU chipset, please test and report.

Update: There is now also a driver for RTL8814AE PCIe cards, ID 10ec:8813. lspci may call it RTL8813AE, same thing. Please test and report if you still have this card.

Update: RTL8812AE and RTL8821AE are supported now. Please report if it works/doesn't work.

🌟 The code in this repo stays in sync with the `wireless-next` repository, with additional changes to accommodate kernel API changes over time.

📌 **Note**: The `wireless-next` repo contains the code set for the ***next*** kernel version. If kernel 6.X is out, kernel mainline repo is on 6.X+1-rcY, and `wireless-next` targets kernel 6.X+2 material.

---

## Compatibility
Compatible with **Linux kernel versions 5.4 and newer** as long as your distro hasn't modified any kernel APIs. RHEL and all distros based on RHEL will have modified kernel APIs and are unlikely to be compatible with this driver.


#### Supported Chipsets
- **PCIe**: RTL8723DE, RTL8812AE, RTL8814AE, RTL8821AE, RTL8821CE, RTL8822BE, RTL8822CE
- **SDIO**: RTL8723CS, RTL8723DS, RTL8821CS, RTL8822BS, RTL8822CS
- **USB** : RTL8723DU, RTL8811AU, RTL8811CU, RTL8812AU, RTL8812BU, RTL8812CU
- **USB** : RTL8814AU, RTL8821AU, RTL8821CU, RTL8822BU, RTL8822CU

---

## Issues 🚨
Report problems in Issues after you have checked the [Q&A](#qa) at bottom of this README.

⚠️ If you see a line such as:

`make[1]: *** /lib/modules/5.17.5-300.fc36.x86_64/build: No such file or directory.` **Stop.**

This indicates **you have NOT installed the kernel headers.** 

Use the following instructions for that step.

## Installation Guide

### Prerequisites 📋
Below are prerequisites for common Linux distributions __before__  installing this driver.

#### Ubuntu
```bash
sudo apt update && sudo apt upgrade
```
```bash
sudo apt install linux-headers-generic build-essential git
```

#### Fedora
```bash
sudo dnf update
```
```bash
sudo dnf install kernel-devel git
```

#### openSUSE
```bash
sudo zypper install make gcc kernel-devel kernel-default-devel git libopenssl-devel
```

#### Arch
```bash
sudo pacman -Syu
```
```bash
sudo pacman -Sy base-devel git linux-firmware
```
Remember to install the corresponding [kernel headers](https://archlinux.org/packages/?q=Headers+and+scripts+for+building+modules) which are also needed for compilation.

#### Raspberry Pi OS
```bash
sudo apt update && sudo apt upgrade
```
```bash
sudo apt install -y raspberrypi-kernel-headers build-essential git
```

---

### Installation Using DKMS 🔄
Installing this driver via DKMS is highly recommended, especially if Secure Boot is enabled on your machine. Using DKMS (Dynamic Kernel Module Support) ensures that the `rtw88` kernel modules are automatically rebuilt and re-signed whenever the Linux kernel is updated. Without DKMS, these drivers would stop working after each kernel update, requiring manual re-compilation and re-signing. DKMS should be available through your distribution’s package manager. You can learn more about DKMS [here](https://github.com/dell/dkms).

1. Install `dkms` and all its required dependencies using your preferred package manager.

2. Clone the `rtw88` GitHub repository
   ```
   git clone https://github.com/lwfinger/rtw88
   ```

3. Build, sign, and install the rtw88 driver
   ```
   cd rtw88
   ```
   ```
   sudo dkms install $PWD
   ```

4. Install the firmware necessary for the rtw88 driver
   ```
   sudo make install_fw
   ```

5. Copy the configuration file `rtw88.conf` to `/etc/modprobe.d/`
   ```
   sudo cp rtw88.conf /etc/modprobe.d/
   ```

6. Enroll the MOK (Machine Owner Key), this is needed **ONLY IF** Secure Boot is enabled on your machine.
   ```
   sudo mokutil --import /var/lib/dkms/mok.pub
   ```
   For Ubuntu-based distro users, run this command instead
   ```
   sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
   ```
   
   Note: At this point, you will be requested to enter a password. Remember this password and re-enter it after rebooting your system in order to enroll your new MOK into your system's UEFI. Please see [this tutorial](https://github.com/dell/dkms?tab=readme-ov-file#secure-boot) for more details.


---

### Installation Using make🛠

You will need to rebuild and reinstall the driver manually after each kernel updates if you choose this way to install the driver. This method is **NOT RECOMMENDED** for systems with Secure Boot enabled.

```bash
git clone https://github.com/lwfinger/rtw88
```
```bash
cd rtw88
```
```bash
make
```
```bash
sudo make install
```
```bash
sudo make install_fw
```
```bash
sudo cp rtw88.conf /etc/modprobe.d/
```
---

### Installation for Arch-based Distros 🛠

This is the best way for Arch-based distro users to install this driver, one more step is required after running `makepkg -si` if Secure Boot is enabled on your machine: Enroll the MOK. Please see the step 6 in [Installation Using DKMS](#installation-using-dkms-) for details.

```bash
git clone https://aur.archlinux.org/rtw88-dkms-git.git
```
```bash
cd rtw88-dkms-git
```
```bash
makepkg -si
```
---

## Important Information
Below is important information for using this driver.

### 1. Blacklisting 🚫
The configuration file `rtw88.conf` in `/etc/modprobe.d/` will blacklist all in-kernel rtw88 drivers, however, it will not blacklist out-of-kernel vendor drivers. You will need to uninstall any out-of-kernel vendor drivers that you have installed that may conflict. 

### 2. Recovery Problems After Sleep/Hibernation 🛌
Some BIOSs have trouble changing power state from D3hot to D0. If you have this problem, run 

``` bash
sudo cp reload_rtw88.sh /usr/lib/systemd/system-sleep/
```

That script will unload the driver before sleep or hibernation, and reload it following resumption.

### 3. How To Load/Unload Kernel Modules (aka Drivers)🪛

The kernel modules for RTL8821CU are used to demonstrate here, you need to change the module name according to the chip that your Wi-Fi adapter uses.

This unloads the modules for RTL8821CU, due to some pecularities in the modprobe utility, two steps are required. If you are not sure the names of the loaded modules for your Wi-Fi adapter, run `lsmod | grep -i rtw`
```
sudo modprobe -r rtw_8821cu
sudo modprobe -r rtw_core

# This command will show nothing because the modules for RTL8821CU were unloaded.
lsmod | grep -i rtw
```

This loads the modules for RTL8821CU, only a single modprobe call is required.
```
sudo modprobe rtw_8821cu

# This command will show that the modules for RTL8821CU were loaded.
lsmod | grep -i rtw
rtw_8821cu             16384  0
rtw_usb                36864  1 rtw_8821cu
rtw_8821c              98304  1 rtw_8821cu
rtw_core              294912  2 rtw_usb,rtw_8821c
mac80211             1220608  2 rtw_usb,rtw_core
cfg80211             1064960  2 rtw_core,mac80211
```

### 4. How To Update The Driver Installed via DKMS

1. Remove the rtw88 driver and its source code.
   ```
   sudo dkms remove rtw88/0.6 --all
   ```
   ```
   sudo rm -r /usr/src/rtw88-0.6/
   ```

2. Run this command in the rtw88 source directory to pull the latest code 
   ```
   git pull
   ```

3. Build, sign, and install the rtw88 driver from the latest code
   ```
   sudo dkms install $PWD
   ```
---

## Q&A

### Q1: How to remove this driver if it doesn't work as expected?

For users who installed this driver via DKMS, run
```
sudo dkms remove rtw88/0.6 --all
```
```
sudo rm -rf /usr/src/rtw88-0.6
```
```
sudo rm -f /etc/modprobe.d/rtw88.conf
```

For users who installed this driver via `make`, run these commands in the rtw88 source directory and then the rtw88 driver will be unloaded and removed.

```
sudo make uninstall
```
```
sudo rm /etc/modprobe.d/rtw88.conf
```

For Arch-based distro users, run

```
sudo pacman -Rn rtw88-dkms-git
```

### Q2: My Wi-Fi adapter is plugged into a USB 3 port. How can I keep it in USB 2 mode to avoid potential interference in the 2.4 GHz band?
The module `rtw_usb` has a parameter named `switch_usb_mode` which can enable or disable USB mode switching, setting it to "n" will keep your adapter in USB 2 mode.

Steps:
1. Open /etc/modprobe.d/rtw88.conf with your preferred editor.
2. Change `switch_usb_mode=y` to `switch_usb_mode=n` in line 7.
3. Unplug the Wi-Fi adapter.
4. Reboot (or [unload the rtw88 drivers](#3-how-to-loadunload-kernel-modules-aka-drivers))
5. Plug your Wi-Fi adapter back in.

### Q3: My Wi-Fi adapter still doesn't work after installing this driver and `lsusb` shows it is in CD-ROM mode, what should I do?
Install `usb_modeswitch` which can switch your adapter from CD-ROM mode to Wi-Fi mode and then your Wi-Fi adapter should be in Wi-Fi mode after reboot.
