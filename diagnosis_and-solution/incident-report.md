# D-Link DWA-X1850 Driver Incident Report

Date: 2026-09-03

## Summary

The D-Link DWA-X1850 adapter stopped working after the system moved to kernel `7.0.0-28-generic`.

An invalid external DKMS module at `/lib/modules/7.0.0-28-generic/updates/dkms/8852au.ko.zst` took priority over the kernel's valid built-in driver.

Removing the external module allowed the signed built-in `rtw89_8852au` driver to load successfully.

## Impact

The USB adapter remained visible as D-Link device `2001:3321`, but no Wi-Fi driver or network interface was available.

## Diagnosis

The main diagnostic commands were:

```bash
uname -r
lsusb | grep 2001:3321
dkms status
lsmod | grep -E '8852|rtw89'
mokutil --sb-state
mokutil --list-new
modinfo 8852au
modinfo rtw89_8852au
sha256sum /lib/modules/7.0.0-28-generic/updates/dkms/8852au.ko.zst
sha256sum /var/lib/dkms/rtl8852au/1.15.0.1/7.0.0-28-generic/x86_64/module/8852au.ko.zst
zstd -t /lib/modules/7.0.0-28-generic/updates/dkms/8852au.ko.zst
journalctl -k -b | grep -Ei 'secure boot|module verification|rtw89|8852'
```

`uname -r` reported kernel `7.0.0-28-generic`.

`dkms status` showed `rtl8852au/1.15.0.1` installed for both the previous `6.17.0-40-generic` kernel and the current kernel.

For the current kernel, DKMS reported that the built and installed modules differed.

SHA-256 checks confirmed that the installed module and the successful DKMS build artifact were different.

`zstd -t` rejected the installed module with `unsupported format`, while the DKMS build artifact decompressed successfully.

The repository's signing command targeted the compressed `8852au.ko.zst` file.

Kernel modules must be signed while they are uncompressed `.ko` files and compressed afterward.

Therefore, signing the compressed file is the most likely cause of its invalid format.

The invalid external module was located under `updates/dkms`, so it took priority over the built-in kernel module.

`modinfo rtw89_8852au` confirmed that the built-in driver supports USB ID `2001:3321` and is signed with the kernel build key.

`mokutil --sb-state` confirmed that Secure Boot was enabled.

`mokutil --list-new` returned no pending MOK enrollment.

## MOK Password Issue

The MOK enrollment password is the temporary password entered during `mokutil --import`, not the Linux account password.

The failed enrollment did not leave a pending MOK request.

No custom MOK is required when using the signed built-in driver.

## Resolution

The main repair commands were:

```bash
sudo dkms remove -m rtl8852au -v 1.15.0.1 --all
sudo rm -- /lib/modules/7.0.0-28-generic/updates/dkms/8852au.ko.zst
sudo depmod -a
sudo modprobe rtw89_8852au
```

The external `rtl8852au/1.15.0.1` DKMS module was removed from all kernels.

DKMS restored an older invalid copy, so that exact unowned file was removed from `/lib/modules/7.0.0-28-generic/updates/dkms/`.

The module dependency index was regenerated with `depmod -a`.

The built-in driver was loaded with `modprobe rtw89_8852au`.

## Verification

The following modules loaded successfully:

- `rtw89_8852au`
- `rtw89_8852a`
- `rtw89_usb`
- `rtw89_core`

The kernel loaded firmware `rtw89/rtw8852a_fw.bin` and created Wi-Fi interface `wlx0c0e767026a7`.

NetworkManager reported the interface as an available, disconnected Wi-Fi device.

No `rtl8852au` DKMS installation or pending MOK enrollment remained.

## Prevention

Use the kernel's built-in `rtw89_8852au` driver on kernels that support USB ID `2001:3321`.

If an external module is required on another kernel, sign the uncompressed `.ko` file before compressing and installing it.
