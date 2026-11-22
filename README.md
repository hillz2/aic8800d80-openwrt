# aic8800d80-openwrt

OpenWrt packages for the out-of-tree **AIC8800D80** USB Wi-Fi driver
([shenmintao/aic8800d80](https://github.com/shenmintao/aic8800d80)), built by
GitHub Actions.

Currently targets **OpenWrt 25.12.6 · ath79/generic (mips_24kc)**, e.g. the
GL.iNet 6416 and other ath79 routers with a USB port.

> **Why 25.12 and not 24.10?** The driver picks cfg80211 APIs by comparing
> `LINUX_VERSION_CODE` against the kernel it is built for, but it links
> against the *backports* wireless stack, whose version differs from the
> kernel. On 24.10 the kernel is 6.6 while cfg80211 is 6.12, and 60+ call
> sites silently select the wrong ABI (`cfg80211_ch_switch_notify`,
> `change_beacon`, ...). On 25.12 the kernel is 6.12, so the gates line up
> with no patching. This is the combination upstream tests in `OPENWRT.md`.

## Packages

| Package | Contents |
|---|---|
| `kmod-aic8800d80` | `aic8800_fdrv`, `aic_load_fw`, `aic_zlp_quirk` kernel modules |
| `aic8800d80-firmware` | all `aic8800*` firmware from the upstream `fw/` tree |

`kmod-aic8800d80` depends on `kmod-cfg80211`, `kmod-mac80211`,
`kmod-usb-core` and the firmware package.

## Build

Pushing to any branch, opening a PR, or running **Actions → build → Run
workflow** builds the packages. The packages are uploaded as workflow
artifacts. Pushing a tag additionally attaches them to a GitHub Release.

The build uses the official OpenWrt SDK for the target, so the module vermagic
matches the release kernel (6.12 for 25.12.6). The SDK ships no wireless stack
at all — no `package/kernel/mac80211`, no staged mac80211 headers and no
prebuilt `kmod-cfg80211`/`kmod-mac80211` packages — so the workflow checks out
`package/kernel/mac80211` from the matching `v25.12.6` tag, builds it, and then
compiles the driver against those headers with `CFG80211_VERSION=6.18.55` (the
backports version in 25.12.6).

## Install on the router

OpenWrt 25.12 uses `apk`, so the artifacts are `.apk` packages. Download them
from the workflow artifact (or release). The artifact also contains the
matching `kmod-cfg80211`, `kmod-mac80211` and `kmod-usb-core` builds, which are
useful if your router image lacks them:

```sh
apk update
apk add --allow-untrusted /tmp/kmod-cfg80211-*.apk /tmp/kmod-mac80211-*.apk /tmp/kmod-usb-core-*.apk
apk add --allow-untrusted /tmp/aic8800d80-firmware-*.apk /tmp/kmod-aic8800d80-*.apk
reboot
```

If your image already has the wireless stack, install only the driver and
firmware:

```sh
apk add --allow-untrusted /tmp/aic8800d80-firmware-*.apk /tmp/kmod-aic8800d80-*.apk
```

Plug the adapter in and check:

```sh
lsusb                       # 368b:8d81 / a69c:8d81 (or the adapter's ID)
dmesg | grep -i aic
iw dev
```

### Requirements / notes

- The USB host controller modules for your router (e.g. `kmod-usb-ohci`,
  `kmod-usb-ehci`) must already be installed.
- Packages are tied to the **exact kernel build**. They are built for OpenWrt
  25.12.6 `ath79/generic`; installing them on a different release or custom
  build will fail the vermagic check. Rebuild for the new release instead.
- The Bluetooth ZLP quirk (`aic_zlp_quirk`) needs `CONFIG_KPROBES`, which stock
  OpenWrt 25.12 does not enable. It is not auto-loaded; Wi-Fi is unaffected.
- If the driver log reports `chip_mcu_id=1`, upstream requires the
  `legacy-mcu1` branch. Run **Actions → build → Run workflow** with
  `driver_ref: legacy-mcu1`, or change `PKG_SOURCE_VERSION` (and
  `PKG_SOURCE_DATE`) in both `package/aic8800d80/Makefile` and
  `package/aic8800d80-firmware/Makefile` to a commit from that branch — the
  matched firmware is part of that branch.

## Adding more targets

Packages are built per target/subtarget, not per device. Add an entry to the
matrix in `.github/workflows/build.yml`:

```yaml
matrix:
  include:
    - name: ath79-generic
      target: ath79
      subtarget: generic
    - name: ramips-mt7621
      target: ramips
      subtarget: mt7621
```

For a different OpenWrt release, update `OPENWRT_VERSION` and
`AIC_CFG80211_VERSION` (from that release's `package/kernel/mac80211`). Keep
the target kernel and the backports version on the same major line if you can:
a kernel older than the backports means the driver's `LINUX_VERSION_CODE`
gates will disagree with the cfg80211 ABI it links against.

## License

Package metadata and build files: GPL-2.0-only. The driver source is
GPL-2.0-only; see upstream.
