# halium9-recovery-external-storage
Install Halium 9 to recovery and external storage on Redmi Note 8 (ginkgo) to allow Dual Booting.

## 1. Before starting
As with any OS operations, there is a chance of loosing data, or even bricking your phone. **Back up any important data before starting**. I will not be held responsible for anything.
There are prebuilt images in the [releases](https://github.com/ElishaAz/halium9-recovery-external-storage/releases). If you are using those, you can skip sections 3 & 4. Make sure to pick the right image for your case. Use the `mmcblk1` if you are installing to the microsd card, and `sda` if you are installing to usb-otg.

## 2. Prepare your external storage
1. You can use either a microsd card or usb-otg drive. It should be at least 10GB.
2. Download a working vendor partition for your halium version. I used [V11.0.9.0.PCOMIXM](https://github.com/TryHardDood/mi-vendor-updater/releases/download/ginkgo_global-stable/fw-vendor_ginkgo_miui_GINKGOGlobal_V11.0.9.0.PCOMIXM_59d042a821_9.0.zip).
3. Extract the vendor.img file from the archive, either by flashing it (not recommended) or by using [brotli](https://manpages.ubuntu.com/manpages/jammy/man1/brotli.1.html) and [dat2img](https://github.com/danielmmmm/dat2img).
4. Format your drive and create two partitions. The **first** has to be the exact size in MB as the vendor image. The **second** should be at least 8GB. Note: the order matters!
5. Flash the vendor image to the first partition. You can use `dd` or ubuntu's `Disks`, as follows:
    - Open Disks.
    - Select the drive.
    - Select the first partition.
    - Unmount it if needed (by pressing the `stop` button).
    - Press the `gear` button and select `Restore Partition Image`.
    - Choose your vendor partition image.
6. Flash your system image:
    - If installing Ubports:
        - Download the ubports image for ginkgo from [gitlab ci](https://gitlab.com/ubports/porting/community-ports/android9/xiaomi-redmi-note-8t/xiaomi-ginkgo-willow/-/jobs).
        - Flash the system image to the second partition on your drive using the same steps as above.
        - If the partition and the image are not the same size, you might need to fix the file system.
    - If installing Droidian:
        - Download the latest droidian image for your device (api 28 for halium 9) from [droidian-images](https://github.com/droidian-images/droidian/releases).
        - Extract `data/rootfs.img` from the archive and place it at the root of the second partition on the drive.
        - Open a terminal in the second partition on the drive, and run the follows (the equivalent of installing the GSI):
```
# resize rootfs
e2fsck -fy rootfs.img
resize2fs -f rootfs.img 8G

# install udev rules
wget https://gitlab.com/ubports/porting/community-ports/android9/xiaomi-redmi-note-8t/xiaomi-ginkgo-willow/-/raw/master/overlay/system/lib/udev/rules.d/70-willow.rules # or your devices udev rules
mkdir m
sudo mount rootfs.img m
sudo mv 70-willow.rules m/etc/udev/rules.d/70-willow.rules
sudo umount m

# halium initramfs workaround,
# create symlink to android-rootfs inside /data
ln -s /halium-system/var/lib/lxc/android/android-rootfs.img android-rootfs.img
```
8. Boot to any working system on your mobile device with the external storage plugged in.
9. Open `/dev` in a terminal (e.g. `adb shell`) and note your external storage partitions' names (e.g `/dev/sda1 & /dev/sda2` for usb-otg and `/dev/mmcblk1p1 & /dev/mmcblk1p2` for microsd).

## 3. Create Halium image
1. Download source and dependancies (this will also start building the default image).
```
sudo apt install android-tools-mkbootimg bc bison build-essential ca-certificates cpio curl flex git kmod libssl-dev libtinfo5 python2 unzip wget xz-utils git
git clone https://gitlab.com/ubports/porting/community-ports/android9/xiaomi-redmi-note-8t/xiaomi-ginkgo-willow
cd xiaomi-ginkgo-willow
git clone https://gitlab.com/ubports/community-ports/halium-generic-adaptation-build-tools build-tools
./build-tools/build.sh -b "build"
```
2. If you get the `yylloc` error: run `grep -rn "YYLTYPE yylloc"` in the kernel directory (`build/downloads/kernel-xiaomi-willow`) and to all occurences other than `dtc-lexer` add the `extern` keyword.
3. Go to the kernel directory `cd build/downloads/kernel-xiaomi-willow`.
4. Run `grep -rn by-name/vendor`  and replace all occurences with the path to your drive's first partition under `/dev/block`. E.g. for external storage, replace
```cc
vendor {
    compatible = "android,vendor";
    dev = "/dev/block/platform/soc/1d84000.ufshc/by-name/vendor";
    mnt_flags = "ro,barrier=1,discard";
    fsmgr_flags = "wait,slotselect,avb";
    status = "ok";
};
```
With
```cc
vendor {
    compatible = "android,vendor";
    dev = "/dev/block/sda1"; // replace here
    mnt_flags = "ro,barrier=1,discard";
    fsmgr_flags = "wait,slotselect,avb";
    status = "ok";
};
```
5. Go back to the `xiaomi-ginkgo-willow` directory.
6. Edit the file `deviceinfo`
   - For Ubports, replace the path of `systempart=` with the path of the second partition on your drive (e.g. `systempart=/dev/sda2`).
   - For Droidian, remove the `systempart=` from the commandline, and add `datapart=` with the path to the second partition on your drive (e.g. `datapart=/dev/sda2`).
7. Build the image by running `./build-tools/build.sh -b "build"`
8. You will find the output `boot.img` and `dtbo.img` images at `build/tmp/partitions`.

### 4. Turn the halium boot image into a recovey image
1. Download [android image kitchen](https://forum.xda-developers.com/attachments/aik-linux-v3-8-all-tar-gz.5300923/) ([source code](https://github.com/osm0sis/Android-Image-Kitchen/tree/AIK-Linux)) and extract it.
2. Download the latest twrp image for your device with the same android version as halium. I used [twrp-3.6.2_9-0-ginkgo.img](https://dl.twrp.me/ginkgo/twrp-3.6.2_9-0-ginkgo.img.html).
3. Move it to the directory of andraid image kitchen.
4. Unpack it by running `./unpackimg.sh twrp-3.6.2_9-0-ginkgo.img`
5. Delete the ramdisk, and rename the `split_img` folder.
6. Copy the `boot.img` we created in the previus section and unpack it too: `./unpackimg.sh boot.img`
7. Copy the files back from the renamed `split_img` folder.
8. Replace the `twrp-3.6.2_9-0-ginkgo.img-cmdline, twrp-3.6.2_9-0-ginkgo.img-kernel, twrp-3.6.2_9-0-ginkgo.img-ramdisk.cpio.gz` files with the files from the boot image. Replace `twrp-3.6.2_9-0-ginkgo.img-recovery_dtbo` with the `dtbo.img` we created in the previus section. Delete all the other files from the boot image (starting with `boot.img`).
9. Repack the image: `./repackimg`
10. The new image, `image-new.img`, is your halium9-recovery-external-storage image!

### 5. Install
Note: if you downloaded an image from the releases instead of building, use it in place of `image-new.img`.
1. Flash the newly created image to recovery using fastboot: `fastboot flash recovery image-new.img` or boot it directly: `fastboot boot image-new.img`.
2. Reboot to recovery: `fastboot reboot recovery`
3. Your halium OS should boot without touching your primary OS!
Note: if you have driver problems (e.g. the screen doesn't turn on) you can try rebooting to bootloader and then booting to recovery again to make sure nothing is left over from the newer android OS.
