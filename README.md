# TwinVisor Functional Prototype

## Introduction

This is a functional prototype of the SOSP'21 paper **TwinVisor: Hardware-isolated Confidential Virtual Machines for ARM** based on ARM Fixed Virtual Platform (FVP). You can find our paper [here](https://dl.acm.org/doi/abs/10.1145/3477132.3483554).

Due to the confidentiality agreement, we cannot release the proprietary code of the performance testing prototype on Hisilicon Kirin 990 hardware. 

The copyright of TwinVisor and its prototype belongs to Institute of Parallel and Distributed Systems (IPADS) from Shanghai Jiao Tong University (SJTU).

The code is based on [OP-TEE project](https://github.com/OP-TEE/manifest/blob/master/fvp.xml) and tested on Ubuntu 20.04 & Ubuntu 18.04 (x86-64).

If you encounter any questions, please contact us via e-mail:

> Dingji Li: dj_lee@sjtu.edu.cn, Zeyu Mi: yzmizeyu@sjtu.edu.cn

## Prerequesites

1. Update the package managers database.

```bash
sudo apt-get update
```

2. Install the following packages.

```bash
sudo apt-get install android-tools-adb android-tools-fastboot autoconf \
        automake bc bison build-essential cmake ccache codespell \
        cscope curl device-tree-compiler \
        expect flex ftp-upload gdisk iasl libattr1-dev libcap-dev \
        libfdt-dev libftdi-dev libglib2.0-dev libgmp-dev libhidapi-dev \
        libmpc-dev libncurses5-dev libpixman-1-dev libssl-dev libtool make \
        mtools netcat ninja-build python-crypto python3-crypto python-pyelftools \
        python3-pycryptodome python3-pyelftools python3-serial python \
        rsync unzip uuid-dev xdg-utils xterm xz-utils zlib1g-dev
```

## Setup

Clone this repo and update submodules.

```bash
export TV_ROOT=$(pwd)
git submodule update --init --recursive
```

Download [FVP Base](https://ipads.se.sjtu.edu.cn:1313/f/73e2572b19a24b32817c/?dl=1) and extract to ``FVP_Base_RevC-2xAEMv8A`` in ``$TV_ROOT`` directory.

```bash
# MD5: 0bd25ec5005c600d6f9b8ebc41aff0ab  FVP_Base_RevC-2xAEMv8A.tar.gz
wget -c https://ipads.se.sjtu.edu.cn:1313/f/73e2572b19a24b32817c/?dl=1 -O $TV_ROOT/FVP_Base_RevC-2xAEMv8A.tar.gz

tar xzf $TV_ROOT/FVP_Base_RevC-2xAEMv8A.tar.gz -C $TV_ROOT/
```

**NOTE:** Following steps should be done in the ``build`` directory.

```bash
cd $TV_ROOT/build
```

First, get the toolchains for aarch64.

```bash
make toolchains -j2

# Test toolchains
ls ../toolchains
```

Next, download the disk image of the prototype. For simplicity, this disk image already contains QEMU, kernel images and rootfs for S-VMs. You can also compile the source code of QEMU and guest kernel, and copy them into the disk image by yourself.

```bash
mkdir -p ../out
# The tarball is about 8GB, and the disk image after decompressed is about 40GB
# MD5: c16d78505fa16a8520ee08a05d1debf7  boot.tar.gz
wget -c https://ipads.se.sjtu.edu.cn:1313/f/73350e5ff3e440a98081/?dl=1 -O ../out/boot.tar.gz

tar xzf ../out/boot.tar.gz -C ../out

# Test boot.img and the guest kernel image
ls ../out/boot.img
ls ../out/Image # for md5 integrity checking
```

Then, compile all of them.

```bash
make all -j$(nproc)
```

Finally, run it.

```bash
make run-only
```

## Run

1. Login the host Linux (N-visor) with username ``root``

2. Mount the rootfs and chroot in host Linux

    ```bash
    mount /dev/vda2 /root
    chroot /root bash
    ./init.sh
    ```

3. Run the guest Linux (S-VM)

   ```bash
   cd /test
   ./s-vm0.sh
   ```

4. Mount the rootfs and chroot in guest Linux

   ```bash
   mount /dev/vda /root
   chroot /root bash
   ```

5. Some workload examples are under ``/test/`` in guest rootfs

    ```bash
    # Take FileIO as an example
    ./fileio.sh
    ```

## Guest QEMU

To build QEMU for S-VMs:

```bash
# For cross compile, take ubuntu **focal** as an example
dpkg --add-architecture arm64
echo "deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports/ focal main multiverse restricted universe" >> /etc/apt/sources.list
echo "deb [arch=arm64] http://ports.ubuntu.com/ubuntu-ports/ focal-updates main multiverse restricted universe" >> /etc/apt/sources.list
apt update
apt install crossbuild-essential-arm64 pkg-config-aarch64-linux-gnu
apt install libpixman-1-dev:arm64 libglib2.0-dev:arm64

mkdir -p $TV_ROOT/qemu/build
cd $TV_ROOT/qemu/build

# Cross compile with static link
../configure --target-list=aarch64-softmmu \
    --cross-prefix=aarch64-linux-gnu- \
    --static
make -j$(nproc)

# Test output
../aarch64-softmmu/qemu-system-aarch64 --version

# Copy the QEMU into the disk image ``$TV_ROOT/out/boot.img``
```

## Guest Linux

To build guest kernel image:

```bash
# For cross compile
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

cd $TV_ROOT/guest-linux
cp prototype-config .config
make all -j$(nproc)

# Test output
ls arch/arm64/boot/Image

# Copy the kernel image to ``$TV_ROOT/out/Image``,
# then re-compile the S-visor under ``$TV_ROOT/build/`` with ``make s-visor``
cp arch/arm64/boot/Image $TV_ROOT/out/
# Also copy the kernel image into the disk image ``$TV_ROOT/out/boot.img``
```
