# A20-Multimedia-boards
source files and instructions for ARM-SBC-M1 board

## General Instructions
The uboot,kernel and device tree files are available for download , either as a patch or as individual files.
for compiling , follow below instructions.

TESTED on Ubuntu 22.04

        apt-get install gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu flex bison swig python3-dev device-tree-compiler git debootstrap libncurses-dev python3-setuptools libssl-dev pip2 pip picocom
        pip install pyelftools

        git clone https://github.com/arm-sbc/A64-Multimedia-boards.git
        git clone git://git.denx.de/u-boot.git

        cd u-boot
        git tag -l
        git branch -C XXXXX ( choose the latest branch) 

        patch -Np1 -i ..//arm-sbc-m1-uboot.patch

        make CROSS_COMPILE=arm-linux-gnueabihf- A20-ARM-SBC-M1_defconfig
        make CROSS_COMPILE=arm-linux-gnueabihf-

#### flashing uboot to sd-card,  first clean the sd-card 
#### use any of below commands based on your requirement 

        sudo dd if=/dev/zero of=/dev/sdX bs=1M count=1  # clear partition and boot sector
        sudo dd if=/dev/zero of=/dev/sdX bs=1k count=1023 seek=1 # clear bootloader without partitions
        sudo dd if=/dev/zero of=/dev/sdX bs=8192 # to clean it completely

then

        sudo dd if=u-boot-sunxi-with-spl.bin of=/dev/sdX bs=1024 seek=8
        sync

now you can insert the card into the board, connect the debug port with a serial cable , only TX, RX and GND port,
#### do not connect the voltage pins
incase of picocom installed and using USB UART cable

        sudo picocom -b 115200 -r -l /dev/ttyUSB0

#### you will see bootlog

#### now remove the sd-card and put back into the computer,
#### create partion of OS ###
        sudo fdisk /dev/sdX
        n ( enter, enter) 
        w
        sudo mkfs.ext4 /dev/sdX1
this will create an ext4 partition.

#### compiling kernel 
        https://kernel.org/
then select the stable veriosn and download the tarballs.
then
        tar xf linux-6.9.X.tar.xz
        cd linux-6.9.X
        patch -Np1 -i ..//arm-sbc-m1-kernel.patch
 
        ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make multi_v7_defconfig
        ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make menuconfig ( you may tweak the configuration)
        ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j8 zImage
        ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make dtbs
        ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make -j8 modules
        ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=OUT make modules modules_install

#### now before copiying kernel files, it required rootfs , create own rootfs or download from
        https://images.linuxcontainers.org/images/
#### by default there will be no root password, to set root password follow below steps,
#### mount the sd-card to /mnt
        sudo mount /dev/sdX /mnt
        sudo tar xf rootfs.tar.xz  -C /mnt
        sudo cp /usr/bin/qemu-aarch64-static /mnt/usr/bin/
        sudo chroot /mnt /usr/bin/qemu-aarch64-static /bin/sh -i
        passwd then set the password
it is possible to add packages at this stage.
#### OR CREATE own rootfs.
#### mount the sd-card to /mnt
        sudo mount /dev/sdX /mnt
        sudo debootstrap --arch=arm64 --foreign <distro> /mnt/  # for arm64 architecture
        sudo debootstrap --arch=armhf --foreign <distro> /mnt/  # for armhf architecture

it is possible to use any distro from debian and ubuntu with their codename.

Ubuntu 24.04 : Noble

Ubuntu 22.04 : Jammy

Debian 12  :  bookworm

Debiam 11  :  bullseye

        sudo cp /usr/bin/qemu-arm-static /mnt/usr/bin/  # for armhf 
        sudo chroot /mnt /usr/bin/qemu-arm-static /bin/sh -i # for armhf

        sudo cp /usr/bin/qemu-aarch64-static /mnt/usr/bin/   # for arm64
        sudo chroot /mnt /usr/bin/qemu-aarch64-static /bin/sh -i # for arm64

        /debootstrap/debootstrap --second-stage
        passwd then set the password
it is possible to add packages at this stage like
       apt install locales
       dpkg-reconfigure locales
       apt install openssh-server

#### add serial console
       systemctl enable serial-getty@ttyS0.service  # for sunxi boards
       systemctl enable serial-getty@ttyS2.service  # for rockchip boards
#### configure ethernet

       nano /etc/netplan/config-eth.yaml
#### then add the foolwing on ubuntu rootfs
               network:
                       renderer: networkd
                         ethernets:
                           end0:
                             dhcp4: true
#### add below for debian rootfs
       nano /etc/network/interfaces
#### then add the following
       auto lo end0
       allow-hotplug end0
       iface lo inet loopback
       iface end0 inet dhcp
#### update source list
##### for ubuntu use the follwing and change the distro name 
       nano /etc/apt/sources.list
       then add below repositories 
        
       deb http://ports.ubuntu.com/ noble main universe
       deb-src http://ports.ubuntu.com/ noble main universe
       deb http://ports.ubuntu.com/ noble-security main universe
       deb-src http://ports.ubuntu.com/ noble-security main universe
       deb http://ports.ubuntu.com/ noblel-updates main universe
       deb-src http://ports.ubuntu.com/ noble -updates main universe
##### for debian use the following and chnage the distro name 
       nano /etc/apt/sources.list
       then add below repositories 
        
       deb http://deb.debian.org/debian bookworm main non-free-firmware
       deb-src http://deb.debian.org/debian bookworm main non-free-firmware
       deb http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware
       deb-src http://deb.debian.org/debian-security/ bookworm-security main non-free-firmware
       deb http://deb.debian.org/debian bookworm-updates main non-free-firmware       
       deb-src http://deb.debian.org/debian bookworm-updates main non-free-firmware
##### make tar ball what you cretaed
       tar -cjpf /home/user/sunxi_rootfs.tar.bz2 .

       change the path as required, also the dot in the end is required
##### copiying kernel files
       sudo cp arch/arm64/boot/Image /mnt/boot
       sudo cp arch/arm64/boot/dts/allwinner/sun50i-a64-arm-sbc-m3x.dtb /mnt/boot
       sudo cp -rf OUT/lib/modules /mnt/lib/
##### download firmware
       git clone  https://github.com/armbian/firmware.git
       then copy the firmware to /mnt/lib/
##### create extlinux for booting
       mkdir -p /mnt/boot/extlinux
       nano /mnt/boot/extlinux/extlinux.conf
       then add the follwing

       label 6.9.9-arm-sunxi (Mainline)
            kernel /boot/zImage
            fdt /boot/sun7i-a20-arm-sbc-m1.dtb
            append  earlyprintk console=ttyS0,115200n8 root=PARTUUID=b4407a7a-01 console=tty1  rootwait rw init=/sbin/init
##### change root=PARTUUID=b4407a7a-01 by following
       sudo blkid /dev/sdX 
       ....output will be as follows 
       /dev/sdd1: UUID="b208c36b-39d5-47b9-b67d-943f4d4798b4" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="8c4fe090-01"
       copy the PARTUUID and replace in extlinux conf.
       sudo umount /mnt
#### Now put sd-card in the board and connect 12v DC, the board will boot and it is possible to see the logs through hdmi/console.     
