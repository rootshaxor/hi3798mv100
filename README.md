#  Hi3798mv100 (Huawei ec6108v9 IPTV) Linux kompilasi dan pembakaran blog
Artikel ini merekam proses kompilasi kernel, membakar uboot, dan mem-flash rootfs Ubuntu 16.04 untuk set-top box Huawei EC6108v9 (chip Hisilicon Hi3798mv100). Pada saat yang sama, saya menebus pengetahuan yang relevan tentang uboot.
##  Lingkungan dasar
Papan target: IPTV menghentikan set-top box Huawei EC6108v9 (hisilicon Hi3798mv100 2G 8G emmc)
Lingkungan kompilasi: Ubuntu 16.04 32bit VM
Kernel linux HiSilicon: HiSTBLinux cocok untuk hi3798mv100 mv200
SDK: HiSTBLinuxV100R005C00SPC041B020
 
##  Persiapan lingkungan
 
```
git clone https://github.com/glinuz/hi3798mv100
# beralih ke direktori kerja
cd HiSTBLinuxV100R005C00SPC041B020 #$SDK_path
#Instal alat kompilasi yang diperlukan, Anda dapat menggunakan skrip shell yang disertakan dengan SDK, atau Anda dapat menginstalnya sendiri
sh server_install.sh
#atau
apt-get install gcc make gettext bison flex bc zlib1g-dev libncurses5-dev lzma
#Salin yang telah ditentukan sebelumnya di SDK
cp configs/hi3798mv100/hi3798mdmo1g_hi3798mv100_cfg.mak ./cfg.mak
source ./env.sh #SDK berbagai variabel lingkungan
#Modifikasi konfigurasi yang telah dikompilasi sesuai kebutuhan
make menuconfig
make build -j4 2>&1 | tee -a buildlog.txt
```
Setelah produksi berhasil, Anda dapat menemukan fastboot-burn.bin, bootargs.bin, hi_kernel.bin yang dikompilasi di out/hi3798mv100, yang masing-masing merupakan file boot uboot, konfigurasi parameter boot uboot, dan kernel linux.
##  Gunakan HiTool untuk membakar ke eMMC
Lihat [hi3798mv100-ec6109.jpg] untuk diagram koneksi TTL. Anda dapat mencari tutorial hitol untuk skema pembakaran tertentu.
 
konfigurasi antarmuka pembakaran hitol [hit00l-burn.png]
 
partisi eMMC adalah uboot 1M, bootargs 1M, kernel 8M, rootfs 128M, lihat [emmc_partitions.xml] untuk detailnya.
 
Jika Anda mengubah ukuran partisi dan menyesuaikan ukuran partisi, Anda perlu memodifikasi bootargs.txt dan emmc_partitions.xml secara bersamaan.
 
configs/hi3798mv100/prebuilts/bootargs.txt, dan buat ulang file bootargs.bin
 
```
bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x4000;bootm 0x1FFFFC0
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)

mkbootargs  -s 1M -r bootargs.txt  -o bootargs.bin
```
Instruksi operasi bootcmd: mulai dari 2M byte pada blok perangkat mmc ke-0 (0x1000 desimal 4096,4096 *512/1024=2M), baca 16×512 byte (0x4000 desimal 16384* 512/1024=8M) ke memori 0x1FFFFC0 dan boot dari di sana.
 
Buka konsol port serial untuk debugging. konsol=ttyAMA0,115200
Output dari proses startup uboot adalah sebagai berikut:
```
Bootrom start
Boot from eMMC
Starting fastboot ...

System startup
DDRS
Reg Version:  v1.1.0
Reg Time:     2016/1/18  14:01:18
Reg Name:     hi3798mdmo1g_hi3798mv100_ddr3_1gbyte_16bitx2_4layers_emmc.reg

Jump to DDR


Fastboot 3.3.0 (root@glinuz) (Jul 25 2020 - 08:25:47)

Fastboot:      Version 3.3.0
Build Date:    Jul 25 2020, 08:26:41
CPU:           Hi3798Mv100 
Boot Media:    eMMC
DDR Size:      1GB


MMC/SD controller initialization.
MMC/SD Card:
    MID:         0x15
    Read Block:  512 Bytes
    Write Block: 512 Bytes
    Chip Size:   7456M Bytes (High Capacity)
    Name:        "8GME4R"
    Chip Type:   MMC
    Version:     5.1
    Speed:       52000000Hz
    Mode:        DDR50
    Bus Width:   8bit
    Boot Addr:   0 Bytes
Net:   upWarning: failed to set MAC address


Boot Env on eMMC
    Env Offset:          0x00100000
    Env Size:            0x00010000
    Env Range:           0x00010000
ID_WORD have already been locked


SDK Version: HiSTBLinuxV100R005C00SPC041B020_20161028

Reserve Memory
    Start Addr:          0x3FFFE000
    Bound Addr:          0x8D24000
    Free  Addr:          0x3F8FC000
    Alloc Block:  Addr         Size
                  0x3FBFD000   0x400000
                  0x3F8FC000   0x300000

Press Ctrl+C to stop autoboot

MMC read: dev # 0, block # 4096, count 16384 ... 16384 blocks read: OK

84937034 Bytes/s
## Booting kernel from Legacy Image at 01ffffc0 ...
   Image Name:   Linux-3.18.24_s40
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    6959232 Bytes = 6.6 MiB
   Load Address: 02000000
   Entry Point:  02000000
   Verifying Checksum ... OK
   XIP Kernel Image ... OK
OK
ATAGS [0x00000100 - 0x00000300], 512Bytes

Starting kernel ...
```
##  Kompilasi lanjutan
###  Sesuaikan kernel linux
File konfigurasi kernel platform ARM mengadopsi format defconfig, dan proses menggunakan dan menyimpan deconfig dengan benar adalah sebagai berikut:
 
sumber/kernel/linux-3.18.y/arch/arm/configs/hi3798mv100_defconfig
sumber cd/kernel/linux-3.18.y/
Anda dapat menggunakan hi3798mv100_defconfig-0812 yang disediakan oleh pustaka git ini
1. Cadangkan dulu hi3798mv100_defconfig
2. make ARCH=arm hi3798mv100_defconfig #Buat file .config konfigurasi kernel linux standar dari defconfig
3. make ARCH=arm menuconfig #Ubah konfigurasi kernel dan simpan
4. make ARCH=arm savedefconfig #Regenerasi file defconfg
5. cp defconfig arch/arm/configs/hi3798mv100_defconfig #Salin file defconfig ke lokasi yang benar.
6. make distclean #Bersihkan file-file yang dikompilasi dan dibuat sebelumnya
7. cd $SDK_path;make linux #kompilasi ulang kernel
 
Parameter kompilasi kernel yang perlu diperhatikan:
 
Buka devtmpfs, / sistem file dev
 
buka buka dengan fhandle syscalls
 
Buka fungsi cgroup
 
###  Modifikasi uboot
```
source/boot/fastboot/include/configs godbox.h
#define CONFIG_SHOW_MEMORY_LAYOUT 1
#define CONFIG_SHOW_REG_INFO      1
#define CONFIG_SHOW_RESERVE_MEM_LAYOUT        1

or
cd $SDK_path;make hiboot CONFIG_SHOW_RESERVE_MEM_LAYOUT='y'
```
CONFIG_SHOW_RESERVE_MEM_LAYOUT='y' Saat kompilasi, aktifkan sakelar untuk menampilkan informasi MEM saat uboot dimulai
##  Ubah parameter startup uboot saat startup
Pada fase startup uboot, Ctrl+C memasuki mode uboot
```
 setenv bootargs console=tty1 console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)  ipaddr=192.168.10.100 gateway=192.168.10.1 netmask=255.255.255.0 netdev=eth0
 saveenv
 reset
```
##  membuat rootfs ubuntu
```
apt-get install binfmt-support debootstrap qemu qemu-user-static
cd;mkdir rootfs
debootstrap --arch=armhf --variant=minbase  --foreign --include=locales,util-linux,apt-utils,ifupdown,systemd-sysv,iproute2,curl,wget,expect,ca-certificates,openssh-server,isc-dhcp-client,vim-tiny,bzip2,cpio,usbutils,netbase,parted,jq,bc,crda,wireless-tools,iw stretch rootfs http://mirrors.ustc.edu.cn/debian/

cd rootfs
cp /usr/bin/qemu-arm-static usr/bin
mount -v --bind /dev dev
mount -vt devpts devpts dev/pts -o gid=5,mode=620
mount -t proc proc proc
mount -t sysfs sysfs sys
mount -t tmpfs tmpfs run
LC_ALL=C LANGUAGE=C LANG=C chroot . /debootstrap/debootstrap --second-stage
LC_ALL=C LANGUAGE=C LANG=C chroot . dpkg --configure -a

LC_ALL=C LANGUAGE=C LANG=C chroot . /bin/bash  #以下命令在chroot环境bash执行
mkdir /proc
mkdir /tmp
mkdir /sys
mkdir /root

mknod /dev/console c 5 1
mknod /dev/ttyAMA0 c 204 64
mknod /dev/ttyAMA1 c 204 65

mknod /dev/ttyS000 c 204 64
mknod /dev/null    c 1   3
mknod /dev/urandom   c 1   9
mknod /dev/zero    c 1   5
mknod /dev/random    c 1   8
mknod /dev/tty    c 5   0

echo "nameserver 114.114.114.114" > /etc/resolv.conf
echo "hi3798m" > /etc/hostname
echo "Asia/Shanghai" > /etc/timezone
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

echo "en_US.UTF-8 UTF-8" > etc/locale.gen
echo "zh_CN.UTF-8 UTF-8" >> etc/locale.gen
echo "zh_CN.GB2312 GB2312" >> etc/locale.gen
echo "zh_CN.GBK GBK" >> etc/locale.gen

locale-gen

echo "LANG=en_US.UTF-8" > /etc/locale.conf

echo "deb http://mirrors.ustc.edu.cn/debian/  stretch main contrib non-free" >  /etc/apt/sources.list

ln -s /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@ttyAMA0.service
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config

apt autoremove
apt-get autoclean
apt-get clean
apt clean
```
Buat cermin rootfs
```
make_ext4fs -l 128M -s rootfs_128M.ext4 ./rootfs
```
Referensi
 
[1] https://wiki.ubuntu.com/ARM/RootfsFromScratch/QemuDebootstrap
 
[2] http://gnu-linux.org/building-ubuntu-rootfs-for-arm.html
 
##  Paket Flash - file biner
rilis unduhan file
fastboot-bin.bin paket partisi uboot
bootargs.bin paket partisi parameter uboot
paket partisi kernel hi_kernel.bin
rootfs_128m.ext paket partisi root root
emmc_partitions.xml file konfigurasi partisi flash
Jika Anda menyesuaikan ukuran partisi, Anda perlu membuat ulang bootargs.bin dan menyesuaikan file konfigurasi partisi.
Gunakan Huawei hi-tool, emmc untuk membakar
 
##  instruksi uboot
Banyak siswa meminta uboot untuk memulai, parameter utama uboot adalah sebagai berikut, chip memori emmc
```
bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x4000;bootm 0x1FFFFC0
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)
```
bootcmd uboot boot: mmc baca <device num> addr blk
Alamat memori instruksi panjang alamat mmc
mmc baca 0 0x1FFFFC0 0x1000 0x4000
bootm 0x1FFFFC0 # Boot kernel dari alamat memori
 
 
##  lainnya
Kemudian, docker python golang dan paket perangkat lunak lainnya ditambahkan ke debootstrap, dan rootfs ditingkatkan menjadi 4GB, dan bootargs emmc_partition yang sesuai telah dimodifikasi pada saat yang bersamaan.
