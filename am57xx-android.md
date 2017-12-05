# Bringing up Android Oreo on X15/AM57xx
## Building Android/Linux kernel/U-boot

### TI linux kernel v4.4

```bash
$ sudo apt-get install liblz4-dev liblz4-tool
$ git clone --depth 1 -b p-ti-android-linux-4.4.y git://git.ti.com/android/kernel.git ti-kernel.git
$ export PATH=/media/xdev/work/reps/android.repo/prebuilts/gcc/linux-x86/arm/arm-linux-androideabi-4.9/bin:$PATH
$ export CROSS_COMPILE=arm-linux-androideabi-
$ export ARCH=arm
$ cd ti-kernel.git
$ ./ti_config_fragments/defconfig_builder.sh -t ti_sdk_am57x_android_release
$ make ti_sdk_am57x_android_release_defconfig
$ make -jn zImage am57xx-evm-reva3.dtb am57xx-beagle-x15.dtb
```

### Android

```bash
$ repo init -u http://git.ti.com/git/android/manifest.git -b oreo-dev
$ repo sync -j20
$ export KERNELDIR=${KERNEL_PATH}
$ source build/envsetup.sh
$ lunch full_am57xevm-userdebug
$ make -j8
```

### U-Boot

It's mandatory to use Linux arm-eabi GCC 6 toolchain:
```bash
$ git clone git://git.denx.de/u-boot.git u-boot.git
$ export PATH=${COMPLIER_DIR}/gcc-linaro-6.3.1-2017.05-x86_64_arm-eabi/bin:$PATH
$ export CROSS_COMPILE=arm-eabi-
$ export ARCH=arm
$ make am57xx_evm_defconfig # or make am57xx_hs_evm_defconfig
$ make
```

In case if you are building for AM57xx HS version, you shound export in advance
env variable `TI_SECURE_DEV_PKG` with prover path to SECDEV package.

# Flashing

1. Replace old U-Boot with newly compiled
Firstly, flash new U-Boot and reboot the board.

Run in U-boot shell:
```
=> fas 1
```

Run on the host PC:
```bash
$ fastboot flash xloader MLO
$ fastboot flash bootloader u-boot.img
$ fastboot reboot
```

2. After booting new U-boot, create new partition table for Android
Run in U-boot shell:
```
=> env default -f -a
=> setenv partitions $partitions_android
=> env save
=> fas 1 // fas=fastboot, 1 - USB controller number (OTG)
```

Run on the host PC:
```bash
$ fastboot oem format # create new partition table, based on description in partitions variable.
```

Run in U-boot shell:
```
=> part list mmc 1 // mmc 1 = eMMC
```

3. Flash other partitions
Run on the host PC:
```bash
$ fastboot flash xloader MLO
$ fastboot flash bootloader u-boot.img
$ fastboot flash environment am57xx-.dtb
$ fastboot flash boot boot.img
$ fastboot flash cache cache.img
$ fastboot flash system system.img
$ fastboot flash userdata userdata.img // if doesn't flash - use sudo ./fastboot.sh
$ fastboot flash recovery recovery.img // if doesn't flash correctly - skip it
$ fastboot flash vendor vendor.img
$ fastboot flash zImage zImage
$ fastboot reboot
```
4. Check of SGX driver is loaded correctly (it's vital for bringing up GUI on TI boards):
Verify Android SGX boot by checking line in dmesg on the target board:
```
[    6.451230] PVR_K: UM DDK-(4001660) and KM DDK-(4001660) match. [ OK ]
```

5. If it doesn't load, rebuild SGX in Android dir and push it to `/vendor/lib/modules/pvrsrvkm.ko`

Run on the host PC:
```bash
$ adb root # restarting adbd as root
$ adb remount # remount vendor paritition as rw
$ adb push device/ti/proprietary-open/jacinto6/sgx_src/pvrsrvkm.ko /vendor/lib/modules
device/ti/proprietary-open/jacinto6/sgx_src/pvrsrvkm.ko: 1 file pushed. 5.1 MB/s (288468 bytes in 0.054s)
```
