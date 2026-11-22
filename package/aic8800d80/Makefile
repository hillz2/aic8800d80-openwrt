# SPDX-License-Identifier: GPL-2.0-only
#
# OpenWrt package for the out-of-tree AIC8800D80 USB Wi-Fi driver.
# Upstream: https://github.com/shenmintao/aic8800d80
#
# The driver is compiled against the mac80211 backports shipped by the target
# release, not against the in-kernel wireless stack. Keep AIC_CFG80211_VERSION
# in sync with package/kernel/mac80211's PKG_VERSION for that release.

include $(TOPDIR)/rules.mk

PKG_NAME:=aic8800d80
PKG_RELEASE:=1

PKG_LICENSE:=GPL-2.0-only
PKG_LICENSE_FILES:=LICENSE

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/shenmintao/aic8800d80.git
PKG_SOURCE_VERSION:=1a2ce4e38e678a0c0d7f6071be415ab74f488240
PKG_SOURCE_DATE:=2026-11-20

PKG_BUILD_PARALLEL:=1

# OpenWrt 24.10.3 ships mac80211 backports 6.12.44 on Linux 6.6.
AIC_CFG80211_VERSION:=6.12.44

# The mac80211 headers and Module.symvers must be staged before this package
# is configured or compiled.
STAMP_CONFIGURED_DEPENDS := $(STAGING_DIR)/usr/include/mac80211-backport/backport/autoconf.h

include $(INCLUDE_DIR)/kernel.mk
include $(INCLUDE_DIR)/package.mk

define KernelPackage/aic8800d80
  SUBMENU:=Wireless Drivers
  TITLE:=AIC8800D80 USB Wi-Fi driver
  DESCRIPTION:=Out-of-tree driver for AIC8800D80 based USB Wi-Fi adapters \
	(Tenda U11, AX913B and compatible AIC8800 boards).
  DEPENDS:=+kmod-cfg80211 +kmod-mac80211 +kmod-usb-core +aic8800d80-firmware
  FILES:= \
	$(PKG_BUILD_DIR)/drivers/aic8800/aic_load_fw/aic_load_fw.ko \
	$(PKG_BUILD_DIR)/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.ko \
	$(PKG_BUILD_DIR)/drivers/aic8800/aic_zlp_quirk/aic_zlp_quirk.ko
  # aic8800_fdrv depends on aic_load_fw, so load the loader first.
  AUTOLOAD:=$(call AutoProbe,aic_load_fw aic8800_fdrv)
endef

define Package/aic8800d80-firmware
  SECTION:=firmware
  CATEGORY:=Firmware
  TITLE:=AIC8800 firmware
  PKGARCH:=all
endef

define Package/aic8800d80-firmware/description
 Firmware blobs for the AIC8800 family (aic8800, aic8800D80, aic8800D80N,
 aic8800D80X2, aic8800DC, aic8800DLN) used by kmod-aic8800d80.
endef

define Package/aic8800d80-firmware/install
	$(INSTALL_DIR) $(1)/lib/firmware
	$(CP) $(PKG_BUILD_DIR)/fw/aic8800* $(1)/lib/firmware/
endef

NOSTDINC_FLAGS := \
	$(KERNEL_NOSTDINC_FLAGS) \
	-I$(STAGING_DIR)/usr/include/mac80211-backport/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211-backport \
	-I$(STAGING_DIR)/usr/include/mac80211/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211 \
	-include backport/backport.h

define Build/Compile
	+$(KERNEL_MAKE) $(PKG_JOBS) \
		M="$(PKG_BUILD_DIR)/drivers/aic8800" \
		NOSTDINC_FLAGS="$(NOSTDINC_FLAGS)" \
		KBUILD_EXTRA_SYMBOLS="$(wildcard $(PKG_SYMVERS_DIR)/*.symvers)" \
		CFG80211_VERSION=$(AIC_CFG80211_VERSION) \
		modules
endef

$(eval $(call KernelPackage,aic8800d80))
$(eval $(call BuildPackage,aic8800d80-firmware))