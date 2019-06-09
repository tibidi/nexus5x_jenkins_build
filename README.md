# nexus5x_jenkins_build
Jenkins pipeline for building nexus pixel 5x.

Links:
  - https://github.com/PixelExperience/manifest (PixelExperience repo)
  - https://github.com/BlissRoms/platform_manifest (How to build on Ubuntu)
  - https://github.com/USBhost/build-tools-gcc (To build the kernel toolchain)
  - the local.xml contains nexus 5x specific repos
  
Pathes again sources:
```
diff --git a/BoardConfig.mk b/BoardConfig.mk
index f892837..f580ece 100644
--- a/BoardConfig.mk
+++ b/BoardConfig.mk
@@ -34,7 +34,8 @@ WITH_DEXPREOPT := true
 DONT_DEXPREOPT_PREBUILTS := true

 # Inline kernel building
-KERNEL_TOOLCHAIN := $(ANDROID_BUILD_TOP)/prebuilts/gcc_toolchain_10/bin/
+# KERNEL_TOOLCHAIN := $(ANDROID_BUILD_TOP)/prebuilts/gcc_toolchain_10/bin/
+KERNEL_TOOLCHAIN := /opt/toolchain/prebuilts/gcc_toolchain_10/bin
 KERNEL_TOOLCHAIN_PREFIX := aarch64-linux-gnu-
 TARGET_KERNEL_SOURCE := kernel/lge/bullhead
 TARGET_KERNEL_CONFIG := flipflop_defconfig
diff --git a/aosp_bullhead.mk b/aosp_bullhead.mk
old mode 100644
new mode 100755
index c526139..db82663
--- a/aosp_bullhead.mk
+++ b/aosp_bullhead.mk
@@ -20,6 +20,8 @@
 # Get the long list of APNs
 PRODUCT_COPY_FILES := device/lge/bullhead/apns-full-conf.xml:system/etc/apns-conf.xml

+PRODUCT_PROPERTY_OVERRIDES = ro.config.low_ram=false
+
 # Inherit some common PixelExperience stuff.
 TARGET_BOOT_ANIMATION_RES := 1080
 TARGET_GAPPS_ARCH := arm64
```
 
