# aic8800d80-openwrt

OpenWrt packages for the out-of-tree **AIC8800D80** USB Wi-Fi driver
([shenmintao/aic8800d80](https://github.com/shenmintao/aic8800d80)), built by
GitHub Actions.

Supports **OpenWrt 25.12.x · ath79/generic (mips_24kc)**, e.g. the GL.iNet 6416
and other ath79 routers with a USB port. A separate build is produced for every
25.12.x point release, because kernel modules only load on the exact kernel
they were compiled against.

**Prebuilt packages:**
[**Releases**](https://github.com/hillz2/aic8800d80-openwrt/releases) — download
the artifacts for *your* release (see the table in
[Step 0](#step-0--check-that-your-router-matches)).

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

One job is built per OpenWrt 25.12.x release (25.12.0 … 25.12.6), because a
kernel module is locked to the exact kernel build it was compiled against:

| OpenWrt | kernel | mac80211 backports |
|---|---|---|
| 25.12.0 | 6.12.71 | 6.18.7 |
| 25.12.1 | 6.12.74 | 6.18.7 |
| 25.12.2 | 6.12.74 | 6.18.7 |
| 25.12.3 | 6.12.85 | 6.18.7 |
| 25.12.4 | 6.12.87 | 6.18.26 |
| 25.12.5 | 6.12.94 | 6.18.26 |
| 25.12.6 | 6.12.112 | 6.18.55 |

To build only some of them, run the workflow manually and set `releases`, e.g.
`25.12.2 25.12.6`.

The build uses the official OpenWrt SDK for the target, so the module vermagic
matches that release's kernel. The SDK ships no wireless stack at all — no
`package/kernel/mac80211`, no staged mac80211 headers and no prebuilt
`kmod-cfg80211`/`kmod-mac80211` packages — so the workflow checks out
`package/kernel/mac80211` from the matching `v<release>` tag, builds it, and
compiles the driver against those headers. `AIC_CFG80211_VERSION` is read from
that mac80211 `Makefile`, so the driver's cfg80211 API gates always match the
backports version it links against.

## Install on OpenWrt 25.12 (tutorial)

Prebuilt packages are attached to
[**Releases**](https://github.com/hillz2/aic8800d80-openwrt/releases). Pick the
one built for your OpenWrt release — the kernel version must match exactly.

### Step 0 — Check that your router matches

Kernel modules only load on the exact kernel they were built against, so the
packages are specific to both a target **and** an OpenWrt point release.
Verify before downloading anything:

```sh
. /etc/openwrt_release
echo "$DISTRIB_RELEASE / $DISTRIB_TARGET"     # e.g. 25.12.2 / ath79/generic
uname -r                                      # e.g. 6.12.74
```

Then pick the row that matches `$DISTRIB_RELEASE`:

| OpenWrt | `uname -r` |
|---|---|
| 25.12.0 | 6.12.71 |
| 25.12.1 | 6.12.74 |
| 25.12.2 | 6.12.74 |
| 25.12.3 | 6.12.85 |
| 25.12.4 | 6.12.87 |
| 25.12.5 | 6.12.94 |
| 25.12.6 | 6.12.112 |

`DISTRIB_TARGET` must be `ath79/generic`. If your release is not in the table,
or `DISTRIB_TARGET` differs, no package here will install — see *Step 0b*.

#### Step 0b — If your release is not covered

There is no single build that works on every OpenWrt 25.x: the kernel rejects a
module whose version string differs. Two options:

- **Upgrade to a covered release.** For 24.10 and older, this is required — the
  packages do not exist for kernel 6.6. Download the `*-sysupgrade.bin` for your
  model from
  <https://downloads.openwrt.org/releases/25.12.6/targets/ath79/generic/>
  (for the GL.iNet 6416: `openwrt-25.12.6-ath79-generic-glinet_6416-squashfs-sysupgrade.bin`),
  back up with **System → Backup / Flash Firmware → Generate archive**, flash
  with **Flash image** and **untick "Keep settings"** (the `opkg` → `apk`
  switch makes keeping 24.10 config unsupported), then re-run Step 0.
- **Build it yourself** for your exact release with
  **Actions → build → Run workflow** and `releases` set to your version (see
  [Build](#build)).

### Step 1 — Download the packages

The filenames embed the kernel version, so they differ per release. Use the
release matching your Step 0 row. For 25.12.2 / kernel 6.12.74:

```sh
cd /tmp
BASE=https://github.com/hillz2/aic8800d80-openwrt/releases/download/<tag>
for f in \
  aic8800d80-firmware-2026.11.20.1a2ce4e3-r1.apk \
  kmod-aic8800d80-6.12.74.2026.11.20.1a2ce4e3-r1.apk \
  kmod-cfg80211-6.12.74.6.18.7-r1.apk \
  kmod-mac80211-6.12.74.6.18.7-r1.apk \
  kmod-usb-core-6.12.74-r1.apk
do
  wget "$BASE/$f" || echo "FAILED: $f"
done
ls -1 *.apk
```

The exact names are listed on the release page; copy them from there rather
than retyping, since the kernel and backports versions both appear in them.
Simplest is to download all five from the release page in a browser and
`scp` them:

```sh
scp *.apk root@192.168.1.1:/tmp/
```

> **Note the dots.** The build produces `2026.11.20~1a2ce4e3` (with a `~`), but
> GitHub rewrites `~` to `.` in release asset names, so the released files use
> `2026.11.20.1a2ce4e3`. That is only a filename difference; the package version
> recorded inside the `.apk` is unchanged. Always install with a glob
> (`kmod-aic8800d80-*.apk`) rather than typing the version by hand.

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
`aic8800d80-firmware`). If you see a dependency error naming a `kernel=` version
other than your `uname -r`, you downloaded the wrong release's packages — go
back to Step 0.

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
  25.12.x `ath79/generic`; installing them on a different point release or
  custom build will fail the vermagic check. Pick the matching release, or
  rebuild for it.
- The Bluetooth ZLP quirk (`aic_zlp_quirk`) needs `CONFIG_KPROBES`, which stock
  OpenWrt 25.12 does not enable. It is not auto-loaded; Wi-Fi is unaffected.
- If the driver log reports `chip_mcu_id=1`, upstream requires the
  `legacy-mcu1` branch. Run **Actions → build → Run workflow** with
  `driver_ref: legacy-mcu1`, or change `PKG_SOURCE_VERSION` (and
  `PKG_SOURCE_DATE`) in both `package/aic8800d80/Makefile` and
  `package/aic8800d80-firmware/Makefile` to a commit from that branch — the
  matched firmware is part of that branch.

## Adding more targets

Packages are built per target/subtarget, not per device. The matrix is
generated by the `prepare` job from the `releases` input, and currently assumes
`ath79/generic` for every release. To add another target, extend the `entries=`
loop in `.github/workflows/build.yml` with a second entry per release, e.g.:

```sh
entries="$entries{\"name\":\"ramips-mt7621-$v\",\"version\":\"$v\",\"target\":\"ramips\",\"subtarget\":\"mt7621\"},"
```

To support a different OpenWrt release, add it to `ALL` in the same job (and to
the tables in this README). `AIC_CFG80211_VERSION` is read from that release's
`package/kernel/mac80211` automatically, so no other edit is needed.

Keep the target kernel and the backports version on the same major line: a
kernel older than the backports means the driver's `LINUX_VERSION_CODE` gates
will disagree with the cfg80211 ABI it links against.

## License

Package metadata and build files: GPL-2.0-only. The driver source is
GPL-2.0-only; see upstream.
