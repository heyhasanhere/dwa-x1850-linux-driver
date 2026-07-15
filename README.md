# D-Link DWA-X1850 Linux Driver (rtl8852au, kernel 6.13–6.17+)

Working `rtl8852au` linux driver for the **D-Link DWA-X1850** USB WiFi adapter (USB ID `2001:3321` / `35bc:0100`, Realtek RTL8852AU chipset) on modern Linux kernels.

If you plugged in a DWA-X1850 and got a kernel panic from `rtw89_8852bu`, a `probe failed with error -22` from `mt7921u`, or a build failure against kernel 6.13+ from the stock Realtek/upstream source — this is for you.

## Why this exists

- The official Realtek source from D-Link's support page (`RTL8852AU_WiFi_linux_v1.15.0.1-0-g487ee886.20210714`, dated 2021) does not compile on kernel 6.x.
- The community upstream, [lwfinger/rtl8852au](https://github.com/lwfinger/rtl8852au), fixed the build up to kernel 6.9 (last time I checked ig in early 2026, I dont remember. Anyway).
- Kernel 6.13–6.17 made several out-of-tree-driver-breaking API changes (`ccflags-y` replacing `EXTRA_CFLAGS` handling, `MODULE_IMPORT_NS` string argument, timer API renames, new `cfg80211_ops` parameters). This repo carries the patches for those.
- The in-tree `rtw89_8852bu` and `mt7921u` drivers will bind to this device and either kernel-panic or fail to probe — they must be blacklisted.

## Credits / provenance

1. **Realtek** — original vendor source (2021).
2. **[Larry Finger (`lwfinger`)](https://github.com/lwfinger/rtl8852au)** — maintains the community upstream, all driver logic and design credit goes here.
3. An intermediate fork (`psyiode/rtl8852a_ubuntu_6.17.0-22-generic_driver`) added the kernel 6.13–6.17 build-compat patches. **I dont know why but that repo has since been deleted/is no longer reachable on GitHub**

No functional changes were made to the driver anywhere in this chain — every patch here is a version-guarded build-compat fix (`#if LINUX_VERSION_CODE>= KERNEL_VERSION(...)`), so on older kernels the code compiles to the original upstream path unchanged.

## Supported chipsets/devices (from upstream):

* BUFFALO WI-U3-1200AX2(/N) — `0411:0312`
* ASUS USB-AX56 — `0b05:1997`, `0b05:1a62`
* EDUP EP-AX1696GS — `0bda:8832`
* Fenvi FU-AX1800P — `0bda:885c`
* Realtek Demo Board — `0bda:8832`, `0bda:885a`, `0bda:885c`
* **D-Link DWA-X1850 — `2001:3321`, `35bc:0100`**
* TP-Link AX1800 — `2357:013f`, `2357:0141`
* ipTIME AX2000U — `0bda:8832`
* ELECOM WDC-X1201DU3 — `056e:4020`

## Prerequisites

```bash
sudo apt install build-essential dkms linux-headers-$(uname -r) git
```

## DWA-X1850 quirk: it enumerates as a USB disk first

The DWA-X1850 ships with a mode that first enumerates as a USB mass-storage device carrying a Windows driver. If `lsusb` shows ID `0bda:1a2b`, that disk mode is active. To force it into WiFi-adapter mode automatically, add a `usb_modeswitch` rule. Edit whichever of these exists on your system:

- `/usr/lib/udev/rules.d/40-usb_modeswitch.rules`
- `/lib/udev/rules.d/40-usb_modeswitch.rules`

and add:

```
# D-Link DWA-X1850 Wifi Dongle
ATTR{idVendor}=="0bda", ATTR{idProduct}=="1a2b", RUN+="usb_modeswitch '/%k'"
```

## Install

```bash
git clone https://github.com/heyhasanhere/dwa-x1850-linux-driver.git rtl8852au
cd rtl8852au
make
ls 8852au.ko        # must exist before continuing
```

### Install via DKMS (recommended)

DKMS rebuilds the module automatically on every kernel update, so you don't need to repeat these steps after a kernel upgrade — DKMS does it for you.

```bash
sudo cp -r . /usr/src/rtl8852au-1.15.0.1
sudo dkms add -m rtl8852au -v 1.15.0.1
sudo dkms build -m rtl8852au -v 1.15.0.1
sudo dkms install --force -m rtl8852au -v 1.15.0.1
```

Confirm:

```bash
ls /lib/modules/$(uname -r)/updates/dkms/8852au.ko.zst   # or 8852au.ko
```

## Secure Boot: signing the module

Skip this section if Secure Boot is disabled (`mokutil --sb-state`).

### 1. Generate a Machine Owner Key (MOK)

```bash
sudo openssl req -new -x509 -newkey rsa:2048 \
  -keyout /var/lib/shim-signed/mok/MOK.priv \
  -out    /var/lib/shim-signed/mok/MOK.pem \
  -days 36500 -subj "/CN=Custom Driver Signing Key/" -nodes

sudo openssl x509 \
  -in  /var/lib/shim-signed/mok/MOK.pem \
  -outform DER \
  -out /var/lib/shim-signed/mok/MOK.der
```

### 2. Sign the installed module

```bash
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 \
  /var/lib/shim-signed/mok/MOK.priv \
  /var/lib/shim-signed/mok/MOK.pem \
  /lib/modules/$(uname -r)/updates/dkms/8852au.ko.zst
```

### 3. Queue MOK enrollment

```bash
HASH=$(mokutil --generate-hash=dwa1850)
echo "$HASH" | sudo tee /tmp/mokhash.txt > /dev/null
sudo mokutil --import /var/lib/shim-signed/mok/MOK.der --hash-file /tmp/mokhash.txt
sudo rm /tmp/mokhash.txt

# Verify key is pending:
sudo mokutil --list-new | grep CN=
```

### 4. Reboot and enroll

A blue **"Perform MOK management"** screen appears once on next reboot:

1. **Enroll MOK** → Continue → Yes
2. Enter the password you set above
3. Reboot

If the screen doesn't appear, the `mokutil --import` step didn't save — repeat step 3 and reboot again.

Alternatively, `make sign-install` in the driver directory drives an interactive version of the same flow (you'll be prompted for a password during build, then asked to enroll it at the next boot). If you mistype the password during enrollment, your machine won't boot until you run `sudo mokutil --reset`, reboot, select "Reset MOK list" at the MOK screen, reboot again, and retry.

## Auto-load on boot + blacklist conflicting drivers

```bash
echo "8852au" | sudo tee /etc/modules-load.d/8852au.conf

# These will kernel-panic or fail to probe if they bind to the device first:
echo "blacklist rtw89_8852au" | sudo tee -a /etc/modprobe.d/blacklist.conf
echo "blacklist rtw89_8852bu" | sudo tee -a /etc/modprobe.d/blacklist.conf
```

## Load and verify

```bash
sudo modprobe 8852au
ip link show          # expect a wlan0 / wlx... interface
dmesg | grep 8852     # expect probe success messages
```

## Updating after a kernel upgrade

DKMS rebuilds automatically. If a future kernel breaks the build again
(new kernel API change not yet patched here), the module will fail to
compile and you'll have no WiFi until new patches land. To rebuild manually
once patched:

```bash
sudo dkms build -m rtl8852au -v 1.15.0.1 -k $(uname -r)
sudo dkms install --force -m rtl8852au -v 1.15.0.1 -k $(uname -r)
# Re-sign if Secure Boot is enabled (see above)
sudo modprobe 8852au
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Key was rejected by service` | Secure Boot, module not signed | Complete the Secure Boot signing steps |
| MOK screen never appears at boot | `mokutil --import` failed silently | Repeat the import step, reboot again |
| `drv_types.h: No such file or directory` | `EXTRA_CFLAGS` ignored by kernel 6.15+ kbuild | Already patched here (`ccflags-y`); if building from vanilla Realtek source, see Manual Patches below |
| Kernel panic / oops on load | Wrong driver bound (`rtw89_8852bu`) | Blacklist it and use `8852au` |
| `probe failed with error -22` | `mt7921u` bound to device | Wrong driver; chip is RTL8852AU, not MT7921 |
| Module missing after kernel update | DKMS build failed | Check `dkms status`, rebuild manually |

## Manual patches (kernel 6.13+ compatibility)

Only needed if you're patching the vanilla Realtek/upstream source yourself instead of using this repo directly.

**1. `Makefile` — `EXTRA_CFLAGS` dropped by kbuild on kernel 6.15+**

```bash
sed -i 's/EXTRA_CFLAGS/ccflags-y/g' Makefile common.mk phl/phl.mk \
  phl/hal_g6/rtl8852a/rtl8852a.mk platform/i386_pc.mk
```

**2. `include/osdep_service_linux.h` — timer API renamed in kernel 6.15/6.16**

```c
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(6, 16, 0))
	_timer *ptimer = timer_container_of(ptimer, in_timer, timer);
#else
	_timer *ptimer = from_timer(ptimer, in_timer, timer);
#endif
```

and

```c
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(6, 15, 0))
	*bcancelled = timer_delete_sync(&ptimer->timer) == 1 ? 1 : 0;
#else
	*bcancelled = del_timer_sync(&ptimer->timer) == 1 ? 1 : 0;
#endif
```

(and the equivalent for `del_timer` / `timer_delete`.)

**3. `core/rtw_mem.c`, `os_dep/osdep_service_linux.c` — `MODULE_IMPORT_NS` takes a string in kernel 6.13+**

```c
#if defined(MODULE_IMPORT_NS)
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(6, 13, 0))
	MODULE_IMPORT_NS("VFS_internal_I_am_really_a_filesystem_and_am_NOT_a_driver");
#else
	MODULE_IMPORT_NS(VFS_internal_I_am_really_a_filesystem_and_am_NOT_a_driver);
#endif
#endif
```

**4. `os_dep/linux/ioctl_cfg80211.c` — new `cfg80211_ops` parameters in kernel 6.14/6.16**

- `set_monitor_channel` gains a `struct net_device *dev` parameter on kernel ≥ 6.14.
- `set_wiphy_params` and `set_tx_power` gain an `int radio_idx` parameter on kernel ≥ 6.16.
- `get_tx_power` gains `int radio_idx` and `unsigned int link_id` on kernel ≥ 6.16.

Guard each new parameter with `#if (LINUX_VERSION_CODE >= KERNEL_VERSION(6, 14, 0))` /
`(6, 16, 0)` as appropriate — see `os_dep/linux/ioctl_cfg80211.c` in this repo
for the exact diff against upstream.
