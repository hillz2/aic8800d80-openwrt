# aic8800d80-openwrt

OpenWrt packages for the out-of-tree **AIC8800D80** USB Wi-Fi driver
([shenmintao/aic8800d80](https://github.com/shenmintao/aic8800d80)), built by
GitHub Actions.

Currently targets **OpenWrt 25.12.6 · ath79/generic (mips_24kc)**, e.g. the
GL.iNet 6416 and other ath79 routers with a USB port.

**Prebuilt packages:**
[Releases → `v25.12.6-ath79-generic`](https://github.com/hillz2/aic8800d80-openwrt/releases/tag/v25.12.6-ath79-generic)
— see [Install on OpenWrt 25.12](#install-on-openwrt-2512-tutorial).

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

## Install on OpenWrt 25.12 (tutorial)

Prebuilt packages are attached to
[**Releases**](https://github.com/hillz2/aic8800d80-openwrt/releases). The
example below uses the `v25.12.6-ath79-generic` release.

### Step 0 — Check that your router matches

The packages are built for one specific target **and** one specific kernel
build. Verify before downloading anything:

```sh
. /etc/openwrt_release
echo "$DISTRIB_RELEASE / $DISTRIB_TARGET"     # expect 25.12.6 / ath79/generic
uname -r                                      # expect 6.12.112
```

Both lines must match the release name (`v25.12.6-ath79-generic`, kernel
`6.12.112`). If `DISTRIB_RELEASE` is `24.10.x` or anything else, the modules
will be rejected by the kernel — see *Step 0b*.

#### Step 0b — If you are still on 24.10

These packages will **not** install on 24.10.x: it runs kernel 6.6, and these
modules are vermagic-locked to 6.12.112. Flash the official 25.12.6 image for
your device first:

1. Download from
   <https://downloads.openwrt.org/releases/25.12.6/targets/ath79/generic/>
   the `*-sysupgrade.bin` for your model (e.g. `glinet_gl-ar750`-style naming —
   for the GL.iNet 6416 look for `glinet_gl-*6416*`).
2. Back up your config: **System → Backup / Flash Firmware → Generate archive**.
3. Flash it via **System → Backup / Flash Firmware → Flash image**, and
   **untick "Keep settings"** — 24.10 → 25.12 is a major upgrade and keeping
   old config across the `opkg` → `apk` switch is not supported.
4. Reconnect and re-run the checks from Step 0.

### Step 1 — Download the packages

Copy all five `.apk` files to the router, e.g. from your PC:

```sh
scp *.apk root@192.168.1.1:/tmp/
```

Or download them directly on the router:

```sh
cd /tmp
BASE=https://github.com/hillz2/aic8800d80-openwrt/releases/download/v25.12.6-ath79-generic
wget -O - "$BASE" 2>/dev/null | grep -o 'href="[^"]*\.apk"' | cut -d'"' -f2 | while read -r f; do
  wget "$BASE/$(basename "$f")"
done
ls -1 *.apk
```

You should see:

```
aic8800d80-firmware-2026.11.20~1a2ce4e3-r1.apk
kmod-aic8800d80-6.12.112.2026.11.20~1a2ce4e3-r1.apk
kmod-cfg80211-6.12.112.6.18.55-r2.apk
kmod-mac80211-6.12.112.6.18.55-r2.apk
kmod-usb-core-6.12.112-r1.apk
```

> `apk` on OpenWrt ignores `~` and `.` differently from the filename above —
> always install with a glob (`kmod-aic8800d80-*.apk`) rather than typing the
> full name.

### Step 2 — Decide whether you need the wireless stack packages

Stock OpenWrt 25.12 `ath79/generic` images normally ship `kmod-cfg80211` and
`kmod-mac80211` already. Check:

```sh
apk list --installed 2>/dev/null | grep -E 'kmod-(cfg80211|mac80211|usb-core)'
```

- **All three listed** → skip to *Step 3*.
- **Any missing** → install them from this release first, so the versions match
  the driver build exactly:

  ```sh
  apk add --allow-untrusted \
    /tmp/kmod-cfg80211-*.apk /tmp/kmod-mac80211-*.apk /tmp/kmod-usb-core-*.apk
  ```

  If `apk` refuses because it wants to *downgrade* or conflicts, your image has
  a different backports build; build the packages against your own tree rather
  than mixing (see *Build*).

### Step 3 — Install firmware and driver

```sh
apk add --allow-untrusted /tmp/aic8800d80-firmware-*.apk /tmp/kmod-aic8800d80-*.apk
```

`apk` resolves the rest of the dependencies from the package metadata
(`kernel`, `kmod-cfg80211`, `kmod-mac80211`, `kmod-usb-core`,
`aic8800d80-firmware`). If you see a dependency error naming `kernel=6.12.112`,
your kernel does not match — go back to Step 0.

### Step 4 — Reboot and verify

```sh
reboot
```

After it comes back, plug in the adapter and check:

```sh
dmesg | grep -i -E 'aic|usb 1-1'    # firmware download + "aicwf_usb_probe"
lsmod | grep aic                   # aic8800_fdrv should be listed
lsusb                              # 368b:8d81 / a69c:8d81 (or your adapter's ID)
ip link                            # a new wlanN interface
iw dev
```

Expected `dmesg` highlights:

```
aicwf_usb_probe
aicwf_usb_download_fw ... (firmware from /lib/firmware/aic8800D80/)
```

Then bring the interface up and scan:

```sh
ip link set wlan0 up
iw dev wlan0 scan | grep SSID | head
```

### Step 5 — Join a network (LuCI or CLI)

**LuCI:** *Network → Wireless* → the new radio appears → **Scan** → join.

**CLI (WPA2):**

```sh
cat >/etc/config/wireless <<'EOF'
config wifi-device 'radio0'
	option type 'mac80211'
	option path 'platform/usb'
	option channel 'auto'
	option hwmode '11a'
	option htmode 'VHT80'

config wifi-iface 'default_radio0'
	option device 'radio0'
	option network 'lan'
	option mode 'sta'
	option ssid 'MyNetwork'
	option encryption 'psk2'
	option key 'MyPassword'
EOF
wifi reload
```

Adjust `option path` to the value `iw dev`/`dmesg` reports for your adapter,
and `hwmode`/`htmode` to your band (the AIC8800 is dual-band).

### Uninstall

```sh
apk del kmod-aic8800d80 aic8800d80-firmware
```

Remove `kmod-cfg80211`/`kmod-mac80211` only if you installed them here and
nothing else needs them.

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
