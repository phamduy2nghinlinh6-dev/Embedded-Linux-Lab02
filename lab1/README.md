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
| QEMU machine | `vexpress-a9` |
| Linux kernel | 5.15 |
| U-Boot | 2022.04 |
| BusyBox | 1.35.0, statically linked |
| Cross-toolchain prefix | `arm-linux-gnueabihf-` |
| Linux boot test | 512 MiB RAM, two virtual CPUs |
| Final initramfs filename | `rootfs-final.cpio.gz` |

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
