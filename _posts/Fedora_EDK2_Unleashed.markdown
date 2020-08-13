
This page have detailed instructions on how to test complete EFI boot flow using EDK2 firmware on HiFive Unleashed.

### Prerequisites:

Docker - To build edk2

Linux kernel source - To Build Linux kernel Image

Pre-built Fedora image

### 1. Build Linux kernel:

```

#clone the tree

git clone https://github.com/atishp04/linux/tree/uefi_riscv_5.10_v5

make defconfig

#if you want to run firmware test suite(fwts) enable below config from menuconfig
CONFIG_EFI_TEST=y

#For efi runtime page table mappings
CONFIG_PTDUMP_DEBUGFS=y

#Build the kernel now
ARCH=riscv CROSS_COMPILE=<cross compile prefix> make -j32
```

### Build EDK2:
These instructions are copied from instructions provided by Daniel[1].

```
# Create a working directory and enter it
mkdir edk2-riscv
cd edk2-riscv

# Clone all repositories with submodules
git clone https://github.com/JohnAZoidberg/riscv-edk2-docker
git clone --depth=1 --recurse-submodules --branch=riscv-dt-fixup-for-atish https://github.com/riscv/riscv-edk2 edk2
git clone --depth=1 --recurse-submodules --branch=riscv-dt-fixup-for-atish https://github.com/riscv/riscv-edk2-platforms edk2-platforms

# Download and unpack the RISC-V
wget https://github.com/riscv/riscv-uefi-edk2-docs/raw/master/gcc-riscv-edk2-ci-toolchain/gcc-riscv-9.2.0-2020.04-x86_64_riscv64-unknown-gnu.tar.xz
tar xf gcc-riscv-9.2.0-2020.04-x86_64_riscv64-unknown-gnu.tar.xz
rm gcc-riscv-9.2.0-2020.04-x86_64_riscv64-unknown-gnu.tar.xz

# Now you can work from within this repo
cd riscv-edk2-docker

# Build the docker container
docker build -t edk2 .

# Start the docker container
./rundocker.sh

# Make linux iso for ramdisk
mkfs.msdos -C linux.iso 14000
sudo losetup /dev/loop0 linux.iso
sudo mount /dev/loop0 /mnt
# Copy the kernel and initramfs that you built previously
sudo cp linux-riscv64.efi /mnt
sudo umount /mnt
sudo losetup -d /dev/loop0
mv linux.iso ../edk2-platforms/Silicon/RISC-V/ProcessorPkg/Universal/EspRamdisk/linux.iso

# Copy the device tree into EDK2 build directory
cp U540.dtb ../edk2-platforms/Platform/RISC-V/PlatformPkg/Universal/Sec/Riscv64/U540.dtb

# Build EDK2
./build-in-docker.sh
```

### Prepare the sdcard:

Download the Fedora image
```
wget http://fedora.riscv.rocks/kojifiles/work/tasks/944/320944/Fedora-Developer-Rawhide-20200108.n.0-sda.raw.xz
```
Extract the 2nd partition of the image
```
guestfish add Fedora-Developer-Rawhide-20200108.n.0-sda.raw : run : download /dev/sda2 Fedora-Developer-Rawhide-20200108.n.0-sda2.raw
```

We need more than 64MB in the first partition as the EDK2 firmware may be bigger. You can adjust sgdisk argument
based on the EDK2 firmware size. **/dev/sda** is the sdcard mount path. It may be **different** on your machine.

```
sudo sgdisk --clear --new=1:2048:270332  --change-name=1:bootloader --typecode=1:2E54B353-1271-4842-806F-E436D6AF6985 \
			--new=2:292864:  --change-name=2:root   --typecode=2:0FC63DAF-8483-4772-8E79-3D69D8477DE4 /dev/sda
```


Copy the EDK2 firmware binary to first partition of the sdcard
```
sudo dd if=U540.fd of=/dev/sda1 bs=1M
```

Copy the Fedora rootfs to the 2nd partition of the sdcard
```
sudo dd if=Fedora-Developer-Rawhide-20200108.n.0-sda2.raw of=/dev/sda2 bs=1M
```

You can remove the sdcard from the host and insert into the HiFive Unleashed board.
After reset, it should boot EDK2 in few minutes.
Once you are at EDK2 shell prompt, mount the ramdisk and opendisk.
```
embeddedramdisk 4f2f3d7b-35ef-411b-9d26-e76ecacbaf8b
map -r
fs0:

#Start Linux with required arguments
linux-riscv64.efi root=/dev/mmcblk0p2 rootwait console=ttySIF0 earlycon
```

[1] https://github.com/JohnAZoidberg/riscv-edk2-docker/
