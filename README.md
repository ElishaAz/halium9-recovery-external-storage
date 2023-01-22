# halium9-recovery-external-storage
Install Halium 9 to recovery and external storage on Redmi Note 8 (ginkgo)

## Prepare your external storage
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

## Create Halium Image
- Download source and dependancies (this will also start building the default image).
```
sudo apt install android-tools-mkbootimg bc bison build-essential ca-certificates cpio curl flex git kmod libssl-dev libtinfo5 python2 unzip wget xz-utils git
git clone https://gitlab.com/ubports/porting/community-ports/android9/xiaomi-redmi-note-8t/xiaomi-ginkgo-willow
cd xiaomi-ginkgo-willow
git clone https://gitlab.com/ubports/community-ports/halium-generic-adaptation-build-tools build-tools
./build-tools/build.sh -b "build"
```
- If you get the `yylloc` error: run `grep -rn "YYLTYPE yylloc"` in the kernel directory (`build/downloads/kernel-xiaomi-willow`) and to all occurences other than `dtc-lexer` add the `extern` keyword.
- Go to the kernel directory `cd build/downloads/kernel-xiaomi-willow`.
- Run `grep -rn by-name/vendor` 
