# K230 Boot Assets

Pre-built Linux boot images for QEMU's Canaan K230 CanMV machine.

The assets cover two build systems and two QEMU boot flows:

* direct Linux boot: QEMU loads OpenSBI, Linux, initrd, and DTB.
* U-Boot boot: QEMU starts SDK U-Boot, then U-Boot starts OpenSBI/Linux with
  `bootm`.

**Emulator target**: QEMU `-M k230` with one T-HEAD C908 little core, 2 GiB
RAM, CLINT, PLIC, K230 watchdogs, and 16550-compatible UARTs.

**QEMU K230 branch**: <https://github.com/zevorn/qemu/tree/chao-k230-v7>

The command lines below follow the QEMU `docs/system/riscv/k230.rst` boot
model.

## Version Matrix

| Component | Version | Source |
|-----------|---------|--------|
| Yocto (poky) | scarthgap 5.0.17 | `yocto-5.0.17-82-ga53cae3de9` |
| meta-riscv | scarthgap | `76727db` |
| meta-k230 | scarthgap | Custom BSP layer |
| Linux (yocto) | 6.18.28 | mainline, `defconfig` + K230 fragment |
| BusyBox (yocto) | 1.36.1 | poky, statically linked |
| k230_sdk | v2.0 | `7e302f733` |
| Buildroot (sdk) | 2020.02 | k230_sdk bundled |
| Linux (sdk) | 5.10.4 | T-HEAD vendor kernel, rebuilt for QEMU PTEs |
| U-Boot (sdk) | 2022.10 | k230_sdk, M-mode U-Boot |
| OpenSBI (QEMU) | 1.7 | bundled in QEMU `pc-bios/` |
| OpenSBI (SDK) | 0.9 | wrapped by `common/fw_jump.uImage` |

## Directory Layout

```
k230-boot-assets/
├── common/
│   ├── u-boot              # SDK U-Boot ELF for QEMU -bios
│   └── fw_jump.uImage      # SDK OpenSBI fw_jump.bin wrapped for bootm
├── buildroot/
│   ├── direct-boot/
│   │   ├── Image           # SDK Linux 5.10.4
│   │   ├── k230.dtb        # Original SDK DTB
│   │   ├── k230-qemu.dtb   # Minimal QEMU DTB used by the commands below
│   │   └── rootfs.cpio.gz
│   └── uboot-boot/
│       ├── Image
│       ├── k230.dtb        # Original SDK DTB
│       ├── k230-qemu.dtb   # Minimal QEMU DTB used by the commands below
│       ├── rootfs.cpio.gz
│       ├── rootfs.ext4     # SDK rootfs artifact, not used without storage
│       └── u-boot.bin      # Raw SDK U-Boot artifact, not used by -bios
└── yocto/
    ├── direct-boot/
    │   ├── Image           # Yocto Linux 6.18.28
    │   ├── k230-canmv.dtb  # Minimal QEMU DTB
    │   └── rootfs.cpio.gz
    └── uboot-boot/
        ├── Image
        ├── k230-canmv.dtb
        └── rootfs.cpio.gz
```

## QEMU Boot Status

| Build System | Boot Mode | QEMU Status |
|-------------|-----------|-------------|
| Yocto | Direct boot | Boots to the initramfs shell |
| Yocto | U-Boot boot | Boots to the initramfs shell through SDK U-Boot |
| Buildroot | Direct boot | Boots to the Buildroot login prompt |
| Buildroot | U-Boot boot | Boots to the Buildroot login prompt through SDK U-Boot |

Buildroot userspace may print non-fatal module symbol warnings because the SDK
rootfs carries extra vendor modules. Those warnings do not block the login
prompt.

## QEMU Boot Commands

Set `QEMU` to the QEMU binary you want to test. For a local QEMU build:

```bash
ROOT=~/yocto/k230-boot-assets
QEMU=~/qemu/build/qemu-system-riscv64
OPENSBI=~/qemu/pc-bios/opensbi-riscv64-generic-fw_dynamic.bin
```

### Yocto Direct Boot

```bash
DST=$ROOT/yocto/direct-boot

$QEMU -M k230 \
    -bios "$OPENSBI" \
    -kernel "$DST/Image" \
    -dtb "$DST/k230-canmv.dtb" \
    -initrd "$DST/rootfs.cpio.gz" \
    -append "console=ttyS0,115200 earlycon=sbi" \
    -nographic -no-reboot
```

Expected final output:

```
meta-k230 initramfs starting...
Dropping to shell...
~ #
```

### Buildroot Direct Boot

```bash
DST=$ROOT/buildroot/direct-boot

$QEMU -M k230 \
    -bios "$OPENSBI" \
    -kernel "$DST/Image" \
    -dtb "$DST/k230-qemu.dtb" \
    -initrd "$DST/rootfs.cpio.gz" \
    -append "console=ttyS0,115200 earlycon=sbi cma=0" \
    -nographic -no-reboot
```

Expected final output:

```
Welcome to Buildroot
canaan login:
```

### Yocto U-Boot Boot

This flow uses SDK U-Boot only as the boot loader. The Yocto kernel, initrd,
and DTB are loaded directly into RAM by QEMU.

```bash
COMMON=$ROOT/common
DST=$ROOT/yocto/uboot-boot
INITRD_END=$(printf "0x%x" $((0x0a100000 + $(stat -c %s "$DST/rootfs.cpio.gz"))))

echo "Use this U-Boot initrd end: $INITRD_END"

$QEMU -M k230 \
    -bios "$COMMON/u-boot" \
    -device loader,file="$COMMON/fw_jump.uImage",addr=0xc100000,force-raw=on \
    -device loader,file="$DST/Image",addr=0x8200000,force-raw=on \
    -device loader,file="$DST/rootfs.cpio.gz",addr=0xa100000,force-raw=on \
    -device loader,file="$DST/k230-canmv.dtb",addr=0xa000000,force-raw=on \
    -nographic -no-reboot
```

Press Enter to stop autoboot. At the `K230#` prompt, run:

```text
setenv bootargs console=ttyS0,115200 earlycon=sbi
fdt addr 0xa000000
fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0xa100000>
fdt set /chosen linux,initrd-end <0x0 INITRD_END>
bootm 0xc100000 - 0xa000000
```

Replace `INITRD_END` with the value printed by the host command.

### Buildroot U-Boot Boot

This flow uses SDK U-Boot and the SDK kernel/rootfs, but it passes the minimal
QEMU DTB. The original SDK DTB describes real K230 peripherals that are not
modeled by QEMU.

```bash
COMMON=$ROOT/common
DST=$ROOT/buildroot/uboot-boot
INITRD_END=$(printf "0x%x" $((0x0a100000 + $(stat -c %s "$DST/rootfs.cpio.gz"))))

echo "Use this U-Boot initrd end: $INITRD_END"

$QEMU -M k230 \
    -bios "$COMMON/u-boot" \
    -device loader,file="$COMMON/fw_jump.uImage",addr=0xc100000,force-raw=on \
    -device loader,file="$DST/Image",addr=0x8200000,force-raw=on \
    -device loader,file="$DST/rootfs.cpio.gz",addr=0xa100000,force-raw=on \
    -device loader,file="$DST/k230-qemu.dtb",addr=0xa000000,force-raw=on \
    -nographic -no-reboot
```

Press Enter to stop autoboot. At the `K230#` prompt, run:

```text
setenv bootargs console=ttyS0,115200 earlycon=sbi cma=0
fdt addr 0xa000000
fdt resize 8192
fdt set /chosen linux,initrd-start <0x0 0xa100000>
fdt set /chosen linux,initrd-end <0x0 INITRD_END>
bootm 0xc100000 - 0xa000000
```

Replace `INITRD_END` with the value printed by the host command.

Exit QEMU with `Ctrl+A X`.

## Buildroot Notes

The SDK Linux Image in this repository was rebuilt with standard RISC-V PTE
bits. The original vendor kernel uses T-HEAD C9xx private MAEE page table
attributes, which QEMU's generic RISC-V MMU does not implement.

The `k230-qemu.dtb` files intentionally describe only the devices currently
modeled by QEMU's `k230` machine. The original `k230.dtb` files are kept as SDK
reference artifacts but are not used by the working command lines above.

## Rebuilding `common/fw_jump.uImage`

```bash
SDK=~/k230_sdk/output/k230_canmv_defconfig
MKIMAGE=$SDK/little/buildroot-ext/host/bin/mkimage

$MKIMAGE \
    -A riscv -O linux -T kernel -C none \
    -a 0x8000000 -e 0x8000000 -n opensbi \
    -d "$SDK/images/little-core/fw_jump.bin" \
    common/fw_jump.uImage
```

## File Origins

| File | Origin | Description |
|------|--------|-------------|
| `common/u-boot` | k230_sdk v2.0 | SDK U-Boot ELF used by QEMU `-bios` |
| `common/fw_jump.uImage` | k230_sdk v2.0 | SDK OpenSBI `fw_jump.bin` wrapped for `bootm` |
| `buildroot/*/Image` | k230_sdk v2.0 | T-HEAD Linux 5.10.4 rebuilt for QEMU PTEs |
| `buildroot/*/k230.dtb` | k230_sdk v2.0 | Original SDK DTB, kept for reference |
| `buildroot/*/k230-qemu.dtb` | meta-k230 | Minimal QEMU-compatible K230 DTB |
| `buildroot/*/rootfs.cpio.gz` | k230_sdk v2.0 | SDK Buildroot initramfs |
| `buildroot/uboot-boot/rootfs.ext4` | k230_sdk v2.0 | SDK ext4 rootfs artifact |
| `buildroot/uboot-boot/u-boot.bin` | k230_sdk v2.0 | Raw SDK U-Boot artifact |
| `yocto/*/Image` | meta-k230 | Mainline Linux 6.18.28 |
| `yocto/*/k230-canmv.dtb` | meta-k230 | Minimal QEMU-compatible K230 DTB |
| `yocto/*/rootfs.cpio.gz` | meta-k230 | BusyBox initramfs |
