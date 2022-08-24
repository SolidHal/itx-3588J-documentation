
Some useful information for liberating the itx-3588J board


#TODO
- SD firmware upgrade layout
- SD boot layout
- dump wiki pages from https://wiki.t-firefly.com/en/Core-3588J/index.html
- make tool to create SD boot&upgrade images from linux



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
(recovery??)
uboot spl (miniloaderall)
uboot tpl ?? CONFIG_SUPPORT_TPL=y
uboot
extlinux
linux

TODO: 
- make uboot output information
- make recovery output information
- make extlinux output information


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

build packs some inis with the u-boot image
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
