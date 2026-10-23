# LAB1 — Building and Booting an ARM Linux System

**Course:** LOS201 — Embedded Linux Systems, FPT University

**Contributor:** Phạm Đặng Khánh Duy

## Overview

This lab builds a minimal embedded Linux system for an ARMv7-A Cortex-A9 target using QEMU `vexpress-a9`. The work covers Linux kernel configuration, U-Boot experiments, a statically linked BusyBox userspace, initramfs packaging, and runtime verification.


## Contents

- [Quick Start](#quick-start)
- [Platform](#platform)
- [Repository Materials](#repository-materials)
- [Build Workflow](#build-workflow)
- [U-Boot Console](#u-boot-console)
- [Boot Linux in QEMU](#boot-linux-in-qemu)
- [Runtime Verification](#runtime-verification)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## Platform

| Component | Configuration |
|---|---|
| Development host | Ubuntu Linux, x86-64 |
| Target architecture | ARMv7-A |
| Target CPU | Cortex-A9 |
| QEMU machine | vexpress-a9 |
| Linux kernel | 5.15 |
| U-Boot | 2022.04 |
| BusyBox | 1.35.0, statically linked |
| Cross-toolchain prefix | arm-linux-gnueabihf- |
| Linux boot test | 512 MiB RAM, two virtual CPUs |
| Final initramfs filename | rootfs-final.cpio.gz |

The original completed workspace is `~/Embedded-Linux-Lab02/lab1`. The rebuild example below uses a separate directory, `~/Embedded-Linux-Lab02/lab1-rebuild`.

## Repository Materials

| Path | Purpose |
|---|---|
| [configs/kernel-5.15.config](configs/kernel-5.15.config) | Saved Linux kernel configuration |
| [configs/u-boot-2022.04.config](configs/u-boot-2022.04.config) | Saved U-Boot configuration |
| [configs/busybox-1.35.0.config](configs/busybox-1.35.0.config) | Saved BusyBox configuration |
| [rootfs/init](rootfs/init) | Initial userspace script, which executes `/sbin/init` |
| [rootfs/etc/inittab](rootfs/etc/inittab) | BusyBox init actions and serial-console shell |
| [rootfs/etc/init.d/rcS](rootfs/etc/init.d/rcS) | Virtual filesystem mounts and hostname setup |
| [rootfs/etc/hostname](rootfs/etc/hostname) | Target hostname |
| [output/zImage](output/zImage) | Tested ARM Linux kernel image |
| [output/vexpress-v2p-ca9.dtb](output/vexpress-v2p-ca9.dtb) | Board device tree |
| [output/rootfs-final.cpio.gz](output/rootfs-final.cpio.gz) | Tested BusyBox initramfs |

The `rootfs/` directory in this repository contains selected startup files. BusyBox binaries, applet symlinks, and the remaining filesystem directories are generated during the build. Upstream source trees are kept locally; the tested Linux boot images are included in `output/`.

## Quick Start

Boot the included, tested Linux images without rebuilding them. Install `qemu-system-arm` on the Ubuntu host first:

```bash
sudo apt install qemu-system-arm
qemu-system-arm \
  -M vexpress-a9 \
  -cpu cortex-a9 \
  -m 512M \
  -smp 2 \
  -nographic \
  -kernel lab1/output/zImage \
  -dtb lab1/output/vexpress-v2p-ca9.dtb \
  -initrd lab1/output/rootfs-final.cpio.gz \
  -append 'console=ttyAMA0,115200 rdinit=/init mem=512M'
Press **Enter** when prompted to activate the console. Expect the LAB01 welcome message and an interactive shell.

This command boots Linux directly through QEMU. See [U-Boot Console](#u-boot-console) for the separate bootloader experiment.

The [boot and runtime validation log](evidence/boot_validation.log) records Linux 5.15.0 on ARMv7, hostname `embedded-lab`, two online CPUs, mounted virtual filesystems, init at PID 1, and the required device nodes.

Exit QEMU with **Ctrl+A**, release the keys, then press **X**.

## Build Workflow

Run these commands in an Ubuntu host terminal. They describe a rebuild using the saved configurations; the original build and boot evidence is in the report. Compiler and host-package versions can affect builds of these older source releases.

### 1. Install host tools

```bash
sudo apt update
sudo apt install build-essential gcc-arm-linux-gnueabihf binutils-arm-linux-gnueabihf \
  qemu-system-arm device-tree-compiler u-boot-tools \
  flex bison bc libssl-dev libelf-dev libncurses-dev \
  cpio gzip xz-utils bzip2 wget
```

Check the cross-compiler and emulator:

```bash
arm-linux-gnueabihf-gcc --version
qemu-system-arm --version
```

### 2. Set workspace paths and obtain sources

Set `REPO_ROOT` to the location of your clone. Keep the exported variables available in the same terminal for the following host commands.

```bash
export REPO_ROOT="$HOME/LOS201-Embedded-Linux-Group"
export LAB_ROOT="$HOME/LOS201/lab01-rebuild"
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-

mkdir -p "$LAB_ROOT"/{downloads,kernel,uboot,busybox,rootfs,output}

wget -c -P "$LAB_ROOT/downloads" https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.tar.xz
wget -c -P "$LAB_ROOT/downloads" https://ftp.denx.de/pub/u-boot/u-boot-2022.04.tar.bz2
wget -c -P "$LAB_ROOT/downloads" https://busybox.net/downloads/busybox-1.35.0.tar.bz2

tar -xf "$LAB_ROOT/downloads/linux-5.15.tar.xz" -C "$LAB_ROOT/kernel"
tar -xf "$LAB_ROOT/downloads/u-boot-2022.04.tar.bz2" -C "$LAB_ROOT/uboot"
tar -xf "$LAB_ROOT/downloads/busybox-1.35.0.tar.bz2" -C "$LAB_ROOT/busybox"
```

### 3. Build Linux and the device tree

The original board configuration was based on `vexpress_defconfig` and then customized. Reusing the saved configuration preserves those choices.

```bash
cd "$LAB_ROOT/kernel/linux-5.15"
cp "$REPO_ROOT/lab01/configs/kernel-5.15.config" .config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- olddefconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
  KCFLAGS="-march=armv7-a" -j"$(nproc)" zImage dtbs modules
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules_prepare

cp arch/arm/boot/zImage "$LAB_ROOT/output/"
cp arch/arm/boot/dts/vexpress-v2p-ca9.dtb "$LAB_ROOT/output/"
```

`KCFLAGS="-march=armv7-a"` records the architecture override used to resolve the ARM instruction-set mismatch encountered during the original build. It is specific to this ARMv7 target. See [the troubleshooting evidence](evidence/vmscan_fix_test.log).

Important configuration features include VExpress support, ARMv7, SMP, initramfs support, PL011 serial console, and devtmpfs.

### 4. Build U-Boot

The original configuration was based on `vexpress_ca9x4_defconfig`.

```bash
cd "$LAB_ROOT/uboot/u-boot-2022.04"
cp "$REPO_ROOT/lab01/configs/u-boot-2022.04.config" .config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- olddefconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j"$(nproc)"
file u-boot
cp u-boot "$LAB_ROOT/output/"
```

The `u-boot` executable should be identified as a 32-bit ARM ELF file.

### 5. Build BusyBox and populate the root filesystem

```bash
cd "$LAB_ROOT/busybox/busybox-1.35.0"
cp "$REPO_ROOT/lab01/configs/busybox-1.35.0.config" .config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- oldconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j"$(nproc)"
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
  CONFIG_PREFIX="$LAB_ROOT/rootfs" install
file "$LAB_ROOT/rootfs/bin/busybox"
```

The saved configuration enables `CONFIG_STATIC=y`. The `file` output should identify an ARM executable that is statically linked.

Create the remaining directories and install the repository's startup files:

```bash
mkdir -p "$LAB_ROOT/rootfs"/{dev,proc,sys,tmp,root,etc/init.d}
cp -a "$REPO_ROOT/lab01/rootfs/." "$LAB_ROOT/rootfs/"
chmod 0755 "$LAB_ROOT/rootfs/init" "$LAB_ROOT/rootfs/etc/init.d/rcS"
chmod 1777 "$LAB_ROOT/rootfs/tmp"
printf 'root:x:0:0:root:/root:/bin/sh\n' > "$LAB_ROOT/rootfs/etc/passwd"
```

Prepare the initial console and null device nodes inside the rebuild root filesystem:

```bash
test -e "$LAB_ROOT/rootfs/dev/console" || \
  sudo mknod -m 600 "$LAB_ROOT/rootfs/dev/console" c 5 1
test -e "$LAB_ROOT/rootfs/dev/null" || \
  sudo mknod -m 666 "$LAB_ROOT/rootfs/dev/null" c 1 3
```

The startup sequence is:

1. The kernel starts `/init` because the boot arguments specify `rdinit=/init`.
2. `/init` uses `exec /sbin/init`, making BusyBox init PID 1.
3. BusyBox init reads `/etc/inittab` and executes `/etc/init.d/rcS`.
4. `rcS` mounts procfs, sysfs, devtmpfs, and tmpfs, then applies `/etc/hostname`.
5. The `ttyAMA0::askfirst:/bin/sh` action provides the interactive serial-console shell.

### 6. Package the initramfs

```bash
set -o pipefail
cd "$LAB_ROOT/rootfs"
find . -print0 \
  | cpio --null -o --format=newc --owner=0:0 \
  | gzip -9 > "$LAB_ROOT/output/rootfs-final.cpio.gz"
gzip -t "$LAB_ROOT/output/rootfs-final.cpio.gz"
```

The `newc` archive stores the filesystem contents and metadata. `--owner=0:0` sets the archived ownership to root. A successful `gzip -t` produces no output and returns exit status zero.

## U-Boot Console

Run this on the Ubuntu host, using `LAB_ROOT` from the build workflow:

```bash
qemu-system-arm \
  -M vexpress-a9 \
  -cpu cortex-a9 \
  -m 512M \
  -nographic \
  -kernel "$LAB_ROOT/output/u-boot"
```

Interrupt autoboot if necessary. At the U-Boot `=>` prompt, inspect the board and experiment with a boot argument variable:

```text
version
bdinfo
setenv bootargs 'console=ttyAMA0,115200 rdinit=/init mem=512M'
printenv bootargs
printenv bootcmd
```

These commands demonstrate the bootloader console and environment. Booting Linux through U-Boot additionally requires loading the kernel, DTB, and initramfs at valid, non-overlapping RAM addresses and using an appropriate boot command.

Exit QEMU with **Ctrl+A**, release the keys, then press **X**.

## Boot Linux in QEMU

The final Linux test documented in the report uses QEMU's direct kernel boot. QEMU loads `zImage`, the board DTB, and the initramfs; U-Boot is not executed in this particular command.

```bash
qemu-system-arm \
  -M vexpress-a9 \
  -cpu cortex-a9 \
  -m 512M \
  -smp 2 \
  -nographic \
  -kernel "$LAB_ROOT/output/zImage" \
  -dtb "$LAB_ROOT/output/vexpress-v2p-ca9.dtb" \
  -initrd "$LAB_ROOT/output/rootfs-final.cpio.gz" \
  -append 'console=ttyAMA0,115200 rdinit=/init mem=512M'
```

To run the author's existing build instead, set `LAB_ROOT="$HOME/LOS201/lab01"` before executing the boot command. That workspace contains both `rootfs.cpio.gz` and `rootfs-final.cpio.gz`; the final boot recorded in the report uses `rootfs-final.cpio.gz`.

| Option | Meaning |
|---|---|
| `-M vexpress-a9` | Select the emulated Versatile Express A9 board |
| `-m 512M` | Allocate 512 MiB of guest memory |
| `-smp 2` | Create two virtual CPUs |
| `-nographic` | Use the terminal for the serial console |
| `-kernel` | Load the Linux kernel image directly |
| `-dtb` | Supply the hardware description |
| `-initrd` | Supply the compressed initramfs |
| `rdinit=/init` | Select the initial userspace program in the initramfs |

Look for `Welcome to LAB01 Embedded Linux`. If prompted to activate the console, press **Enter** to obtain the BusyBox shell. Exit QEMU with **Ctrl+A**, then **X**.

## Runtime Verification

Run these commands **inside the QEMU guest**:

```sh
uname -a
hostname
cat /sys/devices/system/cpu/possible
cat /sys/devices/system/cpu/online
grep '^processor' /proc/cpuinfo
mount
ps
ls -l /dev/console /dev/null /dev/zero /dev/ttyAMA0
```

The submitted report records the following observations from the completed lab:

| Check | Recorded observation | Interpretation |
|---|---|---|
| Kernel and architecture | Linux 5.15.0, ARMv7 | The ARM kernel reached userspace |
| Hostname | `embedded-lab` | Startup applied `/etc/hostname` |
| Online CPUs | `0-1` | Two CPUs are active with `-smp 2` |
| Possible CPUs | `0-3` | The platform supports up to four CPUs |
| Filesystems | procfs, sysfs, devtmpfs, and tmpfs mounted | Startup initialized the required virtual filesystems |
| `/tmp` | tmpfs | Temporary data is held in RAM |
| Processes | init at PID 1 and an interactive shell | BusyBox init and console setup are functioning |
| Device nodes | console, null, zero, and ttyAMA0 available | Device access is present in `/dev` |

Use the [report](report/BAO_CAO_LAB01_Embedded_Linux.pdf) for runtime screenshots. The [kernel build log](evidence/kernel_build_success.log) documents compilation and is separate from runtime evidence.

## Troubleshooting

| Symptom | Explanation or check |
|---|---|
| Kernel panic while mounting the root filesystem | A kernel-only boot can reach this point without userspace. Supply the final initramfs with `-initrd` and select `/init`. |
| Init cannot be executed | Check that `/init`, `/bin/sh`, and `/sbin/init` exist in the archive and that startup scripts are executable with valid shebangs. |
| BusyBox exists but reports a missing file or interpreter | Inspect the ELF architecture and linking mode. A dynamic ARM binary also needs its matching loader and libraries. |
| Assembler rejects `dmb`, `ldrex`, or `strex` | Check the ARM configuration and compiler flags. The original workaround used `KCFLAGS="-march=armv7-a"`; see the saved troubleshooting log. |
| `possible` is `0-3` while `online` is `0-1` | These describe supported CPU indices and currently active CPU indices, respectively. |
| A host command cannot find a path | Confirm whether you are using the original workspace or the separate rebuild workspace, and check `REPO_ROOT` and `LAB_ROOT`. |

## References

- Lecturer-provided LAB01 specification.
- [Linux 5.15 Kbuild documentation](https://www.kernel.org/doc/html/v5.15/kbuild/kbuild.html).
- [U-Boot documentation](https://docs.u-boot.org/en/v2022.04/).
- [BusyBox documentation](https://busybox.net/downloads/BusyBox.html).
- [QEMU Versatile Express documentation](https://www.qemu.org/docs/master/system/arm/vexpress.html).

[Back to the group repository overview](../README.md)
