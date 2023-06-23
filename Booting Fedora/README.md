# Booting Fedora

**Warning: this is an experiment. Many things will not work. You may damage your device.**

## Pre-requisites

1. [TWRP recovery](https://forum.xda-developers.com/t/recovery-unofficial-twrp-for-galaxy-tab-s8-series-snapdragon.4455491/), which will be your safe space, with a functional shell and a working charging mechanism
2. [Project Mu](https://github.com/Robotix22/MU-Qcom), flashed as your secondary bootloader (`boot.img` replacement)
3. [Fedora Workstation (Rawhide) image for AARCH64](https://dl.fedoraproject.org/pub/fedora/linux/development/rawhide/Workstation/aarch64/images/). You'll need to `unxz` it (~16 GB of free space will be required)
4. A portable [parted binary](parted)
5. Robotix22 [DTB](https://github.com/Robotix22/MU-Qcom/raw/8e7ebd3973e54ab22d830f1203fed4877176e99f/Platforms/SM8450Pkg/FdtBlob/sm8450-galaxy-tab-s8-5g.dtb)
6. ADB

An external microSD card will be appreciated.

## Installation

### 1. Preparation

Flash TWRP (as `recovery`) and Project Mu (as `boot`) on your tablet. Boot to TWRP.

Since our TWRP recovery image does not support btrfs partition, we'll need to convert our 

From a Linux machine, execute the following commands:

    # Create a new 10GB ext4 partition file 
    fallocate -l 10G fedora-rootfs.ext4
    mkfs.ext4 fedora-rootfs.ext4

    # Mount Fedora image to a loop device
    losetup -Pf Fedora-Workstation-Rawhide-20230620.n.0.aarch64.raw

    # Mount our btrfs source and ext4 target
    mkdir -p fedora-images/{btrfs,ext4}
    mount /dev/loop4p3 fedora-images/btrfs
    mount fedora-rootfs.ext4 fedora-images/ext4

    # Copy root subvolume to our new partition image
    cp -a fedora-images/btrfs/root/* fedora-images/ext4

    # Unmount and cleanup
    umount fedora-images/btrfs
    umount fedora-images/ext4
    rm fedora-images/ -r


Plug your tablet to your computer, then send Fedora raw image, our ext4 `rootfs` and `parted` binary to your external SD card (ajust Fedora image name as needed): 

    adb push Fedora-Workstation-Rawhide-20230620.n.0.aarch64.raw /external_sd/
    adb push fedora-rootfs.ext4 /external_sd/
    adb push parted /external_sd/

Open a shell to your tablet: 

    adb shell

Then make `parted` binary executable:

    chmod +x /external_sd/parted

Keep your shell open for the next steps.

## 2. Repartition the tablet

We'll need to replace `userdata` with 4 partitions (3 for Fedora + a smaller `userdata`).

Open internal storage in parted: 

    /external_sd/parted /dev/block/sda

Please check that there is 38 partitions, and that the last one is userdata, extending from 14GB to 127GB:

    GNU Parted 3.5
    Using /dev/block/sda
    Welcome to GNU Parted! Type 'help' to view a list of commands.
    (parted) print                                                            
    print
    Warning: Not all of the space available to /dev/block/sda appears to be used,
    you can fix the GPT to use all of the space (an extra 4 blocks) or continue with
    the current setting? 
    Fix/Ignore? ignore                                                        
    ignore
    Model: SAMSUNG KLUDG4UHDC-B0E1 (scsi)
    Disk /dev/block/sda: 127GB
    Sector size (logical/physical): 4096B/4096B
    Partition Table: gpt
    Disk Flags: 

    Number  Start   End     Size    File system  Name           Flags
    1      1085kB  4231kB  3146kB               modemst1
    2      4231kB  7377kB  3146kB               modemst2
    3      7377kB  7381kB  4096B                fsc
    4      7381kB  7389kB  8192B                ssd
    5      7389kB  40.9MB  33.6MB  ext4         persist
    6      40.9MB  61.9MB  21.0MB  ext4         efs
    7      61.9MB  72.4MB  10.5MB               param
    8      72.4MB  82.9MB  10.5MB               debug
    9      82.9MB  104MB   21.0MB  ext4         sec_efs
    10      104MB   105MB   1049kB               misc
    11      105MB   105MB   524kB                keystore
    12      105MB   106MB   524kB                frp
    13      106MB   123MB   16.8MB               rawdump
    14      123MB   165MB   42.4MB               bota
    15      165MB   166MB   524kB                persistent
    16      166MB   170MB   4194kB               steady
    17      170MB   237MB   67.1MB  ext4         dsp
    18      237MB   254MB   16.8MB               dqmdbg
    19      254MB   411MB   157MB                apnhlos
    20      411MB   428MB   16.8MB               keyrefuge
    21      428MB   445MB   16.8MB               keydata
    22      445MB   445MB   65.5kB               vbmeta_system
    23      445MB   478MB   33.6MB  ext4         metadata
    24      478MB   650MB   172MB                modem          msftdata
    25      650MB   751MB   101MB                boot
    26      751MB   856MB   105MB                recovery
    27      856MB   956MB   101MB                vendor_boot
    28      956MB   12.1GB  11.1GB               super
    29      12.1GB  13.1GB  1049MB  ext2         prism
    30      13.1GB  13.2GB  31.5MB  ext2         optics
    31      13.2GB  13.8GB  629MB   ext4         cache
    32      13.8GB  13.9GB  52.4MB  ext4         omr
    33      13.9GB  13.9GB  52.4MB  ext4         spu
    34      13.9GB  13.9GB  8389kB               dtbo
    35      13.9GB  13.9GB  4194kB               testparti
    36      13.9GB  14.0GB  67.1MB               core_nhlos_a
    37      14.0GB  14.0GB  2097kB               catefv
    38      14.0GB  127GB   113GB   fat32        userdata

Delete userdata then make four new partition: 

    rm 38

Create 4 new partitions: 

    (parted) mkpart
    Partition name? fedora_p1
    File system type? fat32
    Start? 14.0G
    End? 16.5G

    (parted) mkpart
    Partition name? fedora_p2
    File system type? ext4
    Start? 16.5G
    End? 18.0G

    (parted) mkpart
    Partition name? fedora_p3
    File system type? ext4
    Start? 18.0G
    End? 40.0G

    (parted) mkpart
    Partition name? userdata
    File system type? ext4
    Start? 40.0G
    End? 127.0G

Set esp on fedora_p1 (partition 38):

    set 38 esp on

Check that everything is okay: 

    (parted) print
    Model: SAMSUNG KLUDG4UHDC-B0E1 (scsi)
    Disk /dev/block/sda: 127GB
    Sector size (logical/physical): 4096B/4096B
    Partition Table: gpt
    Disk Flags: 

    Number  Start   End     Size    File system  Name           Flags
    1      1085kB  4231kB  3146kB               modemst1
    2      4231kB  7377kB  3146kB               modemst2
    3      7377kB  7381kB  4096B                fsc
    4      7381kB  7389kB  8192B                ssd
    5      7389kB  40.9MB  33.6MB  ext4         persist
    6      40.9MB  61.9MB  21.0MB  ext4         efs
    7      61.9MB  72.4MB  10.5MB               param
    8      72.4MB  82.9MB  10.5MB               debug
    9      82.9MB  104MB   21.0MB  ext4         sec_efs
    10      104MB   105MB   1049kB               misc
    11      105MB   105MB   524kB                keystore
    12      105MB   106MB   524kB                frp
    13      106MB   123MB   16.8MB               rawdump
    14      123MB   165MB   42.4MB               bota
    15      165MB   166MB   524kB                persistent
    16      166MB   170MB   4194kB               steady
    17      170MB   237MB   67.1MB  ext4         dsp
    18      237MB   254MB   16.8MB               dqmdbg
    19      254MB   411MB   157MB                apnhlos
    20      411MB   428MB   16.8MB               keyrefuge
    21      428MB   445MB   16.8MB               keydata
    22      445MB   445MB   65.5kB               vbmeta_system
    23      445MB   478MB   33.6MB  ext4         metadata
    24      478MB   650MB   172MB                modem          msftdata
    25      650MB   751MB   101MB                boot
    26      751MB   856MB   105MB                recovery
    27      856MB   956MB   101MB                vendor_boot
    28      956MB   12.1GB  11.1GB               super
    29      12.1GB  13.1GB  1049MB  ext2         prism
    30      13.1GB  13.2GB  31.5MB  ext2         optics
    31      13.2GB  13.8GB  629MB   ext4         cache
    32      13.8GB  13.9GB  52.4MB  ext4         omr
    33      13.9GB  13.9GB  52.4MB  ext4         spu
    34      13.9GB  13.9GB  8389kB               dtbo
    35      13.9GB  13.9GB  4194kB               testparti
    36      13.9GB  14.0GB  67.1MB               core_nhlos_a
    37      14.0GB  14.0GB  2097kB               catefv
    38      14.0GB  16.5GB  2508MB  fat32        fedora_p1      boot, esp
    39      16.5GB  18.0GB  1499MB  ext4         fedora_p2
    40      18.0GB  40.0GB  22.0GB  ext4         fedora_p3
    41      40.0GB  127GB   87.2GB  ext4         userdata

Exit:

    quit

Reboot TWRP the re-open an `adb shell`.

## 3. Copy Fedora image to the new partitions

Mount the Fedora image to a loop device:

    losetup /dev/block/loop7 /external_sd/Fedora-Workstation-Rawhide-20230620.n.0.aarch64.raw

Copy image partitions to your new partitions:

    dd if=/dev/block/loop7p1 of=/dev/block/sda38 bs=4M
    dd if=/dev/block/loop7p2 of=/dev/block/sda39 bs=4M
    dd if=/external_sd/fedora-rootfs.ext4 of=/dev/block/sda40 bs=4M

# 4. Repack `initramfs` (optional)

At this stage, Fedora kernel will be able to boot, but you won't be able to do anything useful. 

To do some debugging you might want to customize `initramfs` to add some kernel modules and replace `systemd init` with a debug script.

To do so, chroot yourself to Fedora:

    mkdir /fedora
    mount /dev/block/sda40 /fedora
    mount --bind /dev /fedora/dev
    mount --bind /sys /fedora/sys
    mount --bind /proc /fedora/proc
    chroot /fedora

Then mount `boot` partition: 

    export PATH=/bin
    mount /boot

Unpack initramfs:

    mkdir /tmp/initrd
    cd /tmp/initrd
    zcat /boot/initramfs-6.4.0-0.rc7.53.fc39.aarch64.img | cpio -idmv

Replace initramfs kernel modules with the one from rootfs, which contains more modules.

    rm -r /tmp/initrd/lib/modules/6.4.0-0.rc7.53.fc39.aarch64/
    cp -a /lib/modules/6.4.0-0.rc7.53.fc39.aarch64/ /tmp/initrd/lib/modules/

Replace `systemd init` with a debug script: 

    rm /tmp/initrd/init
    touch /tmp/initrd/init
    chmod +x /tmp/initrd/init

You may use my [init](init) script as an example.

Append Robotix22 Device Tree Blob to `/boot/dtb/qcom/`:

    adb push sm8450-galaxy-tab-s8-5g.dtb /fedora/boot/dtb/qcom/

Repack `initramfs`:

    find . | cpio -o -c -R root:root | gzip -9 > /boot/initramfs-6.4.0-0.rc7.53.fc39.aarch64.img

Exit chroot then reboot the tablet:

    exit
    reboot

You should now be able to boot `Fedora on #37`.