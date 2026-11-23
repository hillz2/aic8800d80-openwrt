# SPDX-License-Identifier: GPL-2.0-only
#
# OpenWrt package for the out-of-tree AIC8800D80 USB Wi-Fi driver.
# Upstream: https://github.com/shenmintao/aic8800d80
#
# The driver is compiled against the mac80211 backports shipped by the target
# release, not against the in-kernel wireless stack. The backports version
# differs between point releases (6.18.7 on 25.12.0-25.12.3, 6.18.26 on
# 25.12.4/25.12.5, 6.18.55 on 25.12.6), so it is read from the mac80211
# package present in the tree instead of being hardcoded.

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

# mac80211 PKG_VERSION for whatever OpenWrt release this tree is built from.
# CI places package/kernel/mac80211 in the SDK before configuration, so this
# resolves for every supported 25.12.x release without editing the file.
AIC_CFG80211_VERSION:=$(shell sed -n 's/^PKG_VERSION:=//p' $(TOPDIR)/package/kernel/mac80211/Makefile 2>/dev/null)
ifeq ($(strip $(AIC_CFG80211_VERSION)),)
  $(error mac80211 PKG_VERSION not found in $(TOPDIR)/package/kernel/mac80211/Makefile; the mac80211 package must be in the tree before this package is configured)
endif

# The mac80211 headers and Module.symvers must be staged before this package
# is configured or compiled.
STAMP_CONFIGURED_DEPENDS := $(STAGING_DIR)/usr/include/mac80211-backport/backport/autoconf.h

include $(INCLUDE_DIR)/kernel.mk
include $(INCLUDE_DIR)/package.mk

define KernelPackage/aic8800d80
  SUBMENU:=Wireless Drivers
  TITLE:=AIC8800D80 USB Wi-Fi driver
  DEPENDS:=+kmod-cfg80211 +kmod-mac80211 +kmod-usb-core +aic8800d80-firmware
  FILES:= \
	$(PKG_BUILD_DIR)/drivers/aic8800/aic_load_fw/aic_load_fw.ko \
	$(PKG_BUILD_DIR)/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.ko \
	$(PKG_BUILD_DIR)/drivers/aic8800/aic_zlp_quirk/aic_zlp_quirk.ko
  # aic8800_fdrv depends on aic_load_fw, so load the loader first.
  AUTOLOAD:=$(call AutoProbe,aic_load_fw aic8800_fdrv)
endef

define KernelPackage/aic8800d80/description
 Out-of-tree driver for AIC8800D80 based USB Wi-Fi adapters
 (Tenda U11, AX913B and compatible AIC8800 boards).
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