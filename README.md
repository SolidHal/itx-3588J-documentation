
Some useful information for liberating the itx-3588J board


#TODO
- dump wiki pages from https://wiki.t-firefly.com/en/Core-3588J/index.html
- make tool to create SD boot&upgrade images from linux
    - the "firmware" upgrade process requires functional uboot and recovery partitions on the emmc
    - all we really need to have a bootable sd card image is
      - 1) an ext4 partition marked bootable with extlinux
- build packs some ini text files with the u-boot image
    - look at the .bin files the ini files load
    - look at the RKTRUST ini and its binaries
- make a libre linux image that the existing uboot boots
- make a bootable root fs with mesa 22.2 (from bookworm/sid)
  - won't work, panfrost has support for first gen valhall gpus (mali G-57)
    but not the rk3588s mali-G610
    RK3588's G610 is 2nd-gen Valhall ("v10" in developer-speak here)

- test basic functionality
- test 2FA encryption



## Bootloader Source
- linux images use uboot
https://github.com/SolidHal/itx-3588j-u-boot


## Linux Kernel Source
https://github.com/SolidHal/itx-3588j-linux-kernel

## Linux Kernel Config
- pulled from ubuntu image
- compare with one from source when we get it



## Firmware Upgrade sdcard layout
- sd card partitioned as NTFS

### SD card contents
- rksdfw.tag
  - empty file
  - copy in tools/sd_update_resources
- sd_boot_config.config
  - text file
  - copy in tools/sd_update_resources
- sdupdate.img
  - "firmware" contents
    - uboot, kernel, rootfs, etc

sdupdate.img is just the packed image from the build process, like ITX-3588J_Debian11_v0.1.0a_220824.img

so creating and update sd card is just:
- add the tag, config, sdupdate file to an NTFS formatted sdcard with a GPT partition table


## Additional sources
### recovery image
https://github.com/SolidHal/itx-3588j-recovery

### debian rootfs build scripts
https://github.com/SolidHal/itx-3588j-debian

## Docs sources
https://github.com/SolidHal/itx-3588j-docs
https://github.com/SolidHal/itx-3588j-docs-socs


## Linux SDK Build Debian Bullseye

### build reqs
```
sudo apt-get install git ssh make gcc libssl-dev liblz4-tool \
expect g++ patchelf chrpath gawk texinfo chrpath diffstat binfmt-support \
qemu-user-static live-build bison flex fakeroot cmake gcc-multilib g++-multilib unzip \
device-tree-compiler ncurses-dev debootstrap qemu live-build libparted2 \

sudo apt install btrfs-progs command-not-found udisks2 python2.7

TODO make this process not absolutly suck
- python-linaro-image-tools are only availabe for python2 which makes things hard

install repo from contrib manually rather than adding the contrib repo
```

```
https://packages.debian.org/bullseye/repo
sudo apt install ./repo_2*_all.deb
```

install miniconda
```
https://docs.conda.io/en/latest/miniconda.html#linux-installers
```

create env for python2
```
conda create -n 3588J python=2
```

install what we can using conda
```
conda install -c conda-forge chardet
conda install -c conda-forge python-debian
conda install -c conda-forge python-apt
conda install -c anaconda-cluster python-apt
conda install -c conda-forge pyyaml
```

force install python-dbus
```
https://packages.debian.org/buster/python-dbus
sudo dpkg --force-all -i python-dbus_1.2.8-3_amd64.deb
```

force install python-parted
```
https://packages.debian.org/buster/python-parted
sudo dpkg --force-all -i python-parted_3.11.2-10_amd64.deb
```

force install python-linaro-image-tools_2016.05-1_all.deb
```
https://packages.debian.org/buster/python-linaro-image-tools
sudo dpkg --force-all -i python-linaro-image-tools_2016.05-1.1_all.deb
```

force install linaro-image-tools
```
https://packages.debian.org/buster/linaro-image-tools
sudo dpkg --force-all -i linaro-image-tools_2016.05-1.1_amd64.deb
```

#### lie to the package manager

now manually modify `/var/lib/dpkg/status` for the force installed packages
to remove the depends that conda provides for us to avoid breaking apt


### Build

Precompile Configuration
```
sudo ./build.sh itx-3588j-debian.mk
```


#### Full Build

building the rootfs requires sudo
and expects the folder to always be named "ubuntu_rootfs"
```
mkdir -p ubuntu_rootfs/
sudo ./build.sh debian
```

now we can run the rest of the build
```
./build.sh
```

the result will be in

```
rockdev/pack/
```


#### Partial Builds

u-boot
```
./build.sh uboot
```

kernel
```
./build.sh extboot
```

recovery
```
./build.sh recovery
```

debian
```
mkdir -p ubuntu_rootfs/
sudo ./build.sh debian
ln -sf $(pwd)/debian/debian*-rootfs.img $(pwd)/ubuntu_rootfs/rootfs.img
ln -sf $(pwd)/debian/debian*-rootfs.img $(pwd)/ubuntu_rootfs/debian/debian11-rootfs.img
```

Update each part of the .img link to the directory rockdev/:
```
sudo ./mkfirmware.sh
```

Pack the image, the img will be saved to the directory rockdev/pack/.
```
sudo ./build.sh updateimg
```



## Bootable SD card partition layout

kernel cmdline when booting from sd with ubuntu image in
main flash
```
storagemedia=sd androidboot.storagemedia=sd androidboot.mode=normal  storagenode=/mmc@fe2c0000 androidboot.verifiedbootstate=orange ro rootwait earlycon=uart8250,mmio32,0xfeb50000 console=ttyFIQ0 irqchip.gicv3_pseudo_nmi=0 root=PARTLABEL=rootfs rootfstype=ext4 overlayroot=device:dev=PARTLABEL=userdata,fstype=ext4,mkfs=1 coherent_pool=1m systemd.gpt_auto=0 cgroup_enable=memory swapaccount=1
```

```
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 23000000-0000-4C4A-8000-699000005ABB

Device        Start      End  Sectors  Size Type
/dev/sdc1     16384    24575     8192    4M unknown
/dev/sdc2     24576    32767     8192    4M unknown
/dev/sdc3     32768   557055   524288  256M unknown
/dev/sdc4    557056   819199   262144  128M unknown
/dev/sdc5    819200   884735    65536   32M unknown
/dev/sdc6    884736 13467647 12582912    6G unknown
/dev/sdc7  13467648 62333887 48866240 23.3G unknown
```

```
/dev/sdc6 on /media/solidhal/a21904ef-050c-44a0-ab74-3d2633d42051 type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdc3 on /media/solidhal/boot type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdc7 on /media/solidhal/69af03a6-c29a-4f56-88bf-69d42dc10d8f type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
```


- partition1
 -

- partition2
 -

- partition3 - boot
```
aio-3588sjd4.dtb
aio-3588sjd4-mipi101-M101014-BE45-A1.dtb
config-5.10.66
extlinux
Image-5.10.66
initrd-5.10.66
lib
logo.bmp
logo_kernel.bmp
lost+found
rk3588-firefly-itx-3588j.dtb
rk3588-firefly-itx-3588j-dual-mipi101-M101014-BE45-A1.dtb
rk3588-firefly-itx-3588j-mipi101-M101014-BE45-A1.dtb
rk-kernel.dtb
roc-rk3588s-pc.dtb
roc-rk3588s-pc-mipi101-M101014-BE45-A1.dtb
System.map-5.10.66
```

rk-kernel.dtb is a hardlink to rk3588-firefly-itx-3588j.dtb
kernel config matches the one from the stock ubuntu image

- partition4
 -

- partition5
 -

- partition6 - rootfs
```
bin   home        opt            sbin              sys     var
boot  lib         proc           sdcard            system  vendor
data  lost+found  rockchip-test  sha256sum.README  tmp
dev   media       root           sha256sum.txt     udisk
etc   mnt         run            srv               usr
```
- partition7 - overlays
```
docker  lost+found  recovery  rootfs_overlay  rootfs_overlay-workdir  wifi_chip
```

 - rootfs overlays
  - userdata
  - additional tools
  - kernel modules
 - wifi_chip file
   - contents: "AP6275P^@"


## SD firmware update layout

## emmc bootable image

partition layout
from device/rockchip/rk3588/parameter-ubuntu-fit.txt
which is used by
device/rockchip/common/mkfirmware.sh
which packs the bootable image
```
CMDLINE: mtdparts=rk29xxnand:
0x00002000@0x00004000(uboot),
0x00002000@0x00006000(misc),
0x00080000@0x00008000(boot:bootable),
0x00040000@0x00088000(recovery),
0x00010000@0x000c8000(backup),
0x00c00000@0x000d8000(rootfs),
-@0x00cd8000(userdata:grow)
```

package layout
defined by
device/rockchip/rk3588/itx-3588j-ubuntu.mk


package-file	package-file
bootloader	Image/MiniLoaderAll.bin
parameter	Image/parameter.txt
uboot		Image/uboot.img
misc		Image/misc.img
boot		Image/boot.img
recovery	Image/recovery.img
rootfs		Image/rootfs.img
userdata	RESERVED
backup		RESERVED


MiniLoaderAll is linked to

MiniLoaderAll.bin -> ../u-boot/rk3588_spl_loader_v1.07.111.bin

while uboot.img is linked to

uboot.img -> ../u-boot/uboot.img

## boot process (tenative)
uboot spl (miniloaderall)
uboot tpl ?? CONFIG_SUPPORT_TPL=y
uboot
extlinux
linux

## emmc partition layout

```
Disk /dev/mmcblk0: 116.48 GiB, 125069950976 bytes, 244277248 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 73987B6B-4974-4C94-A3E8-58AB2EB7A946

Device            Start       End   Sectors   Size Type
/dev/mmcblk0p1    16384     24575      8192     4M unknown
/dev/mmcblk0p2    24576     32767      8192     4M unknown
/dev/mmcblk0p3    32768    557055    524288   256M unknown
/dev/mmcblk0p4   557056    819199    262144   128M unknown
/dev/mmcblk0p5   819200    884735     65536    32M unknown
/dev/mmcblk0p6   884736  13467647  12582912     6G unknown
/dev/mmcblk0p7 13467648 244277214 230809567 110.1G unknown
```

```
(parted) print
Model: MMC Y2P128 (sd/mmc)
Disk /dev/mmcblk0: 125GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name      Flags
 1      8389kB  12.6MB  4194kB               uboot
 2      12.6MB  16.8MB  4194kB               misc
 3      16.8MB  285MB   268MB   ext4         boot      legacy_boot
 4      285MB   419MB   134MB                recovery
 5      419MB   453MB   33.6MB               backup
 6      453MB   6895MB  6442MB  ext4         rootfs
 7      6895MB  125GB   118GB   ext4         userdata
```

p1, p2 are uboot partitions
only p1 actually has a uboot image
p2 has nearly all 0s, with "false" at byte 17221. Which is kinda strange. Must be some sort of "do not use this" flag to the bootrom

p1 is labeled uboot
p2 is labeled misc

TODO: theory: if p1 is not bootable, it might try p2 instead? copy p1 to p2 before testing uboot images

p3 is the boot partition, its mountable, containing the kernel image, extlinux.conf for uboot, initrd, and device trees
p4 is labeled recovery, its not a mountable fs

p5 is labeled backup, its not a mountable fs
p6 is the rootfs, its mountable
p7 is the overlay/userdata partition

dumped the emmc part table using

```
sfdisk -d /dev/mmcblk0 > emmc_part_table
```

it can be restored by doing
```
sfdisk /dev/mmcblk0 < emmc_part_table
```


## u-boot notes

the config is made up of
configs/rk3588_defconfig
with the following fragment
configs/firefly-linux.config

added the following to the fragment for logging:
```
CONFIG_LOG=y
CONFIG_SPL_LOG=y
CONFIG_LOGLEVEL=7
CONFIG_SPL_LOGLEVEL=7
CONFIG_SYS_CONSOLE_INFO_QUIET=n
```

this seems like an interesting option

```
CONFIG_AUTOBOOT=y
```

build packs some ini text files with the u-boot image
```
********boot_merger ver 1.2********
Info:Pack loader ok.
pack loader okay! Input: /home/solidhal/ITX-3588J-Linux-SDK/rkbin/RKBOOT/RK3588MINIALL.ini
/home/solidhal/ITX-3588J-Linux-SDK/u-boot

Image(no-signed, version=0): uboot.img (FIT with uboot, trust...) is ready
Image(no-signed): rk3588_spl_loader_v1.07.111.bin (with spl, ddr...) is ready
pack uboot.img okay! Input: /home/solidhal/ITX-3588J-Linux-SDK/rkbin/RKTRUST/RK3588TRUST.ini

Platform RK3588 is build OK, with exist .config
/home/solidhal/ITX-3588J-Linux-SDK/prebuilts/gcc/linux-x86/aarch64/gcc-arm-10.3-2021.07-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-
```

## stock u-boot boot process
autoboots to
```
bootcmd=boot_android ${devtype} ${devnum};boot_fit;bootrkp;run distro_bootcmd;
```

which when running ubuntu/debian ends up only really doing `run distro_bootcmd`
which is
```
distro_bootcmd=for target in ${boot_targets}; do run bootcmd_${target}; done
```
which goes through this list
so if the sd card is present, that is booted first. Followed by the emmc, etc...
```
boot_targets=mmc1 mmc0 mtd2 mtd1 mtd0 usb0 pxe dhcp
```
to run one of
```
bootcmd_dhcp=run boot_net_usb_start; if dhcp ${scriptaddr} ${boot_script_dhcp}; then source ${scriptaddr}; fi;
bootcmd_mmc0=setenv devnum 0; run mmc_boot
bootcmd_mmc1=setenv devnum 1; run mmc_boot
bootcmd_mtd0=setenv devnum 0; run mtd_boot
bootcmd_mtd1=setenv devnum 1; run mtd_boot
bootcmd_mtd2=setenv devnum 2; run mtd_boot
bootcmd_pxe=run boot_net_usb_start; dhcp; if pxe get; then pxe boot; fi
bootcmd_usb0=setenv devnum 0; run usb_boot
```
continuing down the emmc/sd boot path:
```
mmc_boot=if mmc dev ${devnum}; then setenv devtype mmc; run scan_dev_for_boot_part; fi
```

so on emmc boot
```
devnum=0
devtype=mmc
```

moving on,
```
scan_dev_for_boot_part=part list ${devtype} ${devnum} -bootable devplist; env exists devplist || setenv devplist 1; for distro_bootpart in ${devplist}; do if fstype ${devtype} ${devnum}:${distro_bootpart} bootfstype; then run scan_dev_for_boot; fi; done
```

so we find the partition on the device that is marked as "bootable" and save it to `devplist`
if we don't find a bootable partition, we just arbitrarily choose the first one

so our vars look like:

```
devnum=0
devtype=mmc
devplist=3
distro_bootpart=3
bootfstype=ext4
```

for reference, below is the output of `part list mmc 0`

```
Partition Map for MMC device 0  --   Partition Type: EFI

Part    Start LBA       End LBA         Name
        Attributes
        Type GUID
        Partition GUID
  1     0x00004000      0x00005fff      "uboot"
        attrs:  0x0000000000000000
        type:   f808d051-1602-4dcd-9452-f9637fefc49a
        guid:   b750e44e-833f-4a30-c38c-b117241d84d4
  2     0x00006000      0x00007fff      "misc"
        attrs:  0x0000000000000000
        type:   c6d08308-e418-4124-8890-f8411e3d8d87
        guid:   a1c81622-7741-47ad-b846-c6972488d396
  3     0x00008000      0x00087fff      "boot"
        attrs:  0x0000000000000004
        type:   2a583e58-486a-4bd4-ace4-8d5454e97f5c
        guid:   43784a32-a03d-4ade-92c6-ede64ff9b794
  4     0x00088000      0x000c7fff      "recovery"
        attrs:  0x0000000000000000
        type:   6115f139-4f47-4baf-8d23-b6957eaee4b3
        guid:   000b305f-484a-4582-9090-4ad0099d47bd
  5     0x000c8000      0x000d7fff      "backup"
        attrs:  0x0000000000000000
        type:   a83fba16-d354-45c5-8b44-3ec50832d363
        guid:   24eeb649-277f-4c11-ffeb-d9f20027a83b
  6     0x000d8000      0x00cd7fff      "rootfs"
        attrs:  0x0000000000000000
        type:   500e2214-b72d-4cc3-d7c1-8419260130f5
        guid:   614e0000-0000-4b53-8000-1d28000054a9
  7     0x00cd8000      0x0e8f5fde      "userdata"
        attrs:  0x0000000000000000
        type:   e099da71-5450-44ea-aa9f-1b771c582805
        guid:   2bfee623-d83c-426a-ab80-21732c9bb7d3
```

on to `run scan_dev_for_boot`
```
scan_dev_for_boot=echo Scanning ${devtype} ${devnum}:${distro_bootpart}...; for prefix in ${boot_prefixes}; do run scan_dev_for_extlinux; run scan_dev_for_scripts; done;
```

so `prefix` is one of
```
boot_prefixes=/ /boot/
```

```
scan_dev_for_extlinux=if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}extlinux/extlinux.conf; then echo Found ${prefix}extlinux/extlinux.conf; run boot_extlinux; echo SCRIPT FAILED: continuing...; fi
```

```
boot_extlinux=sysboot ${devtype} ${devnum}:${distro_bootpart} any ${scriptaddr} ${prefix}extlinux/extlinux.conf
```

```
scriptaddr=0x00500000
```

## Manually boot sdcard
spam ctrl+c to get into uboot
then
sysboot mmc 1:1 any 0x00500000 /extlinux/extlinux.conf

or just let it boot normally, the sdcard is preferred over the internal emmc


## Make a bootable SD card

to get something basic booting need:
```
cat /extlinux/extlinux.conf
label rk-kernel.dtb linux-5.10.66
        kernel /Image-5.10.66
        fdt /rk-kernel.dtb
        initrd /initrd-5.10.66
```
- kernel image
  - arch/arm64/boot/Image
- device tree binary
  - rk3588-firefly-itx-3588j.dtb
- basic initrd

sd card has 2 partitions
gpt table
both formatted as ext4
partition 1 "boot" marked as bootable
partition 2 is ignored for now

## Linux notes
PCIE_ROCKCHIP_DW_HOST

provided kernel source is based on *android* linux 5.10
which explains all of the android related cmdline options...
```
[    1.600617] Kernel command line: storagemedia=emmc androidboot.storagemedia=emmc androidboot.mode=normal  storagenode=/mmc@fe2e0000 androidboot.verifiedbootstate=orange ro rootwait earlycon=uart8250,mmio32,0xfeb50000 console=ttyFIQ0 irqchip.gicv3_pseudo_nmi=0 root=PARTLABEL=rootfs rootfstype=ext4 overlayroot=device:dev=PARTLABEL=userdata,fstype=ext4,mkfs=1 coherent_pool=1m systemd.gpt_auto=0 cgroup_enable=memory swapaccount=1
```


this will complicate getting linux libre working
need to:
1) transform kernel config to something linux-libre can work with DONE
2) determine what patches we much bring in from the provided kernel


to get something basic booting need:
```
cat /boot/extlinux/extlinux.conf
label rk-kernel.dtb linux-5.10.66
        kernel /Image-5.10.66
        fdt /rk-kernel.dtb
        initrd /initrd-5.10.66
```
- kernel image
  - arch/arm64/boot/Image
- device tree binary
  - rk3588-firefly-itx-3588j.dtb
  - kernel/arch/arm64/boot/dts/rockchip/rk3588-firefly-itx-3588j.dts
  - kernel/arch/arm64/boot/dts/rockchip/rk3588-firefly-itx-3588j.dtsi
  - kernel/arch/arm64/boot/dts/rockchip/rk3588-firefly-itx-cam-8ms1m.dtsi
  - etc
- basic initrd

what if we skip the initrd?

test kernel by creating an sdcard w/ extlinux & kernel image that we can boot from the emmcs uboot
- just need one ext4 partition, marked "bootable"

TARGET_ARCH          =arm64
TARGET_KERNEL_CONFIG =rockchip_linux_defconfig
TARGET_KERNEL_DTS    =rk3588-firefly-itx-3588j
TARGET_KERNEL_CONFIG_FRAGMENT =firefly-linux.config
RK_TARGET_PRODUCT    =rk3588


#### failed boot:
```
=> sysboot mmc 1:1 any 0x00500000 /extlinux/extlinux.conf
Retrieving file: /extlinux/extlinux.conf
=================begin===================
122 bytes read in 5 ms (23.4 KiB/s)
1:      rk-kernel.dtb linux-5.10.66
Retrieving file: /initrd-5.10.66
=================begin===================
9233322 bytes read in 744 ms (11.8 MiB/s)
Retrieving file: /Image-5.10.66
=================begin===================
40776616 bytes read in 3267 ms (11.9 MiB/s)
Retrieving file: /rk-kernel.dtb
=================begin===================
160168 bytes read in 17 ms (9 MiB/s)
Fdt Ramdisk skip relocation ## I think this means the image just isn't at a high address?
Bad Linux ARM64 Image magic!
```
^^^^ this was because I was using vmlinux instead of arch/arm64/boot/Image

#### vs working boot:

```
Scanning mmc 0:3...
Found /extlinux/extlinux.conf
Retrieving file: /extlinux/extlinux.conf
=================begin===================
101 bytes read in 3 ms (32.2 KiB/s)
1:      rk-kernel.dtb linux-5.10.66
Retrieving file: /initrd-5.10.66
=================begin===================
9233322 bytes read in 54 ms (163.1 MiB/s)
Retrieving file: /Image-5.10.66
=================begin===================
35807744 bytes read in 229 ms (149.1 MiB/s)
Retrieving file: /rk-kernel.dtb
=================begin===================
159672 bytes read in 9 ms (16.9 MiB/s)
Fdt Ramdisk skip relocation
## Flattened Device Tree blob at 0x0a100000
   Booting using the fdt blob at 0x0a100000
  'reserved-memory' cma: addr=10000000 size=10000000
   Using Device Tree in place at 000000000a100000, end 000000000a129fb7
No resource partition
No file: logo_kernel.bmp
=================begin===================
512 bytes read in 3 ms (166 KiB/s)
logo(Distro): logo_kernel.bmp
=================begin===================
127818 bytes read in 5 ms (24.4 MiB/s)
logo(Distro): logo_kernel.bmp
Adding bank: 0x00200000 - 0x08400000 (size: 0x08200000)
Adding bank: 0x09400000 - 0xf0000000 (size: 0xe6c00000)
Adding bank: 0x100000000 - 0x3fc000000 (size: 0x2fc000000)
Adding bank: 0x3fc500000 - 0x3fff00000 (size: 0x03a00000)
Total: 625.550 ms
```

fails to even get to the basic initramfs, kernel panics

```
[    0.010540] Platform MSI: msi-controller@fe640000 domain created
[    0.011097] Platform MSI: msi-controller@fe660000 domain created
[    0.011918] PCI/MSI: /interrupt-controller@fe600000/msi-controller@fe640000 domain created
[    0.012728] PCI/MSI: /interrupt-controller@fe600000/msi-controller@fe660000 domain created
[    0.013593] EFI services will not be available.
[    0.014398] smp: Bringing up secondary CPUs ...
I/TC: Secondary CPU 1 initializing
I/TC: Secondary CPU 1 switching to normal world boot
[    0.016073] Detected VIPT I-cache on CPU1
[    0.016112] GICv3: CPU1: found redistributor 100 region 0:0x00000000fe6a0000
[    0.016125] GICv3: CPU1: using allocated LPI pending table @0x00000001001c0000
[    1.422700] ITS queue timeout (192 1)
[    1.422705] ITS cmd its_build_mapc_cmd failed
[    2.829432] ITS queue timeout (256 1)
[    2.829436] ITS cmd its_build_invall_cmd failed
[    4.237275] ITS queue timeout (192 1)
[    4.237279] ITS cmd its_build_mapc_cmd failed
[    5.178355] CPU1: failed to come online
[    5.182503] CPU1: failed in unknown state : 0x0
[    5.182913] ------------[ cut here ]------------
[    5.183329] Dying CPU not properly vacated!
[    5.183341] WARNING: CPU: 0 PID: 1 at kernel/sched/core.c:9506 sched_cpu_dying+0xf0/0x1b0
[    5.184460] Modules linked in:
[    5.184742] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 5.19.5-gnu+ #2
[    5.185316] Hardware name: Firefly ITX-3588J HDMI(Linux) (DT)
[    5.185833] pstate: 600000c9 (nZCv daIF -PAN -UAO -TCO -DIT -SSBS BTYPE=--)
[    5.186463] pc : sched_cpu_dying+0xf0/0x1b0
[    5.186843] lr : sched_cpu_dying+0xf0/0x1b0
[    5.187226] sp : ffffffc009fbbc50
[    5.187525] x29: ffffffc009fbbc50 x28: 0000000000000000 x27: 0000000000000000
[    5.188174] x26: ffffffc3f4771000 x25: ffffffc009666770 x24: ffffff83fddd7770
[    5.188824] x23: 0000000000000000 x22: 000000000000005f x21: ffffffc009bba920
[    5.189474] x20: 0000000000000001 x19: ffffff83fdde5940 x18: 0000000000000000
[    5.190122] x17: 0000000000000000 x16: 0000000000000000 x15: 000000000000000a
[    5.190772] x14: 0000000000000000 x13: 2164657461636176 x12: 20796c7265706f72
[    5.191420] x11: 656820747563205b x10: 2d2d2d2d2d2d2d2d x9 : ffffffc0080e7324
[    5.192068] x8 : 5d20657265682074 x7 : 727020746f6e2055 x6 : ffffffc009ee3a69
[    5.192717] x5 : 00000000000affa8 x4 : 000000000000000d x3 : 0000000000000000
[    5.193367] x2 : 0000000000000000 x1 : 0000000000000000 x0 : ffffff8100350000
[    5.194016] Call trace:
[    5.194237]  sched_cpu_dying+0xf0/0x1b0
[    5.194592]  cpuhp_invoke_callback+0x100/0x260
[    5.194999]  cpuhp_invoke_callback_range+0x78/0xac
[    5.195436]  _cpu_up+0x180/0x1a8
[    5.195733]  cpu_up+0x88/0x9c
[    5.196005]  bringup_nonboot_cpus+0x98/0x9c
[    5.196385]  smp_init+0x3c/0x84
[    5.196673]  kernel_init_freeable+0x128/0x2a0
[    5.197071]  kernel_init+0x34/0x138
[    5.197392]  ret_from_fork+0x10/0x20
[    5.197721] ---[ end trace 0000000000000000 ]---
[    5.198137] CPU1 enqueued tasks (0 total):
I/TC: Secondary CPU 2 initializing
I/TC: Secondary CPU 2 switching to normal world boot
```


TODO
so we can boot into initramfs but can't seem to get much out of the init console
- rebuild u-boot with correct uart baud rate, current build has wrong baud still
  - write this to a different sdcard so we can continue using this one to debug
- play around with uboot args?
- the kernel cmdline seems to end up with some interesting tty/console settings... this is likely suspect. Not sure yet where they are coming from


okay, so we can't boot our own initramfs
try booting the stock initramfs (that worked in the past I believe)


booting into stock initramfs didn't really work, though systemd prints things before hanging indefinitely

ported the tty/fiq driver to linux

see the following in the log:

```
[    0.078075] Registered FIQ tty driver
```

in the stock log it says more:
```
[    2.093471] Registered FIQ tty driver
[    2.094064] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    2.094799] ASID allocator initialised with 65536 entries
[    2.096819] printk: console [ramoops-1] enabled
[    2.097245] pstore: Registered ramoops as persistent store backend
[    2.097810] ramoops: using 0xf0000@0x110000, ecc: 0
[    2.142506] rockchip-gpio fd8a0000.gpio: probed /pinctrl/gpio@fd8a0000
[    2.143340] rockchip-gpio fec20000.gpio: probed /pinctrl/gpio@fec20000
[    2.144150] rockchip-gpio fec30000.gpio: probed /pinctrl/gpio@fec30000
[    2.144989] rockchip-gpio fec40000.gpio: probed /pinctrl/gpio@fec40000
[    2.145823] rockchip-gpio fec50000.gpio: probed /pinctrl/gpio@fec50000
[    2.146448] rockchip-pinctrl pinctrl: probed pinctrl
[    2.216071] raid6: neonx8   gen()  5726 MB/s
[    2.272825] raid6: neonx8   xor()  4386 MB/s
[    2.329586] raid6: neonx4   gen()  5720 MB/s
[    2.386342] raid6: neonx4   xor()  4458 MB/s
[    2.443101] raid6: neonx2   gen()  5391 MB/s
[    2.499858] raid6: neonx2   xor()  4208 MB/s
[    2.556619] raid6: neonx1   gen()  4439 MB/s
[    2.613372] raid6: neonx1   xor()  3465 MB/s
[    2.670146] raid6: int64x8  gen()  1370 MB/s
[    2.726892] raid6: int64x8  xor()   881 MB/s
[    2.783651] raid6: int64x4  gen()  1716 MB/s
[    2.840403] raid6: int64x4  xor()   956 MB/s
[    2.897165] raid6: int64x2  gen()  2528 MB/s
[    2.953919] raid6: int64x2  xor()  1392 MB/s
[    3.010677] raid6: int64x1  gen()  2095 MB/s
[    3.067439] raid6: int64x1  xor()  1141 MB/s
[    3.067831] raid6: using algorithm neonx8 gen() 5726 MB/s
[    3.068323] raid6: .... xor() 4386 MB/s, rmw enabled
[    3.068777] raid6: using neon recovery algorithm
[    3.069748] fiq_debugger fiq_debugger.0: IRQ fiq not found
[    3.070254] fiq_debugger fiq_debugger.0: IRQ wakeup not found
[    3.070779] fiq_debugger_probe: could not install nmi irq handler
[ [    3.071389] printk: console [ttyFIQ0] enabled
   3.071389] printk: console [ttyFIQ0] enabled
[    3.072165] printk: bootconsole [uart8250] disabled
[    3.072165] printk: bootconsole [uart8250] disabled
[    3.072697] Registered fiq debugger ttyFIQ0
```

debug why that is. Update: didn't solve it, went a different direction instead

TODO flesh out the initramfs, get a rootfs booting
- probably want to print available /dev/mmc devices and figure out which is our sdcard
  using things like lsblk, and the partition labels


this is likely why we aren't seeing the sdcard:
```
1.128960]  mmcblk0: p1 p2 p3 p4 p5 p6 p7
[    1.130311] rockchip-pinctrl pinctrl: pin gpio4-24 already requested by feb50000.serial; cannot claim for fe2c0000.mmc
[    1.130725] rockchip-spdif fddb0000.spdif-tx: ignoring dependency for device, assuming no driver
[    1.131125] mmcblk0boot0: mmc0:0001 Y2P128 4.00 MiB
[    1.131245] rockchip-pinctrl pinctrl: pin-152 (fe2c0000.mmc) status -22
[    1.132794] mmcblk0boot1: mmc0:0001 Y2P128 4.00 MiB
[    1.133038] rockchip-pinctrl pinctrl: could not request pin 152 (gpio4-24) from group sdmmc-bus4  on device rockchip-pinctrl
[    1.134021] ALSA device list:
[    1.134444] dwmmc_rockchip fe2c0000.mmc: Error applying setting, reverse things back
```
this gpio is likely related to the above issue issue
```
	{RK_GPIO2_D4, RK3588_EMMC_IOC_REG + 0x005C},
```


	sdmmc: mmc@fe2c0000 {
		compatible = "rockchip,rk3588-dw-mshc", "rockchip,rk3288-dw-mshc";
		reg = <0x0 0xfe2c0000 0x0 0x4000>;
		interrupts = <GIC_SPI 203 IRQ_TYPE_LEVEL_HIGH>;
		clocks = <&scmi_clk SCMI_HCLK_SD>, <&scmi_clk SCMI_CCLK_SD>,
			 <&cru SCLK_SDMMC_DRV>, <&cru SCLK_SDMMC_SAMPLE>;
		clock-names = "biu", "ciu", "ciu-drive", "ciu-sample";
		fifo-depth = <0x100>;
		max-frequency = <200000000>;
		pinctrl-names = "default";
		pinctrl-0 = <&sdmmc_clk &sdmmc_cmd &sdmmc_det &sdmmc_bus4>;
		power-domains = <&power RK3588_PD_SDMMC>;

when plugging the sd card into stock:

```
root@firefly:~# [   65.046083] mmc_host mmc2: Bus speed (slot 0) = 148500000Hz (slot req 150000000Hz, actual 148500000HZ div = 0)
[   65.187228] dwmmc_rockchip fe2c0000.mmc: Successfully tuned phase to 360
[   65.187472] mmc2: new ultra high speed SDR104 SDHC card at address aaaa
[   65.191981] mmcblk2: mmc2:aaaa SE32G 29.7 GiB
[   65.211362]  mmcblk2: p1 p2
```

nothing happens when plugging an sd card in to our kernel build

debug why the sd card is not getting seen by the kernel

brought in a number of drivers, still not getting there, but we have some good debug info:

```
[    1.201921] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL probing for dw_mci_rockchip
[    1.202563] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL probing for dw_mci_rockchip: use_rpm = false
[    1.203357] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mcp_pltform_register
[    1.204148] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mcp_pltform_register: complete, calling dw_mci_probe
[    1.204996] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_probe
[    1.205515] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_part_dt
[    1.206085] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_part_dt: complete
[    1.206850] dwmmc_rockchip fe2c0000.mmc: IDMAC supports 32-bit address mode.
[    1.207498] dwmmc_rockchip fe2c0000.mmc: Using internal DMA controller.
[    1.208091] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_probe: checkpoint 3
[    1.208716] dwmmc_rockchip fe2c0000.mmc: Version ID is 270a
[    1.209244] dwmmc_rockchip fe2c0000.mmc: DW MMC controller at irq 43,32 bit host data width,256 deep fifo
[    1.210121] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_init_slot
[    1.210835] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL regulator_dev_lookup supply vqmmc: RETURN EPROBE_DEFER
[    1.211673] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL _regulator_get supply vqmmc: lookup failed! ret = -517
[    1.212512] dwmmc_rockchip fe2c0000.mmc: No vmmc regulator found
[    1.213045] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL: mmc_regulator_get_supply: error: supply.vqmmc is -EPROBE_DEFER
[    1.213997] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_init_slot: mmc_regulator_get_supply failed: ret = -517
[    1.214887] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_init_slot: erro_host_allocated: ret = -517
[    1.215825] dwmmc_rockchip fe2c0000.mmc: slot 1 init failed
[    1.216327] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_probe: err_dmaunmap
[    1.217000] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL dw_mci_probe: returning error: -517
[    1.217695] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL probing for dw_mci_rockchip: error registering: -517
```

the lookup for the vqmmc regulator fails with:

```
[    1.210835] dwmmc_rockchip fe2c0000.mmc: SOLIDHAL regulator_dev_lookup supply vqmmc: RETURN EPROBE_DEFER
```

this is in regulator/core.c
```
 * -ENODEV if lookup fails permanently, -EPROBE_DEFER if lookup could succeed
 * in the future.
```

```
			/*
			 * We have a node, but there is no device.
			 * assume it has not registered yet.
			 */
```

this is the dtsi entry
```
&sdmmc {
	max-frequency = <150000000>;
	no-sdio;
	no-mmc;
	bus-width = <4>;
	cap-mmc-highspeed;
	cap-sd-highspeed;
	disable-wp;
	sd-uhs-sdr104;
	vqmmc-supply = <&vccio_sd_s0>;
	status = "disabled";
};
```

the regulator is defined in rk3588-rk806-single.dtsi
```
			vccio_sd_s0: PLDO_REG5 {
				regulator-always-on;
				regulator-boot-on;
				regulator-min-microvolt = <1800000>;
				regulator-max-microvolt = <3300000>;
				regulator-name = "vccio_sd_s0";
				regulator-state-mem {
					regulator-off-in-suspend;
				};
			};
```

PLD0_REG5 is defined in rk806-regulator.c
```
drivers/regulator/rk806-regulator.c
```
which is getting built into the kernel

these drivers are of interest
```
drivers/mfd/rk806-core.c
drivers/regulator/rk806-regulator.c
```

they both were copied in from the itx tree

turns out its because we didn't have the rk806 spi driver.

now have the kernel booting into a rootfs, with networking up.


## Upstream Uboot notes










