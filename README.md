# Deegle Linux Boot And Firmware

This repo is a playground for the startup firmware used by BeagleBone Black and PocketBeagle 2.

## Setup the repo

This repoistory makes use of submodules.
You can clone it using `git clone --recurse-submodules <URL>`,
or run  `git submodule update --init --recursive` after cloning.

## Build the cross-compile toolchains

For preparing tuned cross-compile toolchains `crosstool-ng` is used.
Execute the following steps to build and install `crosstool-ng`:

- Change into the `crosstool-ng` submodule: `cd crosstool-ng`
- Generate configure script: `./bootstrap`
- Configure: `./configure`
- Build: `make`
- Install: `sudo make install`
- Check installation: `ct-ng --version`

### Build the BeagleBone Black toolchain

- Change into the config folder: `cd bbb/ctng`
- Review the config: `ct-ng menuconfig`
- Build and install the toolchain (using 16 threads): `ct-ng build.16`

The toolchain gets installed into `~/x-tools`. To select the toolchain run:

- `export PATH="${PATH}:/home/tom/x-tools/arm-cortex_a8-linux-gnueabihf/bin"`
- `export ARCH="arm"`
- `export CROSS_COMPILE="arm-cortex_a8-linux-gnueabi-"`

### Build the PocketBeagle 2 toolchain

- Change into the config folder: `cd pocketbeagle/ctng`
- Review the config: `ct-ng menuconfig`
- Build and install the toolchain (using 16 threads): `ct-ng build.16`

The toolchain gets installed into `~/x-tools`. To select the toolchain run:

- `export PATH="${PATH}:/home/tom/x-tools/aarch64-unknown-linux-gnu/bin"`
- `export ARCH="aarch64"`
- `export CROSS_COMPILE="aarch64-unknown-linux-gnu-"`

#### Build the 32bit toolchain for the firmware

- Change into the config folder: `cd pocketbeagle/ctng32`
- Review the config: `ct-ng menuconfig`
- Build and install the toolchain (using 16 threads): `ct-ng build.16`

The toolchain gets installed into `~/x-tools`. To select the toolchain run:

- `export PATH="${PATH}:/home/tom/x-tools/armv7-AM6362-eabihf/bin"`
- `export ARCH="aarch64"`
- `export CROSS_COMPILE="armv7-AM6362-eabihf-"`
- `export CROSS_COMPILE64="aarch64-unknown-linux-gnu-"`

### Bash aliases

For more comfort, you can add the following aliases to your `~/.bashrc`

```bash
export PATH="${PATH}:/home/tom/x-tools/arm-cortex_a8-linux-gnueabihf/bin"
export PATH="${PATH}:/home/tom/x-tools/aarch64-unknown-linux-gnu/bin"
export PATH="${PATH}:/home/tom/x-tools/armv7-rpi2-linux-gnueabihf/bin"

alias tc_bbb=" \
        export ARCH=\"arm\"; \
        export CROSS_COMPILE=\"arm-cortex_a8-linux-gnueabihf-\" \

alias tc_pb2=" \
        export ARCH=\"aarch64\"; \
        export CROSS_COMPILE=\"aarch64-unknown-linux-gnu-\" \

alias tc_info=" \
	env | grep \"ARCH\" ; \
	env | grep \"CROSS_COMPILE\" ; \
	which \"${CROSS_COMPILE}gcc\" ; "

```

## BeagleBone Black

### U-Boot

Build the U-Boot bootloader for BeagleBone Black:

- Change to upstream U-Boot dir: `cd bbb/u-boot`
- Enable the BeagleBone Black toolchain: `tc_bbb; tc_info`
- Select the BeagleBone Black config: `make am335x_evm_defconfig`
- Build U-Boot (using 16 threads): `make -j16`

### Kernel

- Change to Linux dir: `cd linux`
- Enable the BeagleBone Black toolchain: `tc_bbb; tc_info`
- Cleanup old buld artifacts: `make mrproper`
- Select a BeagleBone Black compatible config: `make multi_v7_defconfig`
- Tune the config: `make menuconfig`
- Build the kernel (using 16 threads): `make -j16 zImage dtbs modules`

### Build and SD card image

Partition SD card:

- `sudo parted <device> mklabel msdos`
- `sudo parted <device> mkpart primary fat32 4 132`
- `sudo mkfs.fat /dev/sdf1`
- `sudo mount <device> /mnt`
- `sudo cp ./bbb/u-boot/u-boot.img ./bbb/u-boot/MLO /mnt/`

The result should look like:

```bash
tom@frame:~/sandbox/deegle_boot$ ls -lah /mnt/
insgesamt 1,7M
drwxr-xr-x  2 root root  16K  1. Jan 1970  .
drwxr-xr-x 19 root root 4,0K 14. Dez 12:20 ..
-rwxr-xr-x  1 root root 108K 15. Dez 10:31 MLO
-rwxr-xr-x  1 root root 1,5M 15. Dez 10:31 u-boot.img`
```

### Test the image

Add user to dialout group:

```bash
tom@frame:~$ sudo usermod -aG dialout tom
tom@frame:~$ newgrp dailout
```

Open serial terminal, e.g. gtkterm, and the serial interface, e.g. /dev/ttyUSB0.
Interrupt boot to enter U-boot console:

```
U-Boot SPL 2025.10 (Dec 15 2025 - 10:46:16 +0100)
Trying to boot from MMC1


U-Boot 2025.10 (Dec 15 2025 - 10:46:16 +0100)

CPU  : AM335X-GP rev 2.1
Model: TI AM335x BeagleBone Black
DRAM:  512 MiB
Core:  161 devices, 18 uclasses, devicetree: separate
WDT:   Started wdt@44e35000 with servicing every 1000ms (60s timeout)
NAND:  0 MiB
MMC:   OMAP SD/MMC: 0, OMAP SD/MMC: 1
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1... 
<ethaddr> not set. Validating first E-fuse MAC
Net:   eth2: ethernet@4a100000using musb-hdrc, OUT ep1out IN ep1in STATUS ep2in
MAC de:ad:be:ef:00:01
HOST MAC de:ad:be:ef:00:00
RNDIS ready
, eth3: usb_ether
Hit any key to stop autoboot: 0
=> 
```

### Network boot

Install TFTP server:

```bash
sudo apt install tftpd-hpa
``` 

Check config: `cat /etc/default/tftpd-hpa`

Prepare folder for TFTP server:

```bash
sudo mkdir -p /srv/tftp
sudo chown -R root:tom /srv/tftp
sudo chmod -R 775 /srv/tftp
```

Allow port 69 on firewall:

```bash
tom@frame:~/sandbox/deegle_boot$ sudo ufw allow 69
Rule added
Rule added (v6)
tom@frame:~/sandbox/deegle_boot$ sudo ufw enable
Firewall is active and enabled on system startup
tom@frame:~/sandbox/deegle_boot$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                  
69                         ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6)             
69 (v6)                    ALLOW       Anywhere (v6)       
```

Copy kernel and device tree to TFTP folder:

- `cp linux/arch/arm/boot/zImage /srv/tftp/zImage_bbb`
- `cp ./linux/arch/arm/boot/dts/ti/omap/am335x-boneblack.dtb /srv/tftp/`

Install NFS server: `sudo apt install nfs-kernel-server nfs-common`

Prepare folder for TFTP server:

```bash
sudo mkdir -p /srv/nfs/bbb
sudo chown -R root:tom /srv/nfs/bbb
sudo chmod -R 775 /srv/nfs/bbb
```

Configure NFS exports:

```bash
tom@frame:~/sandbox/deegle_boot$ cat /etc/exports
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/srv/nfs        192.168.42.0/24(rw,sync,no_root_squash,no_subtree_check,crossmnt)
```

Open firewall and restart NFS server:

```bash
tom@frame:~/sandbox/deegle_boot$ sudo ufw allow nfs
tom@frame:~/sandbox/deegle_boot$ sudo ufw enable
tom@frame:~/sandbox/deegle_boot$ sudo systemctl restart nfs-server
```

Get your hosts IP address and connect BeagleBone Black to ethernet.

Enter U-Boot console and:

```
setenv bootargs 'root=/dev/nfs rw ip=dhcp console=ttyO0,115200n8 nfsroot=192.168.42.168:/srv/nfs/bbb,nfsvers=3'
setenv bootcmd 'setenv autoload no; dhcp; setenv serverip 192.168.42.168; tftp 0x81000000 zImage_bbb; tftp 0x82000000 am335x-boneblack.dtb; bootz 0x81000000 - 0x82000000'
saveenv
boot
```

The output should be:

```
=> setenv bootargs 'root=/dev/nfs rw ip=dhcp console=ttyO0,115200n8 nfsroot=192.168.42.168:/srv/nfs/bbb,nfsvers=3'
=> setenv bootcmd 'setenv autoload no; dhcp; setenv serverip 192.168.42.168; tftp 0x81000000 zImage_bbb; tftp 0x82000000 am335x-boneblack.dtb; bootz 0x81000000 - 0x82000000'
=> saveenv
Saving Environment to FAT... OK
=> boot
link up on port 0, speed 100, full duplex
BOOTP broadcast 1
DHCP client bound to address 192.168.42.38 (6 ms)
link up on port 0, speed 100, full duplex
Using ethernet@4a100000 device
TFTP from server 192.168.42.168; our IP address is 192.168.42.38
Filename 'zImage_bbb'.
Load address: 0x81000000
Loading: ##################################################  11.4 MiB
	 2.9 MiB/s
done
Bytes transferred = 11973120 (b6b200 hex)
link up on port 0, speed 100, full duplex
Using ethernet@4a100000 device
TFTP from server 192.168.42.168; our IP address is 192.168.42.38
Filename 'am335x-boneblack.dtb'.
Load address: 0x82000000
Loading: ##################################################  68.8 KiB
	 2.3 MiB/s
done
Bytes transferred = 70448 (11330 hex)
Kernel image @ 0x81000000 [ 0x000000 - 0xb6b200 ]
## Flattened Device Tree blob at 82000000
   Booting using the fdt blob at 0x82000000
Working FDT set to 82000000
   Loading Device Tree to 8ffeb000, end 8ffff32f ... OK
Working FDT set to 8ffeb000

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
...
```

At the very end, we will run into a kernel panic since we have no root filesystem:

```
[    6.191857] VFS: Mounted root (nfs filesystem) on device 0:16.
[    6.199404] devtmpfs: error mounting -2
[    6.214713] Freeing unused kernel image (initmem) memory: 2048K
[    6.224072] Run /sbin/init as init process
[    6.230402] Run /etc/init as init process
[    6.236866] Run /bin/init as init process
[    6.242327] Run /bin/sh as init process
[    6.246390] Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance.
[    6.260639] CPU: 0 UID: 0 PID: 1 Comm: swapper/0 Not tainted 6.18.1 #1 NONE 
[    6.267730] Hardware name: Generic AM33XX (Flattened Device Tree)
[    6.273853] Call trace: 
[    6.273873]  unwind_backtrace from show_stack+0x10/0x14
[    6.281698]  show_stack from dump_stack_lvl+0x54/0x68
[    6.286790]  dump_stack_lvl from vpanic+0xc4/0x2ec
[    6.291616]  vpanic from __do_trace_suspend_resume+0x0/0x48
[    6.297243] ---[ end Kernel panic - not syncing: No working init found.  Try passing init= option to kernel. See Linux Documentation/admin-guide/init.rst for guidance. ]---
```

### Minimal root filesystem

- Copy libs and linker: `cp -R ~/x-tools/arm-cortex_a8-linux-gnueabihf/arm-cortex_a8-linux-gnueabihf/sysroot/* /srv/nfs/bbb`

#### Build busybox

- `cd busybox`
- Enable toolchain: `tc_bbb; tc_info`
- `make defconfig`

If needed, fix menuconfig, see `https://unix.stackexchange.com/questions/790203/make-menuconfig-fails-while-working-on-busybox-only`.

- `make -j16`
- `make install CONFIG_PREFIX=/srv/nfs/bbb/`


#### Finalize root filesystem

- Create missing folder: `mkdir -p /srv/nfs/bbb/{etc,etc/init.d,proc,sys,dev,tmp,root,var,lib,mnt,boot}`
- Install kernel modules: `make INSTALL_MOD_PATH=/srv/nfs/bbb/ modules_install`

Then reboot the device. The serial output should look like:

```bash
[    8.240334] Freeing unused kernel image (initmem) memory: 2048K
[    8.247786] Run /sbin/init as init process
[    8.254929] Run /etc/init as init process
[    8.261041] Run /bin/init as init process
[    8.267860] Run /bin/sh as init process
/bin/sh: can't access tty; job control turned off
~ # 
```

## PocketBeagle 2

![PocketBeagle2 Boot Flow](boot_diagram_am62.svg)

For more details of the PocketBeagle boot flow, see https://docs.u-boot.org/en/v2025.01/board/ti/am62x_sk.html.

## Setup environment

The necessary environment variables for building the firmware are configured in `pocketbeagle/build_env`
and should look like: 

```bash
export CC32="armv7-rpi2-linux-gnueabihf"
export CC64="aarch64-unknown-linux-gnu"

export REPO_BASE="/home/tom/sandbox/deegle_boot/pocketbeagle"

export LNX_FW_PATH="${REPO_BASE}/ti-linux-firmware"
export TFA_PATH="${REPO_BASE}/trusted-firmware-a"
export OPTEE_PATH="${REPO_BASE}/optee_os"

export UBOOT_CFG_CORTEXR="am62_pocketbeagle2_r5_defconfig"
export UBOOT_CFG_CORTEXA="am62_pocketbeagle2_a53_defconfig"
export TFA_BOARD="lite"

# we dont use any extra TFA parameters
unset TFA_EXTRA_ARGS

export OPTEE_PLATFORM="k3-am62x"
export OPTEE_EXTRA_ARGS="CFG_WITH_SOFTWARE_PRNG=y"

```

### Build ARM trusted firmware

- `cd ./pocketbeagle/trusted-firmware-a/`
- `source ../build_env`
- `make -j16 CROSS_COMPILE=$CC64 ARCH=aarch64 PLAT=k3 SPD=opteed $TFA_EXTRA_ARGS TARGET_BOARD=$TFA_BOARD K3_USART=0x6 all`

### Build OpTEE

- `cd ./pocketbeagle/optee_os/`
- `source ../build_env`
- `source ../venv/bin/activate`
- `pip install -r ../pyhton_tools.txt`
- `make -j16 CROSS_COMPILE=$CC32 CROSS_COMPILE64=$CC64 CFG_ARM64_core=y $OPTEE_EXTRA_ARGS PLATFORM=$OPTEE_PLATFORM CFG_CONSOLE_UART=0x6 all`

### Build U-Boot

#### U-Boot for R5

- `cd ./pocketbeagle/optee_os/`
- `sudo apt install -y bc bison flex libgnutls28-dev libssl-dev swig uuid-dev yamllint`
- `source ../build_env`
- `source ../venv/bin/activate`
- `make -j16 $UBOOT_CFG_CORTEXR`
- `make -j16 CROSS_COMPILE=$CC32 BINMAN_INDIRS=$LNX_FW_PATH`

#### U-Boot for A53

- `cd ./pocketbeagle/optee_os/`
- `sudo apt install -y bc bison flex libgnutls28-dev libssl-dev swig uuid-dev yamllint`
- `source ../build_env`
- `source ../venv/bin/activate`
- `make -j16 $UBOOT_CFG_CORTEXA`
- `make -j16 CROSS_COMPILE=$CC64 BINMAN_INDIRS=$LNX_FW_PATH BL31=$TFA_PATH/build/k3/$TFA_BOARD/release/bl31.bin TEE=$OPTEE_PATH/out/arm-plat-k3/core/tee-raw.bin`

### Kernel

- Change to Linux dir: `cd linux`
- Enable the BeagleBone Black toolchain: `tc_pb2; tc_info`
- Set kernel arch: `export ARCH="arm64"`
- Check toolchain config: `tc_info`
- Cleanup old buld artifacts: `make mrproper`
- Select a BeagleBone Black compatible config: `make defconfig`
- Tune the config: `make menuconfig`
- Build the kernel (using 16 threads): `make -j16 Image dtbs modules`

### Build and SD card image

Partition SD card:

- `sudo parted <device> mklabel msdos`
- `sudo parted <device> mkpart primary fat32 1 129`
- `sudo mkfs.fat <device>`
- `sudo mount <device> /mnt`


- `sudo cp sudo cp u-boot/tispl.bin /mnt/`
- `sudo cp u-boot/u-boot.img /mnt/`
- `sudo cp u-boot/tiboot3-am62x-hs-fs-evm.bin /mnt/tiboot.bin`
- `sudo cp ../linux/arch/arm64/boot/dts/ti/k3-am62-pocketbeagle2.dtb /mnt/`
- `sudo cp extlinux.conf /mnt/`

The result should look like:

```bash
tom@frame:~/sandbox/deegle_boot/pocketbeagle$ ls -lah /mnt/
insgesamt 2,1M
drwxr-xr-x  2 root root  16K  1. Jan 1970  .
drwxr-xr-x 19 root root 4,0K 14. Dez 12:20 ..
-rwxr-xr-x  1 root root  199 15. Dez 15:00 extlinux.conf
-rwxr-xr-x  1 root root  54K 15. Dez 15:00 k3-am62-pocketbeagle2.dtb
-rwxr-xr-x  1 root root 256K 15. Dez 14:58 tiboot.bin
-rwxr-xr-x  1 root root 908K 15. Dez 14:57 tispl.bin
-rwxr-xr-x  1 root root 868K 15. Dez 14:58 u-boot.img
```

Remove the SD card: `sync; sudo umount /mnt`

### Test the image

TODO: How to get early serial logs?

### Network boot

TODO

### Minimal root filesystem

- Create folder: `mkdir -p /srv/nfs/pb2`
- Copy libs and linker: `cp -R ~/x-tools/aarch64-unknown-linux-gnu/aarch64-unknown-linux-gnu/sysroot/* /srv/nfs/pb2`

#### Build busybox

- `cd busybox`
- `make mrproper`
- Enable toolchain: `tc_pb2; tc_info`
- `make defconfig`

- If needed, fix menuconfig, see `https://unix.stackexchange.com/questions/790203/make-menuconfig-fails-while-working-on-busybox-only`.
- If build issue: `make menuconfig` and disable tc network tool

- `make -j16`
- `sudo make install CONFIG_PREFIX=/srv/nfs/pb2/`


#### Finalize root filesystem

- Create missing folder: `mkdir -p /srv/nfs/pb2/{etc,etc/init.d,proc,sys,dev,tmp,root,var,lib,mnt,boot}`
- Install kernel modules: `sudo make INSTALL_MOD_PATH=/srv/nfs/pb2/ modules_install`

Then reboot the device. The serial output should look like:

```bash
```


