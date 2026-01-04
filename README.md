A minimal environment for linux kernel development to run in `qemu-kvm` on Ubuntu host.

Compile the kernel, use [BusyBox](https://www.busybox.net/) for rootfs and debug the kernel.

## Setup variables

```sh
ROOT=$PWD
TOOLCHAIN=arm-linux-gnueabihf
LINUX_MAJOR_VERSION=6
LINUX=linux-$LINUX_MAJOR_VERSION.18.3
BUSYBOX=busybox-1.36.1
```

## Download

```sh
wget https://cdn.kernel.org/pub/linux/kernel/v$LINUX_MAJOR_VERSION.x/$LINUX.tar.xz
tar -xf $LINUX.tar.xz
wget http://busybox.net/downloads/$BUSYBOX.tar.bz2
tar -xf $BUSYBOX.tar.bz2
```

## Install toolchain
```sh
sudo apt install -y gcc-$TOOLCHAIN gdb-multiarch
```

## Compile the kernel

### Configure the kernel
```sh
cd $ROOT/$LINUX
make ARCH=arm CROSS_COMPILE=$TOOLCHAIN- defconfig
```

### Enable debugging (optional)
```sh
echo '
# Debug enable
CONFIG_DEBUG_INFO=y
CONFIG_KGDB=y

# Debug info for symbolization.
CONFIG_DEBUG_INFO_DWARF4=y

# Memory bug detector
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y

RANDOMIZE_BASE=n
' >> .config
```

### Compile
```sh
make ARCH=arm CROSS_COMPILE=$TOOLCHAIN- -j$(nproc)
```

## Build rootfs
```sh
cd $ROOT/$BUSYBOX

make ARCH=arm CROSS_COMPILE=$TOOLCHAIN- defconfig
echo '
CONFIG_STATIC=y
' >> .config
sed -i 's/CONFIG_TC=y/CONFIG_TC=n/' .config

sudo rm -rf ../rootfs && mkdir ../rootfs
make ARCH=arm CROSS_COMPILE=$TOOLCHAIN- CONFIG_PREFIX=../rootfs -j$(nproc) install

cd ../rootfs
mkdir -p etc/init.d
echo '#!/bin/sh' > etc/init.d/rcS
chmod +x etc/init.d/rcS

ln -s bin/busybox init

mkdir dev && cd dev
sudo mknod -m 660 mem c 1 1
sudo mknod -m 660 tty2 c 4 2
sudo mknod -m 660 tty3 c 4 3
sudo mknod -m 660 tty4 c 4 4

cd ..
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../rootfs.cpio.gz
```

## Run the box

```sh
cd $ROOT
qemu-system-arm -M virt -m 256M -kernel $LINUX/arch/arm/boot/zImage \
	-initrd rootfs.cpio.gz -append "root=/dev/mem" -nographic
```
You can set `-enable-kvm` to accelerate a virtual machine only if the host CPU architecture and the guest CPU architecture are the same.
E.g. your CPU is Intel or AMD and you compile for `ARCH=x86`.

### Or run the box with debugging (optional)
```sh
cd $ROOT
qemu-system-arm \
	-M virt \
	-m 256M \
	-kernel $LINUX/arch/arm/boot/zImage \
	-initrd rootfs.cpio.gz \
	-append "root=/dev/mem" \
  -net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
  -net nic,model=e1000 \
	-nographic \
	-s \
	-S \
	-pidfile vm.pid \
  2>&1 | tee vm.log
```

### Cross-debugging (optional)
Start the GDB client in another terminal

```sh
cd $ROOT/$LINUX
gdb-multiarch -ex 'add-auto-load-safe-path scripts/gdb' -ex 'target remote :1234' ./vmlinux
# press `continue` and Ctrl-C to trigger an interrupt
```
