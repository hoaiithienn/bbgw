# Ref:
- https://forum.digikey.com/t/debian-getting-started-with-the-beaglebone-black/12967
- https://weworkweplay.com/play/automatically-connect-a-raspberry-pi-to-a-wifi-network/

# ARM Cross Compiler: GCC

- This is a pre-built (64bit) version of GCC that runs on generic linux, sorry (32bit) x86 users, it’s time to upgrade…

- Download/Extract:
```
#user@localhost:~$
wget -c https://mirrors.edge.kernel.org/pub/tools/crosstool/files/bin/x86_64/11.3.0/x86_64-gcc-11.3.0-nolibc-arm-linux-gnueabi.tar.xz
tar -xf x86_64-gcc-11.3.0-nolibc-arm-linux-gnueabi.tar.xz
export CC=`pwd`/gcc-11.3.0-nolibc/arm-linux-gnueabi/bin/arm-linux-gnueabi-
```

- Test Cross Compiler:
```
#user@localhost:~$
${CC}gcc --version

#Test Output:
arm-linux-gnueabi-gcc (GCC) 11.3.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

# Bootloader: U-Boot

- Das U-Boot – the Universal Boot Loader: http://www.denx.de/wiki/U-Boot 235
- Depending on your Linux Distribution, you will also need a host gcc and other tools, so with Debian/Ubuntu start with installing the build-essential meta package.
```
#Requirements
sudo apt install bison build-essential flex swig
```

- Download:
```
#user@localhost:~$
git clone -b v2022.04 https://github.com/u-boot/u-boot --depth=1
cd u-boot/
```

- Patches:
```
#user@localhost:~/u-boot$
git pull --no-edit https://git.beagleboard.org/beagleboard/u-boot.git v2022.04-bbb.io-am335x-am57xx
```

- Configure and Build:
```
#user@localhost:~/u-boot$
make -j8 ARCH=arm CROSS_COMPILE=${CC} distclean
make -j8 ARCH=arm CROSS_COMPILE=${CC} am335x_evm_defconfig
make -j8 ARCH=arm CROSS_COMPILE=${CC}
```

# Linux Kernel

- This script will build the kernel, modules, device tree binaries and copy them to the deploy directory.

- Download TI BSP:
```
#~/
git clone https://github.com/RobertCNelson/ti-linux-kernel-dev ./kernelbuildscripts
cd kernelbuildscripts/
```

- For TI v5.10.x:
```
#~/kernelbuildscripts/
git checkout origin/ti-linux-5.10.y -b tmp
```

- Utilize multi-core processor to compile kernel faster by edit **build_kernel.sh** script and add **-j8** to all **make** commands.

- Build:
```
#user@localhost:~/kernelbuildscripts$
./build_kernel.sh
```

# Root File System
- Debian 11 user and its password: **debian:temppwd** or **root:root**

- Download debian:
```
#user@localhost:~$
wget -c https://rcn-ee.com/rootfs/eewiki/minfs/debian-11.5-minimal-armhf-2022-10-06.tar.xz
```

- Verify:
```
#user@localhost:~$
sha256sum debian-11.5-minimal-armhf-2022-10-06.tar.xz
#sha256sum output:
0ea53b23483af4bcccca53f3fd907b5e404f2f607a6577d78e8c4daf5b869ad0  debian-11.5-minimal-armhf-2022-10-06.tar.xz
```

- Extract:
```
#user@localhost:~$
tar xf debian-11.5-minimal-armhf-2022-10-06.tar.xz
```

# Setup microSD card: 

- We need to access the External Drive to be utilized by the target device. Run lsblk to help figure out what linux device has been reserved for your External Drive.
```
#Example: for DISK=/dev/sdX
lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 465.8G  0 disk
├─sda1   8:1    0   512M  0 part /boot/efi
└─sda2   8:2    0 465.3G  0 part /                <- Development Machine Root Partition
sdb      8:16   1   962M  0 disk                  <- microSD/USB Storage Device
└─sdb1   8:17   1   961M  0 part                  <- microSD/USB Storage Partition
#Thus you would use:
export DISK=/dev/sdb
```
```
#Example: for DISK=/dev/mmcblkX
lsblk
NAME      MAJ:MIN   RM   SIZE RO TYPE MOUNTPOINT
sda         8:0      0 465.8G  0 disk
├─sda1      8:1      0   512M  0 part /boot/efi
└─sda2      8:2      0 465.3G  0 part /                <- Development Machine Root Partition
mmcblk0     179:0    0   962M  0 disk                  <- microSD/USB Storage Device
└─mmcblk0p1 179:1    0   961M  0 part                  <- microSD/USB Storage Partition
#Thus you would use:
export DISK=/dev/mmcblk0
```

- Erase partition table/labels on microSD card:
```
sudo dd if=/dev/zero of=${DISK} bs=1M count=10
```

- Install Bootloader:
```
#user@localhost:~$
sudo dd if=./u-boot/MLO of=${DISK} count=2 seek=1 bs=128k
sudo dd if=./u-boot/u-boot-dtb.img of=${DISK} count=4 seek=1 bs=384k
```

- Create Partition Layout: With util-linux v2.26, sfdisk was rewritten and is now based on libfdisk.
```
#Check the version of sfdisk installed on your pc is atleast 2.26.x or newer.
sudo sfdisk --version
#Example Output
sfdisk from util-linux 2.27.1
```
```
#sfdisk >= 2.26.x
sudo sfdisk ${DISK} <<-__EOF__
4M,,L,*
__EOF__
```

- Format Partition: With mkfs.ext4 1.43, we need to make sure metadata_csum and 64bit ext4 features are disabled. As the version of U-Boot needed for this target CAN NOT correctly handle reading files with these newer ext4 options.
```
#mkfs.ext4 -V
sudo mkfs.ext4 -V
mke2fs 1.43-WIP (15-Mar-2016)
        Using EXT2FS Library version 1.43-WIP
#mkfs.ext4 >= 1.43
```
```
#for: DISK=/dev/mmcblkX
sudo mkfs.ext4 -L rootfs -O ^metadata_csum,^64bit ${DISK}p1
```
``` 
#for: DISK=/dev/sdX
sudo mkfs.ext4 -L rootfs -O ^metadata_csum,^64bit ${DISK}1
```

- Mount Partition: On most systems these partitions may be auto-mounted…
```
sudo mkdir -p /media/rootfs/
 
for: DISK=/dev/mmcblkX
sudo mount ${DISK}p1 /media/rootfs/
 
for: DISK=/dev/sdX
sudo mount ${DISK}1 /media/rootfs/
```

# Copy images to SD card:

- Backup Bootloader: This version of MLO/u-boot.img will be used on the “eMMC” flasher script on this page.
```
#~/
sudo mkdir -p /media/rootfs/opt/backup/uboot/
sudo cp -v ./u-boot/MLO /media/rootfs/opt/backup/uboot/
sudo cp -v ./u-boot/u-boot-dtb.img /media/rootfs/opt/backup/uboot/
```

- Install Kernel and Root File System: To help new users, since the kernel version can change on a daily basis. The kernel building scripts listed on this page will now give you a hint of what kernel version was built.

```
-----------------------------
Script Complete
eewiki.net: [user@localhost:~$ export kernel_version=5.X.Y-Z]
-----------------------------
```

- Copy and paste that “export kernel_version=5.X.Y-Z” exactly as shown in your own build/desktop environment and hit enter to create an environment variable to be used later.
```
export kernel_version=5.X.Y-Z
```

- Copy Root File System
```
#Debian; Root File System: user@localhost:~$
sudo tar xfvp ./debian-*-*-armhf-*/armhf-rootfs-*.tar -C /media/rootfs/
sync
```

- Set uname_r in /boot/uEnv.txt
```
#user@localhost:~$
sudo sh -c "echo 'uname_r=${kernel_version}' >> /media/rootfs/boot/uEnv.txt"
```

- Copy Kernel Image
```
#user@localhost:~$
sudo cp -v ./kernelbuildscripts/deploy/${kernel_version}.zImage /media/rootfs/boot/vmlinuz-${kernel_version}
```

- Copy Kernel Device Tree Binaries
```
#user@localhost:~$
sudo mkdir -p /media/rootfs/boot/dtbs/${kernel_version}/
sudo tar xfv ./kernelbuildscripts/deploy/${kernel_version}-dtbs.tar.gz -C /media/rootfs/boot/dtbs/${kernel_version}/
```

- Copy Kernel Modules
```
#user@localhost:~$
sudo tar xfv ./kernelbuildscripts/deploy/${kernel_version}-modules.tar.gz -C /media/rootfs/
```

- File Systems Table (/etc/fstab)
```
#user@localhost:~/$
sudo sh -c "echo '/dev/mmcblk0p1  /  auto  errors=remount-ro  0  1' >> /media/rootfs/etc/fstab"
```

- Networking: Edit **/etc/network/interfaces** and add **/etc/wpa_supplicant/wpa_supplicant.conf**


**/etc/network/interfaces**: 
```
#user@localhost:~/$
nano /media/rootfs/etc/network/interfaces
```
```
#/etc/network/interfaces
# interfaces(5) file used by ifup(8) and ifdown(8)
# Include files from /etc/network/interfaces.d:
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

## Setup wifi 
allow-hotplug wlan0
iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
iface default inet dhcp

## Static IP
address 192.168.1.11  # Static IP you want
netmask 255.255.255.0
gateway 192.168.1.1   # IP of your router
```

**/etc/wpa_supplicant/wpa_supplicant.conf**
```
#user@localhost:~/$
nano /media/rootfs/etc/wpa_supplicant/wpa_supplicant.conf
```
```
#/etc/wpa_supplicant/wpa_supplicant.conf
network={
        ssid="SSID_NAME"
        psk="SSID_PASSWORD"
        }
```

- U-Boot Overlays: Add the following lines to **/boot/uEnv.txt**:
```
#user@localhost:~/$
nano /media/rootfs/boot/uEnv.txt
```
**/boot/uEnv.txt**:

```
#Edit kernel version accordingly
uname_r=5.10.168-ti-r72

#Enable overlays:
enable_uboot_overlays=1

#Enable Cape Universal
enable_uboot_cape_universal=1
```

- Remove microSD/SD card
```
sync
sudo umount /media/rootfs
```

# Create disk image file:

- Use "Create Disk Image" function of Ubuntu Disk tool

- Compress disk image on host computer:

```
#user@localhost:~/$
wget https://raw.githubusercontent.com/Drewsif/PiShrink/master/pishrink.sh
chmod +x pishrink.sh
sudo ./pishrink.sh -aZ src.img dst.img
```

- To write image to microSD card, use "Restore Disk Image" and "Resize" tool of Ubuntu Disk tool. 

# Beaglebone Green Wireless booting up process:

- Setup UART FTDI USB: Logic level 3V3
    - BBGW TX <-> USB RX
    - BBGW RX <-> USB TX
    - BBGW GND <-> USB GND

- Insert microSD card to Beaglebone Green Wireless and power on

- Sample booting up log:

<details>
<summary>Click to expand</summary>

```
U-Boot SPL 2022.04-ge0d31da5 (Aug 04 2023 - 18:48:26 +0000)
Trying to boot from MMC2


U-Boot 2022.04-ge0d31da5 (Aug 04 2023 - 18:48:26 +0000)

CPU  : AM335X-GP rev 2.1
Model: TI AM335x BeagleBone Black
DRAM:  512 MiB
Reset Source: Global warm SW reset has occurred.
Reset Source: Power-on reset has occurred.
RTC 32KCLK Source: External.
Core:  150 devices, 14 uclasses, devicetree: separate
WDT:   Started wdt@44e35000 with servicing (60s timeout)
MMC:   OMAP SD/MMC: 0, OMAP SD/MMC: 1
Loading Environment from EXT4... ** File not found /boot/uboot.env **

** Unable to read "/boot/uboot.env" from mmc0:1 **
Board: BeagleBone Black
<ethaddr> not set. Validating first E-fuse MAC
BeagleBone Black:
Model: SeeedStudio BeagleBone Green Wireless:
BeagleBone Cape EEPROM: no EEPROM at address: 0x54
BeagleBone Cape EEPROM: no EEPROM at address: 0x55
BeagleBone Cape EEPROM: no EEPROM at address: 0x56
BeagleBone Cape EEPROM: no EEPROM at address: 0x57
Net:   Could not get PHY for ethernet@4a100000: addr 0
eth2: ethernet@4a100000, eth3: usb_ether
Press SPACE to abort autoboot in 0 seconds
board_name=[A335BNLT] ...
board_rev=[GW1A] ...
switch to partitions #0, OK
mmc0 is current device
SD/MMC found on device 0
Couldn't find partition 0:2 0x82000000
Can't set block device
Couldn't find partition 0:2 0x82000000
Can't set block device
switch to partitions #0, OK
mmc0 is current device
Scanning mmc 0:1...
libfdt fdt_check_header(): FDT_ERR_BADMAGIC
Scanning disk mmc@48060000.blk...
Scanning disk mmc@481d8000.blk...
Found 4 disks
No EFI system partition
BootOrder not defined
EFI boot manager: Cannot load any image
gpio: pin 56 (gpio 56) value is 0
gpio: pin 55 (gpio 55) value is 0
gpio: pin 54 (gpio 54) value is 0
gpio: pin 53 (gpio 53) value is 1
switch to partitions #0, OK
mmc0 is current device
gpio: pin 54 (gpio 54) value is 1
Checking for: /uEnv.txt ...
Checking for: /boot/uEnv.txt ...
gpio: pin 55 (gpio 55) value is 1
79 bytes read in 3 ms (25.4 KiB/s)
Loaded environment from /boot/uEnv.txt
Checking if uname_r is set in /boot/uEnv.txt...
gpio: pin 56 (gpio 56) value is 1
Running uname_boot ...
loading /boot/vmlinuz-5.10.168-ti-r72 ...
11350528 bytes read in 717 ms (15.1 MiB/s)
debug: [enable_uboot_overlays=1] ...
debug: [enable_uboot_cape_universal=1] ...
debug: [uboot_base_dtb_univ=am335x-bonegreen-wireless-uboot-univ.dtb] ...
uboot_overlays: [uboot_base_dtb=am335x-bonegreen-wireless-uboot-univ.dtb] ...
uboot_overlays: Switching too: dtb=am335x-bonegreen-wireless-uboot-univ.dtb ...
loading /boot/dtbs/5.10.168-ti-r72/am335x-bonegreen-wireless-uboot-univ.dtb ...
192303 bytes read in 19 ms (9.7 MiB/s)
Found 0 extension board(s).
uboot_overlays: [fdt_buffer=0x60000] ...
uboot_overlays: loading /boot/dtbs/5.10.168-ti-r72/overlays/BB-ADC-00A0.dtbo ...
645 bytes read in 8 ms (78.1 KiB/s)
uboot_overlays: loading /boot/dtbs/5.10.168-ti-r72/overlays/BB-BONE-eMMC1-01-00A0.dtbo ...
1605 bytes read in 8 ms (195.3 KiB/s)
uboot_overlays: loading /boot/dtbs/5.10.168-ti-r72/overlays/BB-BBGW-WL1835-00A0.dtbo ...
5240 bytes read in 9 ms (568.4 KiB/s)
debug: [console=ttyS0,115200n8 bone_capemgr.uboot_capemgr_enabled=1 root=/dev/mmcblk0p1 ro rootfstype=ext4 rootwait] ...
debug: [bootz 0x82000000 - 88000000] ...
Kernel image @ 0x82000000 [ 0x000000 - 0xad3200 ]
## Flattened Device Tree blob at 88000000
   Booting using the fdt blob at 0x88000000
   Loading Device Tree to 8ff6d000, end 8fffffff ... OK

Starting kernel ...

[    0.000000] Booting Linux on physical CPU 0x0
[    0.000000] Linux version 5.10.168-ti-r72 (thien@DrWatson) (arm-linux-gnueabi-gcc (GCC) 10.5.0, GNU ld (GNU Binutils) 2.40) #1 SMP PREEMPT Sun Oct 29 18:20:44 +07 2023
[    0.000000] CPU: ARMv7 Processor [413fc082] revision 2 (ARMv7), cr=10c5387d
[    0.000000] CPU: PIPT / VIPT nonaliasing data cache, VIPT aliasing instruction cache
[    0.000000] OF: fdt: Machine model: TI AM335x BeagleBone Green Wireless
[    0.000000] Memory policy: Data cache writeback
[    0.000000] efi: UEFI not found.
[    0.000000] cma: Reserved 48 MiB at 0x9c800000
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000080000000-0x000000009fdfffff]
[    0.000000]   HighMem  empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080000000-0x000000009fdfffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080000000-0x000000009fdfffff]
[    0.000000] CPU: All CPU(s) started in SVC mode.
[    0.000000] AM335X ES2.1 (sgx neon)
[    0.000000] percpu: Embedded 21 pages/cpu s54604 r8192 d23220 u86016
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 129412
[    0.000000] Kernel command line: console=ttyS0,115200n8 bone_capemgr.uboot_capemgr_enabled=1 root=/dev/mmcblk0p1 ro rootfstype=ext4 rootwait
[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes, linear)
[    0.000000] mem auto-init: stack:off, heap alloc:on, heap free:off
[    0.000000] Memory: 443896K/522240K available (15360K kernel code, 1448K rwdata, 4040K rodata, 1024K init, 446K bss, 29192K reserved, 49152K cma-reserved, 0K highmem)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=1, Nodes=1
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu:     RCU event tracing is enabled.
[    0.000000] rcu:     RCU restricting CPUs from NR_CPUS=2 to nr_cpu_ids=1.
[    0.000000]  Trampoline variant of Tasks RCU enabled.
[    0.000000]  Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=1
[    0.000000] NR_IRQS: 16, nr_irqs: 16, preallocated irqs: 16
[    0.000000] IRQ: Found an INTC at 0x(ptrval) (revision 5.0) with 128 interrupts
[    0.000000] TI gptimer clocksource: always-on /ocp/interconnect@44c00000/segment@200000/target-module@31000
[    0.000011] sched_clock: 32 bits at 24MHz, resolution 41ns, wraps every 89478484971ns
[    0.000032] clocksource: dmtimer: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 79635851949 ns
[    0.000415] TI gptimer clockevent: 24000000 Hz at /ocp/interconnect@48000000/segment@0/target-module@40000
[    0.002272] Console: colour dummy device 80x30
[    0.002364] Calibrating delay loop... 995.32 BogoMIPS (lpj=1990656)
[    0.048471] pid_max: default: 32768 minimum: 301
[    0.049121] LSM: Security Framework initializing
[    0.049245] Yama: becoming mindful.
[    0.049550] AppArmor: AppArmor initialized
[    0.049572] TOMOYO Linux initialized
[    0.049766] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.049785] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.051461] CPU: Testing write buffer coherency: ok
[    0.051552] CPU0: Spectre v2: using BPIALL workaround
[    0.072896] Setting up static identity map for 0x80100000 - 0x80100060
[    0.080491] rcu: Hierarchical SRCU implementation.
[    0.089738] EFI services will not be available.
[    0.100502] smp: Bringing up secondary CPUs ...
[    0.100524] smp: Brought up 1 node, 1 CPU
[    0.100537] SMP: Total of 1 processors activated (995.32 BogoMIPS).
[    0.100548] CPU: All CPU(s) started in SVC mode.
[    0.101471] devtmpfs: initialized
[    0.140811] VFP support v0.3: implementor 41 architecture 3 part 30 variant c rev 3
[    0.141347] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.141393] futex hash table entries: 256 (order: 2, 16384 bytes, linear)
[    0.145247] pinctrl core: initialized pinctrl subsystem
[    0.146431] DMI not present or invalid.
[    0.147387] NET: Registered protocol family 16
[    0.150576] DMA: preallocated 256 KiB pool for atomic coherent allocations
[    0.190162] l3-aon-clkctrl:0000:0: failed to disable
[    0.190940] audit: initializing netlink subsys (disabled)
[    0.192386] thermal_sys: Registered thermal governor 'fair_share'
[    0.192401] thermal_sys: Registered thermal governor 'bang_bang'
[    0.192419] thermal_sys: Registered thermal governor 'step_wise'
[    0.192429] thermal_sys: Registered thermal governor 'user_space'
[    0.192439] thermal_sys: Registered thermal governor 'power_allocator'
[    0.193383] cpuidle: using governor ladder
[    0.193446] cpuidle: using governor menu
[    0.196593] audit: type=2000 audit(0.180:1): state=initialized audit_enabled=0 res=1
[    5.392880] hw-breakpoint: debug architecture 0x4 unsupported.
[    5.440484] Kprobes globally optimized
[    5.464900] raid6: skip pq benchmark and using algorithm neonx8
[    5.464933] raid6: using neon recovery algorithm
[    5.472607] iommu: Default domain type: Translated 
[    5.476911] SCSI subsystem initialized
[    5.481090] usbcore: registered new interface driver usbfs
[    5.481184] usbcore: registered new interface driver hub
[    5.481247] usbcore: registered new device driver usb
[    5.482516] mc: Linux media interface: v0.10
[    5.482574] videodev: Linux video capture interface: v2.00
[    5.482738] pps_core: LinuxPPS API ver. 1 registered
[    5.482752] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    5.482780] PTP clock support registered
[    5.485362] NetLabel: Initializing
[    5.485390] NetLabel:  domain hash size = 128
[    5.485399] NetLabel:  protocols = UNLABELED CIPSOv4 CALIPSO
[    5.485499] NetLabel:  unlabeled traffic allowed by default
[    5.486643] clocksource: Switched to clocksource dmtimer
[    6.641040] VFS: Disk quotas dquot_6.6.0
[    6.641217] VFS: Dquot-cache hash table entries: 1024 (order 0, 4096 bytes)
[    6.641510] FS-Cache: Loaded
[    6.641928] CacheFiles: Loaded
[    6.643075] AppArmor: AppArmor Filesystem Enabled
[    6.656561] NET: Registered protocol family 2
[    6.656877] IP idents hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    6.658442] tcp_listen_portaddr_hash hash table entries: 512 (order: 0, 6144 bytes, linear)
[    6.658836] TCP established hash table entries: 4096 (order: 2, 16384 bytes, linear)
[    6.658910] TCP bind hash table entries: 4096 (order: 3, 32768 bytes, linear)
[    6.658973] TCP: Hash tables configured (established 4096 bind 4096)
[    6.659591] MPTCP token hash table entries: 512 (order: 1, 8192 bytes, linear)
[    6.659695] UDP hash table entries: 256 (order: 1, 8192 bytes, linear)
[    6.659756] UDP-Lite hash table entries: 256 (order: 1, 8192 bytes, linear)
[    6.660017] NET: Registered protocol family 1
[    6.673480] RPC: Registered named UNIX socket transport module.
[    6.673504] RPC: Registered udp transport module.
[    6.673514] RPC: Registered tcp transport module.
[    6.673524] RPC: Registered tcp NFSv4.1 backchannel transport module.
[    6.673540] NET: Registered protocol family 44
[    6.675620] hw perfevents: enabled with armv7_cortex_a8 PMU driver, 5 counters available
[    6.681458] Initialise system trusted keyrings
[    6.682018] workingset: timestamp_bits=14 max_order=17 bucket_order=3
[    6.690218] zbud: loaded
[    6.698486] NFS: Registering the id_resolver key type
[    6.698571] Key type id_resolver registered
[    6.698585] Key type id_legacy registered
[    6.698987] nfs4filelayout_init: NFSv4 File Layout Driver Registering...
[    6.699010] nfs4flexfilelayout_init: NFSv4 Flexfile Layout Driver Registering...
[    6.699063] jffs2: version 2.2. (NAND) (SUMMARY)  © 2001-2006 Red Hat, Inc.
[    6.700189] fuse: init (API version 7.32)
[    6.789161] NET: Registered protocol family 38
[    6.789195] xor: automatically using best checksumming function   neon      
[    6.789210] Key type asymmetric registered
[    6.789221] Asymmetric key parser 'x509' registered
[    6.789313] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 243)
[    6.793709] io scheduler mq-deadline registered
[    7.520066] ti-sysc: probe of 44e31000.target-module failed with error -16
[    7.648891] omap_i2c 4802a000.i2c: bus 1 rev0.11 at 100 kHz
[    7.675688] ti-sysc: probe of 48040000.target-module failed with error -16
[    7.882535] omap-mailbox 480c8000.mailbox: omap mailbox rev 0x400
[    8.006523] omap_i2c 4819c000.i2c: bus 2 rev0.11 at 100 kHz
[    8.499477] debugfs: Directory '49000000.dma' with parent 'dmaengine' already present!
[    8.499529] edma 49000000.dma: TI EDMA DMA engine driver
[    8.533410] pinctrl-single 44e10800.pinmux: 142 pins, size 568
[    8.535237] gpio-of-helper ocp:cape-universal: Failed to get gpio property of 'P8_03'
[    8.535266] gpio-of-helper ocp:cape-universal: Failed to create gpio entry
[    8.557031] Serial: 8250/16550 driver, 6 ports, IRQ sharing disabled
[    8.561901] printk: console [ttyS0] disabled
[    8.562026] 44e09000.serial: ttyS0 at MMIO 0x44e09000 (irq = 20, base_baud = 3000000) is a 8250
[    9.403338] printk: console [ttyS0] enabled
[    9.409541] 48022000.serial: ttyS1 at MMIO 0x48022000 (irq = 27, base_baud = 3000000) is a 8250
[    9.420246] 48024000.serial: ttyS2 at MMIO 0x48024000 (irq = 28, base_baud = 3000000) is a 8250
[    9.431101] 481a6000.serial: ttyS3 at MMIO 0x481a6000 (irq = 41, base_baud = 3000000) is a 8250
[    9.441629] 481a8000.serial: ttyS4 at MMIO 0x481a8000 (irq = 42, base_baud = 3000000) is a 8250
[    9.452074] 481aa000.serial: ttyS5 at MMIO 0x481aa000 (irq = 43, base_baud = 3000000) is a 8250
[    9.465600] omap_rng 48310000.rng: Random Number Generator ver. 20
[    9.472213] random: crng init done
[    9.590827] loop: module loaded
[    9.594334] at24 2-0054: supply vcc not found, using dummy regulator
[    9.630159] at24 2-0055: supply vcc not found, using dummy regulator
[    9.665929] at24 2-0056: supply vcc not found, using dummy regulator
[    9.702237] at24 2-0057: supply vcc not found, using dummy regulator
[    9.826721] usbcore: registered new interface driver smsc95xx
[    9.835619] am335x-phy-driver 47401300.usb-phy: supply vcc not found, using dummy regulator
[    9.844463] am335x-phy-driver 47401300.usb-phy: dummy supplies not allowed for exclusive requests
[    9.860316] am335x-phy-driver 47401b00.usb-phy: supply vcc not found, using dummy regulator
[    9.869202] am335x-phy-driver 47401b00.usb-phy: dummy supplies not allowed for exclusive requests
[    9.885046] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[    9.891927] ehci-platform: EHCI generic platform driver
[    9.897971] ehci-omap: OMAP-EHCI Host Controller driver
[    9.917296] musb-hdrc musb-hdrc.1: MUSB HDRC host driver
[    9.922980] musb-hdrc musb-hdrc.1: new USB bus registered, assigned bus number 1
[    9.930792] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 5.10
[    9.939126] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    9.946399] usb usb1: Product: MUSB HDRC host driver
[    9.951405] usb usb1: Manufacturer: Linux 5.10.168-ti-r72 musb-hcd
[    9.957630] usb usb1: SerialNumber: musb-hdrc.1
[    9.963174] hub 1-0:1.0: USB hub found
[    9.967103] hub 1-0:1.0: 1 port detected
[    9.984458] omap_rtc 44e3e000.rtc: already running
[    9.991223] omap_rtc 44e3e000.rtc: registered as rtc0
[    9.996442] omap_rtc 44e3e000.rtc: setting system clock to 2023-10-30T16:29:17 UTC (1698683357)
[   10.007276] i2c /dev entries driver
[   10.016726] omap_wdt: OMAP Watchdog Timer Rev 0x01: initial timeout 60 sec
[   10.024545] softdog: initialized. soft_noboot=0 soft_margin=60 sec soft_panic=0 (nowayout=0)
[   10.033130] softdog:              soft_reboot_cmd=<not set> soft_active_on_boot=0
[   10.043036] cpuidle: enable-method property 'ti,am3352' found operations
[   10.052082] sdhci: Secure Digital Host Controller Interface driver
[   10.058387] sdhci: Copyright(c) Pierre Ossman
[   10.062900] sdhci-pltfm: SDHCI platform and OF driver helper
[   10.079705] sdhci-omap 481d8000.mmc: supply vqmmc not found, using dummy regulator
[   10.087636] ledtrig-cpu: registered to indicate activity on CPUs
[   10.095392] sdhci-omap 47810000.mmc: supply vqmmc not found, using dummy regulator
[   10.106200] omap-aes 53500000.aes: OMAP AES hw accel rev: 3.2
[   10.117884] omap-aes 53500000.aes: will run requests pump with realtime priority
[   10.129061] omap-sham 53100000.sham: hw accel on OMAP rev 4.3
[   10.135711] omap-sham 53100000.sham: will run requests pump with realtime priority
[   10.148609] hid: raw HID events driver (C) Jiri Kosina
[   10.153992] mmc1: SDHCI controller on 481d8000.mmc [481d8000.mmc] using External DMA
[   10.163969] usbcore: registered new interface driver usbhid
[   10.170464] usbhid: USB HID core driver
[   10.175155] remoteproc remoteproc0: wkup_m3 is available
[   10.191466] NET: Registered protocol family 10
[   10.205440] Segment Routing with IPv6
[   10.209573] mip6: Mobile IPv6
[   10.212773] NET: Registered protocol family 17
[   10.221956] Key type dns_resolver registered
[   10.226390] mpls_gso: MPLS GSO support
[   10.230462] ThumbEE CPU extension supported.
[   10.234980] Registering SWP/SWPB emulation handler
[   10.239887] omap_voltage_late_init: Voltage driver support not added
[   10.247675] registered taskstats version 1
[   10.252230] Loading compiled-in X.509 certificates
[   10.257506] zswap: loaded using pool lzo/zbud
[   10.263115] mmc1: new high speed MMC card at address 0001
[   10.269166] Key type .fscrypt registered
[   10.273988] Key type fscrypt-provisioning registered
[   10.283842] mmcblk1: mmc1:0001 M62704 3.53 GiB 
[   10.292504] mmcblk1boot0: mmc1:0001 M62704 partition 1 2.00 MiB
[   10.298594] Btrfs loaded, crc32c=crc32c-generic
[   10.303318] AppArmor: AppArmor sha1 policy hashing enabled
[   10.309579] mmcblk1boot1: mmc1:0001 M62704 partition 2 2.00 MiB
[   10.319298] mmcblk1rpmb: mmc1:0001 M62704 partition 3 512 KiB, chardev (240:0)
[   10.337402]  mmcblk1: p1
[   10.351981] OMAP GPIO hardware version 0.1
[   10.383533] tps65217-pmic: Failed to locate of_node [id: -1]
[   10.401168] tps65217-bl: Failed to locate of_node [id: -1]
[   10.409848] tps6521x_pwrbutton tps65217-pwrbutton: DMA mask not set
[   10.417232] input: tps65217_pwr_but as /devices/platform/ocp/44c00000.interconnect/44c00000.interconnect:segment@200000/44e0b000.target-module/44e0b000.i2c/i2c-0/0-0024/tps65217-p0
[   10.435820] tps65217 0-0024: TPS65217 ID 0xe version 1.2
[   10.442980] at24 0-0050: 32768 byte 24c256 EEPROM, writable, 1 bytes/write
[   10.449987] usb 1-1: new high-speed USB device number 2 using musb-hdrc
[   10.456924] omap_i2c 44e0b000.i2c: bus 0 rev0.11 at 400 kHz
[   10.463438] gpio-61 (LS_BUF_EN): hogged as output/high
[   10.470528] gpio-112 (MCASP0_AHCLKR): hogged as output/low
[   10.481654] gpio-of-helper ocp:cape-universal: Allocated GPIO id=0 name='P8_03'
[   10.489335] gpio-of-helper ocp:cape-universal: Allocated GPIO id=1 name='P8_04'
[   10.496920] gpio-of-helper ocp:cape-universal: Allocated GPIO id=2 name='P8_05'
[   10.504479] gpio-of-helper ocp:cape-universal: Allocated GPIO id=3 name='P8_06'
[   10.512297] gpio-of-helper ocp:cape-universal: Allocated GPIO id=4 name='P8_07'
[   10.519926] gpio-of-helper ocp:cape-universal: Allocated GPIO id=5 name='P8_08'
[   10.527473] gpio-of-helper ocp:cape-universal: Allocated GPIO id=6 name='P8_09'
[   10.535252] gpio-of-helper ocp:cape-universal: Allocated GPIO id=7 name='P8_10'
[   10.542847] gpio-of-helper ocp:cape-universal: Allocated GPIO id=8 name='P8_11'
[   10.550404] gpio-of-helper ocp:cape-universal: Allocated GPIO id=9 name='P8_12'
[   10.558043] gpio-of-helper ocp:cape-universal: Allocated GPIO id=10 name='P8_13'
[   10.565686] gpio-of-helper ocp:cape-universal: Allocated GPIO id=11 name='P8_15'
[   10.573406] gpio-of-helper ocp:cape-universal: Allocated GPIO id=12 name='P8_16'
[   10.581058] gpio-of-helper ocp:cape-universal: Allocated GPIO id=13 name='P8_18'
[   10.588684] gpio-of-helper ocp:cape-universal: Allocated GPIO id=14 name='P8_19'
[   10.596316] gpio-of-helper ocp:cape-universal: Allocated GPIO id=15 name='P8_20'
[   10.604096] gpio-of-helper ocp:cape-universal: Allocated GPIO id=16 name='P8_21'
[   10.611754] gpio-of-helper ocp:cape-universal: Allocated GPIO id=17 name='P8_22'
[   10.619388] gpio-of-helper ocp:cape-universal: Allocated GPIO id=18 name='P8_23'
[   10.627013] gpio-of-helper ocp:cape-universal: Allocated GPIO id=19 name='P8_24'
[   10.634677] gpio-of-helper ocp:cape-universal: Allocated GPIO id=20 name='P8_25'
[   10.642304] gpio-of-helper ocp:cape-universal: Allocated GPIO id=21 name='P8_27'
[   10.649936] gpio-of-helper ocp:cape-universal: Allocated GPIO id=22 name='P8_28'
[   10.657587] gpio-of-helper ocp:cape-universal: Allocated GPIO id=23 name='P8_29'
[   10.665203] gpio-of-helper ocp:cape-universal: Allocated GPIO id=24 name='P8_30'
[   10.672850] gpio-of-helper ocp:cape-universal: Allocated GPIO id=25 name='P8_31'
[   10.680477] gpio-of-helper ocp:cape-universal: Allocated GPIO id=26 name='P8_32'
[   10.688113] gpio-of-helper ocp:cape-universal: Allocated GPIO id=27 name='P8_33'
[   10.695745] gpio-of-helper ocp:cape-universal: Allocated GPIO id=28 name='P8_34'
[   10.703357] gpio-of-helper ocp:cape-universal: Allocated GPIO id=29 name='P8_35'
[   10.710995] gpio-of-helper ocp:cape-universal: Allocated GPIO id=30 name='P8_36'
[   10.718646] gpio-of-helper ocp:cape-universal: Allocated GPIO id=31 name='P8_37'
[   10.726267] gpio-of-helper ocp:cape-universal: Allocated GPIO id=32 name='P8_38'
[   10.733897] gpio-of-helper ocp:cape-universal: Allocated GPIO id=33 name='P8_39'
[   10.741529] gpio-of-helper ocp:cape-universal: Allocated GPIO id=34 name='P8_40'
[   10.749160] gpio-of-helper ocp:cape-universal: Allocated GPIO id=35 name='P8_41'
[   10.756797] gpio-of-helper ocp:cape-universal: Allocated GPIO id=36 name='P8_42'
[   10.764413] gpio-of-helper ocp:cape-universal: Allocated GPIO id=37 name='P8_43'
[   10.772038] gpio-of-helper ocp:cape-universal: Allocated GPIO id=38 name='P8_44'
[   10.779656] gpio-of-helper ocp:cape-universal: Allocated GPIO id=39 name='P8_45'
[   10.787288] gpio-of-helper ocp:cape-universal: Allocated GPIO id=40 name='P8_46'
[   10.794942] gpio-of-helper ocp:cape-universal: Allocated GPIO id=41 name='P9_11'
[   10.802567] gpio-of-helper ocp:cape-universal: Allocated GPIO id=42 name='P9_12'
[   10.810221] gpio-of-helper ocp:cape-universal: Allocated GPIO id=43 name='P9_13'
[   10.818036] gpio-of-helper ocp:cape-universal: Allocated GPIO id=44 name='P9_14'
[   10.825683] gpio-of-helper ocp:cape-universal: Allocated GPIO id=45 name='P9_15'
[   10.833323] gpio-of-helper ocp:cape-universal: Allocated GPIO id=46 name='P9_16'
[   10.840954] gpio-of-helper ocp:cape-universal: Allocated GPIO id=47 name='P9_17'
[   10.848594] gpio-of-helper ocp:cape-universal: Allocated GPIO id=48 name='P9_18'
[   10.856224] gpio-of-helper ocp:cape-universal: Allocated GPIO id=49 name='P9_19'
[   10.863859] gpio-of-helper ocp:cape-universal: Allocated GPIO id=50 name='P9_20'
[   10.871501] gpio-of-helper ocp:cape-universal: Allocated GPIO id=51 name='P9_21'
[   10.879150] gpio-of-helper ocp:cape-universal: Allocated GPIO id=52 name='P9_22'
[   10.886815] gpio-of-helper ocp:cape-universal: Allocated GPIO id=53 name='P9_23'
[   10.894454] gpio-of-helper ocp:cape-universal: Allocated GPIO id=54 name='P9_24'
[   10.902387] gpio-of-helper ocp:cape-universal: Allocated GPIO id=55 name='P9_25'
[   10.910099] gpio-of-helper ocp:cape-universal: Allocated GPIO id=56 name='P9_26'
[   10.917751] gpio-of-helper ocp:cape-universal: Allocated GPIO id=57 name='P9_27'
[   10.925386] gpio-of-helper ocp:cape-universal: Allocated GPIO id=58 name='P9_28'
[   10.933029] gpio-of-helper ocp:cape-universal: Allocated GPIO id=59 name='P9_29'
[   10.940644] gpio-of-helper ocp:cape-universal: Allocated GPIO id=60 name='P9_31'
[   10.948286] gpio-of-helper ocp:cape-universal: Allocated GPIO id=61 name='P9_41'
[   10.955933] gpio-of-helper ocp:cape-universal: Allocated GPIO id=62 name='P9_91'
[   10.963556] gpio-of-helper ocp:cape-universal: Allocated GPIO id=63 name='P9_42'
[   10.971215] gpio-of-helper ocp:cape-universal: Allocated GPIO id=64 name='P9_92'
[   10.978678] gpio-of-helper ocp:cape-universal: ready
[   10.993512] omap_gpio 44e07000.gpio: Could not set line 6 debounce to 200000 microseconds (-22)
[   11.005361] sdhci-omap 47810000.mmc: supply vqmmc not found, using dummy regulator
[   11.015238] of_cfs_init
[   11.017771] of_cfs_init: OK
[   11.023494] sdhci-omap 48060000.mmc: Got CD GPIO
[   11.030486] sdhci-omap 48060000.mmc: supply vqmmc not found, using dummy regulator
[   11.070964] mmc0: SDHCI controller on 48060000.mmc [48060000.mmc] using External DMA
[   11.104136] usb 1-1: New USB device found, idVendor=05e3, idProduct=0610, bcdDevice=32.98
[   11.112406] usb 1-1: New USB device strings: Mfr=0, Product=1, SerialNumber=0
[   11.119679] usb 1-1: Product: USB2.0 Hub
[   11.125298] hub 1-1:1.0: USB hub found
[   11.129611] hub 1-1:1.0: 4 ports detected
[   11.144880] mmc0: new high speed SDHC card at address aaaa
[   11.150463] mmc2: SDHCI controller on 47810000.mmc [47810000.mmc] using External DMA
[   11.163923] mmcblk0: mmc0:aaaa SL16G 14.8 GiB 
[   11.169328] sdhci-omap 47810000.mmc: card claims to support voltages below defined range
[   11.183238]  mmcblk0: p1
[   11.219007] EXT4-fs (mmcblk0p1): mounted filesystem with ordered data mode. Opts: (null)
[   11.227542] VFS: Mounted root (ext4 filesystem) readonly on device 179:769.
[   11.235801] mmc2: new high speed SDIO card at address 0001
[   11.249353] devtmpfs: mounted
[   11.254123] Freeing unused kernel memory: 1024K
[   11.259492] Run /sbin/init as init process
[   11.298469] Not activating Mandatory Access Control as /sbin/tomoyo-init does not exist.
[   11.926515] systemd[1]: systemd 247.3-7+deb11u4 running in system mode. (+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +Z)
[   11.951631] systemd[1]: Detected architecture arm.

Welcome to Debian GNU/Linux 11 (bullseye)!

[   11.979191] systemd[1]: Set hostname to <arm>.
[   13.437710] systemd[1]: Queued start job for default target Graphical Interface.
[   13.455180] systemd[1]: Created slice system-getty.slice.
[  OK  ] Created slice system-getty.slice.
[   13.481771] systemd[1]: Created slice system-modprobe.slice.
[  OK  ] Created slice system-modprobe.slice.
[   13.506085] systemd[1]: Created slice system-serial\x2dgetty.slice.
[  OK  ] Created slice system-serial\x2dgetty.slice.
[   13.533358] systemd[1]: Created slice User and Session Slice.
[  OK  ] Created slice User and Session Slice.
[   13.555753] systemd[1]: Started Dispatch Password Requests to Console Directory Watch.
[  OK  ] Started Dispatch Password …ts to Console Directory Watch.
[   13.579622] systemd[1]: Started Forward Password Requests to Wall Directory Watch.
[  OK  ] Started Forward Password R…uests to Wall Directory Watch.
[   13.604634] systemd[1]: Set up automount Arbitrary Executable File Formats File System Automount Point.
[  OK  ] Set up automount Arbitrary…s File System Automount Point.
[   13.631349] systemd[1]: Reached target Local Encrypted Volumes.
[  OK  ] Reached target Local Encrypted Volumes.
[   13.655449] systemd[1]: Reached target Paths.
[  OK  ] Reached target Paths.
[   13.675077] systemd[1]: Reached target Remote File Systems.
[  OK  ] Reached target Remote File Systems.
[   13.698992] systemd[1]: Reached target Slices.
[  OK  ] Reached target Slices.
[   13.719166] systemd[1]: Reached target Swap.
[  OK  ] Reached target Swap.
[   13.744402] systemd[1]: Listening on Syslog Socket.
[  OK  ] Listening on Syslog Socket.
[   13.768189] systemd[1]: Listening on fsck to fsckd communication Socket.
[  OK  ] Listening on fsck to fsckd communication Socket.
[   13.791654] systemd[1]: Listening on initctl Compatibility Named Pipe.
[  OK  ] Listening on initctl Compatibility Named Pipe.
[   13.816646] systemd[1]: Listening on Journal Audit Socket.
[  OK  ] Listening on Journal Audit Socket.
[   13.840134] systemd[1]: Listening on Journal Socket (/dev/log).
[  OK  ] Listening on Journal Socket (/dev/log).
[   13.864383] systemd[1]: Listening on Journal Socket.
[  OK  ] Listening on Journal Socket.
[   13.888571] systemd[1]: Listening on udev Control Socket.
[  OK  ] Listening on udev Control Socket.
[   13.912026] systemd[1]: Listening on udev Kernel Socket.
[  OK  ] Listening on udev Kernel Socket.
[   13.936316] systemd[1]: Condition check resulted in Huge Pages File System being skipped.
[   13.952745] systemd[1]: Mounting POSIX Message Queue File System...
         Mounting POSIX Message Queue File System...
[   13.995162] systemd[1]: Mounting Kernel Debug File System...
         Mounting Kernel Debug File System...
[   14.049389] systemd[1]: Mounting Kernel Trace File System...
         Mounting Kernel Trace File System...
[   14.100909] systemd[1]: Starting Restore / save the current clock...
         Starting Restore / save the current clock...
[   14.147331] systemd[1]: Starting Create list of static device nodes for the current kernel...
         Starting Create list of st…odes for the current kernel...
[   14.208950] systemd[1]: Starting Load Kernel Module configfs...
         Starting Load Kernel Module configfs...
[   14.268696] systemd[1]: Starting Load Kernel Module drm...
         Starting Load Kernel Module drm...
[   14.311962] systemd[1]: Starting Load Kernel Module fuse...
         Starting Load Kernel Module fuse...
[   14.345920] systemd[1]: Condition check resulted in Set Up Additional Binary Formats being skipped.
[   14.364052] systemd[1]: Starting File System Check on Root Device...
         Starting File System Check on Root Device...
[   14.420333] systemd[1]: Starting Journal Service...
         Starting Journal Service...
[   14.492315] systemd[1]: Starting Load Kernel Modules...
         Starting Load Kernel Modules...
[   14.564452] systemd[1]: Starting Coldplug All udev Devices...
         Starting Coldplug All udev Devices...
[   14.703710] systemd[1]: Mounted POSIX Message Queue File System.
[  OK  ] Mounted POSIX Message Queue File Sy[   14.736865] systemd[1]: Mounted Kernel Debug File System.
stem.
[  OK  ] Mounted Kernel Debug File System.
[   14.803618] systemd[1]: Mounted Kernel Trace File System.
[  OK  ] Mounted Kernel Trace File System.
[   14.853062] systemd[1]: Finished Restore / save the current clock.
[  OK  ] Finished Restore / save the current clock.
[   14.916899] systemd[1]: Finished Create list of static device nodes for the current kernel.
[  OK  ] Finished Create list of st… nodes for the current kernel.
[   14.962379] systemd[1]: modprobe@configfs.service: Succeeded.
[   14.999570] systemd[1]: Finished Load Kernel Module configfs.
[  OK  ] Finished Load Kernel Module configfs.
[   15.038261] systemd[1]: modprobe@drm.service: Succeeded.
[   15.065917] systemd[1]: Finished Load Kernel Module drm.
[  OK  ] Finished Load Kernel Module drm.
[   15.106330] systemd[1]: modprobe@fuse.service: Succeeded.
[   15.139220] systemd[1]: Finished Load Kernel Module fuse.
[  OK  ] Finished Load Kernel Module fuse.
[   15.180076] systemd[1]: Finished File System Check on Root Device.
[  OK  ] Finished File System Check on Root Device.
[   15.223780] systemd[1]: Finished Load Kernel Modules.
[  OK  ] Finished Load Kernel Modules.
[   15.282038] systemd[1]: Mounting FUSE Control File System...
         Mounting FUSE Control File System...
[   15.383654] systemd[1]: Mounting Kernel Configuration File System...
         Mounting Kernel Configuration File System...
[   15.484446] systemd[1]: Started File System Check Daemon to report status.
[  OK  ] Started File System Check Daemon to report status.
[   15.576355] systemd[1]: Starting Remount Root and Kernel File Systems...
         Starting Remount Root and Kernel File Systems...
[   15.665189] systemd[1]: Starting Apply Kernel Variables...
         Starting Apply Kernel Variables...
[   15.758295] systemd[1]: Mounted FUSE Control File System.
[  OK  ] Mounted FUSE Control File System.
[   15.824686] systemd[1]: Mounted Kernel Configuration File System.
[  OK  ] Mounted Kernel Configuration File System.
[   15.938517] EXT4-fs (mmcblk0p1): re-mounted. Opts: errors=remount-ro
[   15.959441] systemd[1]: Finished Apply Kernel Variables.
[  OK  ] Finished Apply Kernel Variables.
[   16.027577] systemd[1]: Finished Remount Root and Kernel File Systems.
[  OK  ] Finished Remount Root and Kernel File Systems.
[   16.071458] systemd[1]: Condition check resulted in Rebuild Hardware Database being skipped.
[   16.099490] systemd[1]: Condition check resulted in Platform Persistent Storage Archival being skipped.
[   16.165069] systemd[1]: Starting Load/Save Random Seed...
         Starting Load/Save Random Seed...
[   16.232486] systemd[1]: Starting Create System Users...
         Starting Create System Users...
[   16.284988] systemd[1]: Started Journal Service.
[  OK  ] Started Journal Service.
         Starting Flush Journal to Persistent Storage...
[  OK  ] Finished Load/Save Random Seed.
[   16.660325] systemd-journald[172]: Received client request to flush runtime journal.
[  OK  ] Finished Create System Users.
         Starting Create Static Device Nodes in /dev...
[  OK  ] Finished Create Static Device Nodes in /dev.
[  OK  ] Reached target Local File Systems (Pre).
[  OK  ] Reached target Local File Systems.
         Starting Rule-based Manage…for Device Events and Files...
[  OK  ] Finished Coldplug All udev Devices.
         Starting Helper to synchronize boot up for ifupdown...
[  OK  ] Finished Helper to synchronize boot up for ifupdown.
[  OK  ] Finished Flush Journal to Persistent Storage.
         Starting Raise network interfaces...
         Starting Create Volatile Files and Directories...
[  OK  ] Started Rule-based Manager for Device Events and Files.
[  OK  ] Finished Create Volatile Files and Directories.
         Starting Network Time Synchronization...
         Starting Update UTMP about System Boot/Shutdown...
[  OK  ] Finished Raise network interfaces.
[  OK  ] Finished Update UTMP about System Boot/Shutdown.
[  OK  ] Started Network Time Synchronization.
[  OK  ] Reached target System Initialization.
[  OK  ] Started Daily Cleanup of Temporary Directories.
[  OK  ] Reached target System Time Set.
[  OK  ] Reached target System Time Synchronized.
[  OK  ] Started Periodic ext4 Onli…ata Check for All Filesystems.
[  OK  ] Started Discard unused blocks once a week.
[  OK  ] Started Daily rotation of log files.
[  OK  ] Reached target Timers.
[  OK  ] Listening on Avahi mDNS/DNS-SD Stack Activation Socket.
[  OK  ] Listening on D-Bus System Message Bus Socket.
[  OK  ] Reached target Sockets.
[  OK  ] Reached target Basic System.
         Starting Avahi mDNS/DNS-SD Stack...
[  OK  ] Started Regular background program processing daemon.
[  OK  ] Started D-Bus System Message Bus.
         Starting Network Manager...
         Starting Remove Stale Onli…t4 Metadata Check Snapshots...
         Starting Authorization Manager...
         Starting System Logging Service...
         Starting User Login Management...
         Starting WPA supplicant...
[  OK  ] Started System Logging Service.
[  OK  ] Started Avahi mDNS/DNS-SD Stack.
[  OK  ] Started WPA supplicant.
[  OK  ] Started Authorization Manager.
         Starting Modem Manager...
[  OK  ] Finished Remove Stale Onli…ext4 Metadata Check Snapshots.
[  OK  ] Started Network Manager.
[  OK  ] Reached target Network.
         Starting A high performanc… and a reverse proxy server...
         Starting OpenBSD Secure Shell server...
         Starting Permit User Sessions...
         Starting Hostname Service...
[  OK  ] Started User Login Management.
[  OK  ] Finished Permit User Sessions.
[  OK  ] Started Getty on tty1.
[  OK  ] Started OpenBSD Secure Shell server.
[  OK  ] Started Modem Manager.
[  OK  ] Started A high performance…er and a reverse proxy server.
[  OK  ] Started Hostname Service.
         Starting Network Manager Script Dispatcher Service...
[  OK  ] Started Network Manager Script Dispatcher Service.
[  OK  ] Found device /dev/ttyS0.
[  OK  ] Started Serial Getty on ttyS0.
[  OK  ] Reached target Login Prompts.
[  OK  ] Reached target Multi-User System.
[  OK  ] Reached target Graphical Interface.
         Starting Update UTMP about System Runlevel Changes...
[  OK  ] Finished Update UTMP about System Runlevel Changes.
[   28.182829] remoteproc remoteproc0: powering up wkup_m3
[   28.190792] remoteproc remoteproc0: Booting fw image am335x-pm-firmware.elf, size 217148
[   28.215142] remoteproc remoteproc0: remote processor wkup_m3 is now up
[   28.215166] wkup_m3_ipc 44e11324.wkup_m3_ipc: CM3 Firmware Version = 0x192

Debian GNU/Linux 11 arm ttyS0

default username:password is [debian:temppwd]

arm login: [   31.163035] CAN device driver interface
[   31.399603] c_can_platform 481cc000.can: c_can_platform device registered (regs=0cc4273a, irq=46)
[   31.554820] c_can_platform 481d0000.can: c_can_platform device registered (regs=7b05e633, irq=47)
[   34.113418] PM: bootloader does not support rtc-only!
[   36.194815] remoteproc remoteproc1: 4a334000.pru is available
[   36.230897] remoteproc remoteproc2: 4a338000.pru is available
[   36.695520] cfg80211: Loading compiled-in X.509 certificates for regulatory database
[   36.708798] cfg80211: Loaded X.509 cert 'sforshee: 00b28ddf47aef9cea7'
[   37.198116] wl18xx_driver wl18xx.2.auto: Direct firmware load for ti-connectivity/wl18xx-conf.bin failed with error -2
[   37.218907] wlcore: ERROR could not get configuration binary ti-connectivity/wl18xx-conf.bin: -2
[   37.242375] wlcore: WARNING falling back to default config
[   37.758746] wlcore: wl18xx HW: 183x or 180x, PG 2.2 (ROM 0x11)
[   37.775581] wlcore: WARNING Detected unconfigured mac address in nvs, derive from fuse instead.
[   37.793454] wlcore: WARNING This default nvs file can be removed from the file system
[   37.840223] wlcore: loaded
[   39.383549] wlcore: PHY firmware version: Rev 8.2.0.0.243
[   39.515652] wlcore: firmware booted (Rev 8.9.0.0.83)
[   44.053844] wlan0: authenticate with c0:06:c3:bd:5e:db
[   44.071814] wlan0: send auth to c0:06:c3:bd:5e:db (try 1/3)
[   44.099377] wlan0: authenticated
[   44.108210] wlan0: associate with c0:06:c3:bd:5e:db (try 1/3)
[   44.119262] wlan0: RX AssocResp from c0:06:c3:bd:5e:db (capab=0x1431 status=0 aid=29)
[   44.166881] wlan0: associated
[   45.047355] cryptd: max_cpu_qlen set to 1000
[   45.158432] IPv6: ADDRCONF(NETDEV_CHANGE): wlan0: link becomes ready
[   45.181042] wlcore: Association completed.
debian
Password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Oct 30 15:56:19 UTC 2023 from 192.168.1.13 on pts/0
debian@arm:~$ 
```
</details>