# Build [Debian Linux](https://www.debian.org/) and the [Linux Libre Kernel](https://www.fsfla.org/ikiwiki/selibre/linux-libre/) on [FreeBSD](https://www.freebsd.org/)

## Overview

The [Debian package manager `dpkg` is ported to FreeBSD as part of the `BusyBox` port](https://cgit.freebsd.org/ports/tree/sysutils/busybox/pkg-descr), and the [Debian tool `debootstrap`, used to install a Debian base system, is ported to FreeBSD](https://cgit.freebsd.org/ports/tree/sysutils/debootstrap/pkg-descr), and the AMD64 Debian package binaries for the FreeBSD kernel are available in the [Debian 7 kFreeBSD distribution](https://www.debian.org/releases/wheezy/kfreebsd-amd64/release-notes.en.pdf). So, let's create [FreeBSD Jails](https://docs.freebsd.org/en/books/handbook/jails/) to build the Debian Linux user space and to compile the Linux Libre kernel.

## References

- [Scott William Beasley's Debian from Scratch](https://github.com/scottwilliambeasley/debian-from-scratch.git)
- [Linux from Scratch](http://www.linuxfromscratch.org)

## Requirements

Use [FreeBSD 13.0](https://www.freebsd.org/releases/13.0R/announce/).

Install BusyBox and Debootstrap:

```sh
sudo pkg install sysutils/busybox sysutils/debootstrap
```

## Build

### [BusyBox](https://busybox.net/)

Create the user space part of the operating system:

```sh
sudo mkdir -p /jails/busybox
sudo mkdir -p /jails/busybox/bin
sudo mkdir -p /jails/busybox/dev
sudo mkdir -p /jails/busybox/etc
sudo mkdir -p /jails/busybox/lib
sudo mkdir -p /jails/busybox/libexec

sudo cp /usr/local/bin/busybox /jails/busybox/bin/
sudo sh -c 'echo "root::0:0::0:0:secret &:/root:/bin/bash" > /jails/busybox/etc/master.passwd'
sudo pwd_mkdb -d /jails/busybox/etc/ -p /jails/busybox/etc/master.passwd
sudo sh -c 'echo "nameserver 8.8.8.8" > /jails/busybox/etc/resolv.conf'
sudo cp -R /lib/* /jails/busybox/lib/
sudo cp -R /libexec/* /jails/busybox/libexec/
```

### [Debian](https://www.debian.org/)

Get the Debian 7 with FreeBSD kernel installation media, and create the user space part of the operating system:

```sh
wget http://cdimage.debian.org/mirror/cdimage/archive/7.11.0/kfreebsd-amd64/iso-dvd/debian-7.11.0-kfreebsd-amd64-DVD-1.iso
wget http://cdimage.debian.org/mirror/cdimage/archive/7.11.0/kfreebsd-amd64/iso-dvd/debian-update-7.11.0-kfreebsd-amd64-DVD-1.iso
wget http://cdimage.debian.org/mirror/cdimage/archive/7.11.0/kfreebsd-amd64/iso-dvd/debian-update-7.11.0-kfreebsd-amd64-DVD-2.iso

sudo mkdir -p /jails/debian
sudo mkdir /media/debian-7-kfreebsd-dvd1
sudo mdconfig debian-7.11.0-kfreebsd-amd64-DVD-1.iso
sudo mount -t cd9660 /dev/md0 /media/debian-7-kfreebsd-dvd1
sudo debootstrap --arch=kfreebsd-amd64 wheezy /jails/debian file:///media/debian-7-kfreebsd-dvd1
sudo umount /media/debian-7-kfreebsd-dvd1
sudo rm -rf /media/debian-7-kfreebsd-dvd1
sudo mdconfig -d -o force -u /dev/md0
sudo sh -c 'echo "en_US.UTF-8 UTF-8" > /jails/debian/etc/locale.gen'
```

### Start the FreeBSD Jail Systems

Move [the FreeBSD Jails configuration file](./jail.conf) in place and update it with your network interface identifier and with the IP addresses that you want to use, then enable and start the [FreeBSD Jail](https://docs.freebsd.org/en/books/handbook/jails/) service, and list all running systems:

```sh
sudo cp jail.conf /etc/jail.conf
sudo sh -c 'sed -ir "s/\[busybox-ip\]/192.168.42.73/" /etc/jail.conf'
sudo sh -c 'sed -ir "s/\[debian-ip\]/192.168.42.74/" /etc/jail.conf'
sudo sh -c 'sed -ir "s/\[network-interface\]/ue0/" /etc/jail.conf'
sudo sysrc jail_enable="YES"
sudo service jail start
jls
```

### Access the FreeBSD Jail Systems

#### Access the BusyBox Jail

```sh
sudo jexec busybox /bin/busybox ash
```

Create soft links to all the BusyBox utilities:

```sh
cd /bin
busybox --list | busybox xargs -n1 busybox ln -s busybox
cd ..
```

Do what you want to do in the system with a BusyBox user space:

The user space part of a Linux operating system usually consists of:
- [GNU Coreutils](https://www.gnu.org/software/coreutils/)
- [GNU Binutils](https://www.gnu.org/software/binutils)
- The GNU Build System, including the [GNU Compiler Collection (GCC)](https://www.gnu.org/software/gcc/) and the [GNU Debugger (GDB)](https://www.sourceware.org/gdb/).

```sh
# Do what you want to do in the system with a BusyBox user space, and then exit back out to FreeBSD
# ...
# @todo: https://github.com/scottwilliambeasley/debian-from-scratch.git
# @todo: http://www.linuxfromscratch.org
# ...
exit
```

#### Access the Debian Jail

Set up the Debian system's package manager to get binary packages from the installation media for Debian with the FreeBSD kernel, and get the packages required to build the Linux kernel:

```sh
sudo mkdir /jails/debian/media/debian-7-kfreebsd-dvd1
sudo mkdir /jails/debian/media/debian-7-kfreebsd-update-dvd1
sudo mkdir /jails/debian/media/debian-7-kfreebsd-update-dvd2
sudo mdconfig debian-7.11.0-kfreebsd-amd64-DVD-1.iso
sudo mdconfig debian-update-7.11.0-kfreebsd-amd64-DVD-1.iso
sudo mdconfig debian-update-7.11.0-kfreebsd-amd64-DVD-2.iso
sudo mount -t cd9660 /dev/md0 /jails/debian/media/debian-7-kfreebsd-dvd1
sudo mount -t cd9660 /dev/md1 /jails/debian/media/debian-7-kfreebsd-update-dvd1
sudo mount -t cd9660 /dev/md2 /jails/debian/media/debian-7-kfreebsd-update-dvd2

sudo sh -c 'echo "deb [trusted=yes] file:///media/debian-7-kfreebsd-dvd1 wheezy main contrib" > /jails/debian/etc/apt/sources.list'
sudo sh -c 'echo "deb [trusted=yes] file:///media/debian-7-kfreebsd-update-dvd1 wheezy main contrib non-free" >> /jails/debian/etc/apt/sources.list'
sudo sh -c 'echo "deb [trusted=yes] file:///media/debian-7-kfreebsd-update-dvd2 wheezy main contrib non-free" >> /jails/debian/etc/apt/sources.list'
sudo chroot /jails/debian/ apt-get update
sudo chroot /jails/debian/ apt-get install -y build-essential libncurses-dev libelf1 bison flex libssl-dev bc git
sudo chroot /jails/debian/ apt-get install -y kfreebsd-headers-8.3-1 kfreebsd-kernel-headers util-linux util-linux-locales linux-base linux-source-3.2 linux-support-3.2.0-4
sudo chroot /jails/debian/ apt-get install -y libgmp10 libmpfr4 libmpc2
sudo chroot /jails/debian/ apt-get install -y ftp

sudo umount /jails/debian/media/debian-7-kfreebsd-dvd1
sudo umount /jails/debian/media/debian-7-kfreebsd-update-dvd1
sudo umount /jails/debian/media/debian-7-kfreebsd-update-dvd2
sudo mdconfig -d -o force -u /dev/md0
sudo mdconfig -d -o force -u /dev/md1
sudo mdconfig -d -o force -u /dev/md2
```

Copy the example Linux kernel configuration:

```sh
sudo mkdir -p /jails/debian/home/Projects
sudo cp kernel.config /jails/debian/home/Projects/.config
```

Access the Debian system, get the Linux Libre kernel sources, configure it, get and build a more recent version of [GCC](https://gcc.gnu.org/), and build and install the Linux Libre kernel:

```sh
sudo jexec debian /bin/bash
```

```sh
cd /home/Projects

# @todo:
# git clone -b main --single-branch https://github.com/jackvz/gcc.git
# cd gcc
# ./configure \
#   --disable-shared \
#   --disable-bootstrap \
#   --disable-libstdcxx-pch \
#   --enable-languages=all \
#   --enable-libgomp \
#   --enable-lto \
#   --enable-threads=posix \
#   --enable-tls \
#   --with-gmp=/tmp/gcc \
#   --with-mpfr=/usr/lib/x86_64-kfreebsd-gnu \
#   --with-mpc=/usr/lib/x86_64-kfreebsd-gnu \
#   --with-libelf=/tmp/gcc \
#   --with-fpmath=sse
# make && make install
# cd ..

git clone -b main --single-branch https://github.com/jackvz/linux-libre.git
cp .config ./linux-libre/.config
cd linux-libre
scripts/config --disable MODULE_SIG
scripts/config --disable DEBUG_INFO
# make defconfig
# make menuconfig
make
make modules_install
make install
exit
```

### Stop the and Clear Out the Jails

Stop all running [FreeBSD Jail](https://docs.freebsd.org/en/books/handbook/jails/) systems and disable the service:

```sh
sudo service jail stop
sudo sysrc jail_enable="NO"
jls
```

Remove the Jails:

```sh
sudo rm -rf /jails/busybox
sudo rm -rf /jails/debian
```
