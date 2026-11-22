# SPDX-License-Identifier: GPL-2.0-only
#
# Firmware blobs for the AIC8800 family, used by kmod-aic8800d80.
#
# Kept in its own package directory (rather than sharing the driver's
# Makefile) because OpenWrt builds one source tree per Makefile: a
# KernelPackage and a BuildPackage in the same file cannot both get build
# targets. Firmware is also architecture independent, so it is PKGARCH:=all.

include $(TOPDIR)/rules.mk

PKG_NAME:=aic8800d80-firmware
PKG_RELEASE:=1

PKG_LICENSE:=GPL-2.0-only
PKG_LICENSE_FILES:=LICENSE

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/shenmintao/aic8800d80.git
PKG_SOURCE_VERSION:=1a2ce4e38e678a0c0d7f6071be415ab74f488240
PKG_SOURCE_DATE:=2026-11-20

include $(INCLUDE_DIR)/package.mk

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

$(eval $(call BuildPackage,aic8800d80-firmware))