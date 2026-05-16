# LG C1 O20 kexec / mmcoops / Android Port Research

> Research log for LG OLED C1 (`OLED65C17LB`) webOS 6.0 / O20B0 platform.
>
> Goal: determine whether a custom kernel can be booted through `kexec`, whether LG's crash/dump paths can help debug it, and what this implies for a possible Android/Linux userspace port.

---

## 1. Device under test

Runtime information collected from the TV:

```text
Model:        OLED65C17LB
SoC/chip:     O20B0
Firmware:     sver=4.05.48 / bver=4.05.48
Kernel:       4.4.84-229.1.kavir.2
Architecture: arm64 / aarch64
Rootfs:       /dev/mmcblk0p27, squashfs
```

Observed `/proc/cmdline`:

```text
root=/dev/mmcblk0p27 ro rootfstype=squashfs vmalloc=1216M
hma=856M@0x047800000;360M@0x0a9800000
lg1k.use_vmap=1 hma.use_vmap=1 use_vmap=1 cma_on
ethaddr=74:E6:B8:A1:69:BF rev=1
console=ttyAMA0,115200n81
chip=O20B0
sver=4.05.48 bver=4.05.48
quiet loglevel=0
snapshot resume=/dev/mmcblk0p51
devtmpfs.mount=1 rootwait
emmc_size=0x1d2000000
sbkey=0x7d0e0000
portProtection
mmcoops=dump
wdtlog=dump@1M
modelName=OLED65C17LB
...
cmdEnd
```

Important command line flags:

```text
mmcoops=dump
wdtlog=dump@1M
snapshot
resume=/dev/mmcblk0p51
quiet loglevel=0
console=ttyAMA0,115200n81
```

---

## 2. Original objective

The initial goal was to test whether the LG C1 can boot a custom Linux kernel via `kexec`.

The working directory on the TV was:

```text
/tmp/usb/sda/sda1/lgc1-kexec
```

Files available on USB included:

```text
Image
initramfs.cpio.gz
lgc1-running.dtb
kexec
ld-linux-aarch64.so.1
libc.so.6
libz.so.1
cmdline-webos.txt
cpuinfo.txt
meminfo.txt
iomem.txt
partitions.txt
reserved-memory.txt
dmesg-before-kexec.txt
...
```

The local `kexec` binary was executed with its own dynamic loader/libraries from USB:

```sh
./ld-linux-aarch64.so.1 --library-path . ./kexec ...
```

---

## 3. Runtime kernel capabilities

The runtime kernel supports normal `kexec`:

```text
CONFIG_KEXEC=y
CONFIG_KEXEC_CORE=y
```

But does **not** support upstream crash dump / kdump:

```text
# CONFIG_CRASH_DUMP is not set
# CONFIG_PSTORE is not set
```

Other relevant config found later in LG's open source package:

```text
CONFIG_DPM_WATCHDOG=y
CONFIG_DPM_WATCHDOG_TIMEOUT=5
CONFIG_ARM_SP805_WATCHDOG=y
CONFIG_MMCOOPS=y
# CONFIG_WATCHDOG_NOWAYOUT is not set
# CONFIG_PANIC_ON_OOPS is not set
CONFIG_PANIC_TIMEOUT=0
```

This matches the live system behavior:

```text
/sys/kernel/kexec_loaded          exists
/sys/kernel/kexec_crash_loaded    exists but remains 0
/sys/kernel/kexec_crash_size      exists but remains 0
```

---

## 4. Memory map and crashkernel state

From `/proc/iomem`:

```text
00000000-337fffff : System RAM
47800000-7cffffff : System RAM
80000000-bfffffff : System RAM
```

No `crashkernel=` argument was present in `/proc/cmdline`:

```sh
tr ' ' '\n' < /proc/cmdline | grep -i crash
# no output
```

Crash kernel state:

```text
/sys/kernel/kexec_crash_loaded = 0
/sys/kernel/kexec_crash_size   = 0
```

This means upstream `kexec -p` cannot be expected to work without a reserved crashkernel memory region.

---

## 5. Normal kexec test results

A normal `kexec -l` could load the image.

Representative command:

```sh
cd /tmp/usb/sda/sda1/lgc1-kexec

CMDLINE='console=ttyAMA0,115200n81 earlycon=pl011,mmio32,0xfe000000 ignore_loglevel loglevel=8 rdinit=/init root=/dev/ram0 rw maxcpus=1 nr_cpus=1 reset_devices irqpoll'

./ld-linux-aarch64.so.1 --library-path . ./kexec -l ./Image \
  --dtb=./lgc1-running.dtb \
  --initrd=./initramfs.cpio.gz \
  --append="$CMDLINE" \
  --debug
```

Then:

```sh
cat /sys/kernel/kexec_loaded
# 1
```

But executing it:

```sh
./ld-linux-aarch64.so.1 --library-path . ./kexec -e
```

caused the TV to reboot after roughly 10–15 seconds.

No useful output was visible because:

```text
No UART available.
No visible framebuffer output.
No pstore.
No crashkernel/kdump.
```

The TV eventually booted back into webOS.

---

## 6. kexec -p / crash kernel path

The TV has a `kdump.service` unit and a script:

```text
/lib/systemd/system/kdump.service
/lib/systemd/system/scripts/kdump.sh
```

The script expects:

```text
crashkernel in /proc/cmdline
/usr/sbin/kexec
/boot/kdImage-1.0.14-162
/sbin/init.kdump
```

The relevant logic observed in the LG kdump script:

```sh
if grep -q -s "crashkernel" $proc_cmd
then
    exec /usr/sbin/kexec -i -p /boot/kdImage-${kdver} ...
fi
```

But on this retail system:

```text
/proc/cmdline has no crashkernel
/usr/sbin/kexec does not exist
/boot/kdImage-1.0.14-162 does not exist
/sbin/init.kdump does not exist
```

So LG's `kdump.service` is present but non-functional on this firmware.

Conclusion:

```text
LG kdump path exists as a leftover/stub, but is unusable on this retail system.
```

---

## 7. Faultmanager / PowerOnReason evidence

After failed `kexec -e` attempts, webOS faultmanager logs consistently reported:

```text
PowerOnReason=cpuAbnormal
reason":"cpuAbnormal"
power on reason string:cpuAbnormal, enum:14
```

Examples from faultmanager/webOS logs:

```text
getPowerOnReason : { "returnValue": true, "reason": "cpuAbnormal" }
power on reason string:cpuAbnormal, enum:14
tvpowerd POWERON_REASON cpuAbnormal is power on reason
```

Other normal boot reasons were also observed:

```text
remoteKey
micomPower
```

Therefore `cpuAbnormal` is not a generic value; it is a meaningful MICOM/platform-level classification.

Observed MICOM raw mappings from logs:

```text
0x2d -> cpuAbnormal
0x31 -> micomPower
0x20 -> remoteKey
```

Interpretation:

```text
kexec -e causes a platform/MICOM abnormal CPU reset, not a normal remote/power boot reason.
```

---

## 8. LG dump / mmcoops / wdtlog infrastructure

The live kernel image contained strings:

```text
mmcoops:Oops/panic log will be stored at %s partition (offset: 0x%llx)
wdtlog:pmlog/legacy log will be stored at %s partition (offset: 0x%llx)
wdtlog: Start second kernel
wdtlog: file copy completed: %u bytes
mmcoops=
wdtlog=
```

The boot command line contains:

```text
mmcoops=dump
wdtlog=dump@1M
```

This indicates LG has a proprietary crash/logging path separate from upstream `pstore`.

Kernel symbols found from `/proc/kallsyms` and later confirmed in `System.map`:

```text
wdt_log_kexec_setup
wdt_log_save
wdt_log_ready
wdt_detect_isr
wdt_detect_proc
mmcoops_init
wdt_log_init
sys_proc_read_kdump_meminfo
diag_panic_event
diag_invoke_reset
```

---

## 9. Identifying the LG `dump` partition

The standard Linux sysfs partition information did not expose partition names.

`blkid` only recognized some partitions as `squashfs` or `ext4`; the `dump` partition was raw and did not appear by name.

A naive GPT parser failed because `/dev/mmcblk0` does **not** start with a standard GPT header. The first MiB contained bootloader / DDR init strings such as:

```text
reset_reinit_ddr
disable_retention_by_micom
check_retention_from_ddr
shadow_rom
ddr_rom_bin hash ok
[pmtRSA]
boot partition offset 0x%x
```

The actual LG partition table was found in tiny partitions:

```text
/dev/mmcblk0p2
/dev/mmcblk0p3
```

These contain LG `partinfo` data, including names:

```text
secureboot
partinfo
mapbak
boot
kernel
rootfs
dbboot
dump
hist
reserved
```

Parsing `p2.bin` revealed:

```text
FOUND_DUMP_NAME=dump
FOUND_DUMP_OFFSET=0xe7980000
FOUND_DUMP_SIZE=0xa00000
FOUND_DUMPDEV=/dev/mmcblk0p52
```

Cross-check against Linux partition table:

```text
/dev/mmcblk0p52
start = 7609344 sectors
size  = 20480 sectors

7609344 * 512 = 0xe7980000
20480   * 512 = 0x00a00000 = 10 MiB
```

Conclusion:

```text
LG dump partition = /dev/mmcblk0p52
size = 10 MiB
```

---

## 10. Full partition map observed

Abbreviated relevant partition map:

```text
p2      partinfo
p3      partinfo backup
p4      lxboot / bootloader/config area
p11     env_nvm
p21     kernel image / compressed kernel area
p27     rootfs active, squashfs
p38     likely alternate rootfs slot
p51     snapshot/resume
p52     dump
p53     hist
p54     db8, ext4
p55     data, ext4
p56     apps, ext4
```

Important confirmed entries:

```text
p51 = resume/snapshot from cmdline: resume=/dev/mmcblk0p51
p52 = dump
p54 = db8
p55 = data
p56 = apps
```

---

## 11. Testing whether `/dev/mmcblk0p52` actually works

Initially, `/dev/mmcblk0p52` was all zeroes:

```text
p52 is all zero
strings p52-dump-full.bin -> no output
```

A controlled Linux panic was triggered:

```sh
echo 1 > /proc/sys/kernel/sysrq
sync
echo c > /proc/sysrq-trigger
```

After reboot, `/dev/mmcblk0p52` changed and contained a persistent kernel panic log:

```text
sysrq: SysRq : Trigger a crash
Unable to handle kernel NULL pointer dereference at virtual address 00000000
Internal error: Oops: 96000046 [#1] PREEMPT SMP
Kernel panic - not syncing: Fatal exception
Call trace:
--- diagnosis panic event ----
```

Conclusion:

```text
LG mmcoops/dump works for normal Linux panics.
```

This is extremely important because it proves the `dump` partition is real and usable.

---

## 12. Testing kexec against the dump partition

A clean before/after test was performed:

1. Copy `/dev/mmcblk0p52` before `kexec -e`.
2. Run `kexec -e`.
3. Let the TV reboot naturally.
4. Copy `/dev/mmcblk0p52` after reboot.
5. Compare both files.

Result:

```text
P52_UNCHANGED
p52-after-interesting.txt -> empty
cmp diff -> empty
```

MD5 before and after matched for the tested run:

```text
p52 before == p52 after
```

Faultmanager still showed `cpuAbnormal` after the reboot.

Conclusion:

```text
A normal SysRq panic populates /dev/mmcblk0p52.
A failed kexec -e does not change /dev/mmcblk0p52.
```

Therefore:

```text
The kexec failure does not go through the normal Linux panic/oops/mmcoops path.
```

---

## 13. LG Open Source package

The correct LG open source package was identified as:

```text
Category:    TV/AV > TV
Model:       OLED65C17LB
Description: webOS 6.0 ON 1.0
```

This matches the TV:

```text
modelName=OLED65C17LB
webOS 6.0 / C1 2021
O20B0
```

The package was split into:

```text
webOS 6.0 ON_1.0_1.tar.gz   ~3.1G
webOS 6.0 ON_1.0_2.tar.gz   ~89M
webOS 6.0 ON_1.0_3.tar.gz   ~1.7G
```

The relevant SoC package was found inside:

```text
webOS 6.0 ON_1.0_2.tar.gz
└── soc/o20n_mt7921.tgz
```

After extraction, relevant tree:

```text
o20n_mt7921/linux-rockhopper_build_o20/
o20n_mt7921/linux-rockhopper_build_o20/kernel-source/
o20n_mt7921/linux-rockhopper_build_o20/.config
o20n_mt7921/linux-rockhopper_build_o20/System.map
```

The kernel source/build tree corresponds to:

```text
linux-rockhopper
4.4.84-214-r19.20 build metadata
O20 platform
```

The live TV kernel was:

```text
4.4.84-229.1.kavir.2
```

So the open source package is clearly the correct platform family, even if not byte-identical to the installed binary.

---

## 14. Source package findings

The package confirms the important config:

```text
CONFIG_KEXEC_CORE=y
CONFIG_KEXEC=y
# CONFIG_CRASH_DUMP is not set
CONFIG_DPM_WATCHDOG=y
CONFIG_DPM_WATCHDOG_TIMEOUT=5
# CONFIG_WATCHDOG_NOWAYOUT is not set
CONFIG_ARM_SP805_WATCHDOG=y
CONFIG_MMCOOPS=y
# CONFIG_PSTORE is not set
# CONFIG_PANIC_ON_OOPS is not set
CONFIG_PANIC_TIMEOUT=0
```

`System.map` contains:

```text
ffffffc00009b6c8 T machine_kexec_cleanup
ffffffc00009b6d0 T machine_kexec_prepare
ffffffc00009b708 T machine_kexec
ffffffc00066b730 T wdt_log_kexec_setup
ffffffc00066b810 T wdt_log_save
ffffffc00066b9c0 T wdt_log_ready
ffffffc00066ba88 t wdt_detect_isr
ffffffc00066bad0 t wdt_detect_proc
ffffffc000a09830 t sys_proc_read_kdump_meminfo
ffffffc000a0a830 t diag_panic_event
ffffffc000a0a910 T diag_invoke_reset
ffffffc0016f7ca4 t mmcoops_init
ffffffc0016f7d74 t wdt_log_init
```

However, the published `kernel-source` appears incomplete for the most interesting pieces.

Files present:

```text
kernel-source/drivers/staging/webos/logger/Kconfig
kernel-source/drivers/staging/webos/logger/Makefile
kernel-source/drivers/staging/webos/logger/ll_mmc.h
kernel-source/drivers/staging/webos/logger/wdt_log.h
```

Files expected but not found:

```text
mmcoops.c
wdt_log.c
wdt_detect.c
arch/arm64/kernel/machine_kexec.c
```

This means LG published config, headers, build metadata, and `System.map`, but not all implementation source for these LG-specific pieces.

---

## 15. Disassembly of `machine_kexec()`

Because `vmlinux` was not available, `Image.lg` was analyzed as a flat AArch64 binary using `System.map`.

Target symbols:

```text
ffffffc00009b708 T machine_kexec
ffffffc00066b730 T wdt_log_kexec_setup
ffffffc00066b810 T wdt_log_save
ffffffc00066b9c0 T wdt_log_ready
ffffffc00066bad0 t wdt_detect_proc
ffffffc000a0a830 t diag_panic_event
ffffffc000a0a910 T diag_invoke_reset
ffffffc0016f7ca4 t mmcoops_init
ffffffc0016f7d74 t wdt_log_init
```

Resolved direct calls inside `machine_kexec()`:

```text
ffffffc00009b728: bl 0xffffffc0003e4498 -> lzo1x_1_do_compress+0x440
ffffffc00009b79c: bl 0xffffffc00009b4e0 -> _kexec_image_info+0x0
ffffffc00009b810: bl 0xffffffc0003c9c00 -> sha_transform+0x1020
ffffffc00009b81c: bl 0xffffffc00009ef30 -> __pi___flush_dcache_area+0x0
ffffffc00009b828: bl 0xffffffc00009eec0 -> flush_icache_range+0x0
ffffffc00009b86c: bl 0xffffffc00009ef30 -> __pi___flush_dcache_area+0x0
ffffffc00009b89c: bl 0xffffffc00009ef30 -> __pi___flush_dcache_area+0x0
ffffffc00009b910: bl 0xffffffc00009ef30 -> __pi___flush_dcache_area+0x0
ffffffc00009b944: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009b984: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009b9a0: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009b9bc: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009b9d8: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009b9f4: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009ba14: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009ba30: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009ba48: bl 0xffffffc00041c6c8 -> pinconf_groups_show+0x70
ffffffc00009ba60: bl 0xffffffc000163960 -> printk+0x8c
ffffffc00009ba68: bl 0xffffffc00009fbb0 -> setup_mm_for_reboot+0x0
```

No direct call was seen to:

```text
ffffffc00066b730 wdt_log_kexec_setup
ffffffc00066b810 wdt_log_save
ffffffc00066b9c0 wdt_log_ready
```

This is one of the most important findings.

---

## 16. Main technical conclusion

The runtime behavior and static analysis match:

```text
Normal panic:
  goes through LG panic/diagnosis/mmcoops path
  writes persistent log to /dev/mmcblk0p52

kexec -e:
  does not call wdt_log_kexec_setup()
  does not call wdt_log_save()
  does not populate /dev/mmcblk0p52
  eventually reboots as cpuAbnormal
```

Therefore:

```text
The failed kexec path is not a normal Linux panic/oops.
It likely fails after control has left the running webOS kernel, or in a platform/MICOM/watchdog path that does not preserve a Linux dump.
```

In practical terms:

```text
A new kernel used as a kexec payload cannot fix this by itself,
because the old LG kernel's machine_kexec() path runs first.
```

---

## 17. What this means for booting a custom kernel

Current state:

```text
kexec -l works.
kexec -e causes reboot/cpuAbnormal.
No UART.
No pstore.
No crashkernel.
LG kdump missing required retail components.
mmcoops works for panic but not for kexec.
machine_kexec does not call LG wdtlog setup.
```

Therefore:

```text
Custom kernel via current kexec path is not viable at this stage.
```

This does not prove custom kernel boot is impossible forever. It means this particular path is blocked without one of:

```text
UART access
a bootloader/flash path
a patched currently running LG kernel
a working recovery path
a deeper exploit/bootchain method
```

---

## 18. Flashing a custom kernel

In theory, booting a custom kernel may require flashing or replacing the kernel image used by lxboot.

However, the bootloader strings contain many verification warnings:

```text
secureboot
[pmtRSA]
[RSA]
kernel verification failed
verification failed
fullverify
System Halted
kernel.lzo
kernel decompression failed
```

This strongly suggests that blindly writing a kernel partition is dangerous.

Do **not** flash kernel/rootfs/tvservice partitions without:

```text
UART
full eMMC backup
verified partition map
known boot image format
known signing/hash requirements
known recovery path
```

The TV itself warns:

```text
NEVER EVER OVERWRITE SYSTEM PARTITIONS LIKE KERNEL, ROOTFS, TVSERVICE.
Your TV will be bricked, guaranteed.
```

Given secure boot / verification strings, flashing a modified kernel may fail at bootloader verification unless the image format and signing/hash mechanisms are understood.

---

## 19. Android port implications

Because custom kernel boot via `kexec` is currently blocked, a near-term Android port would likely have to run on top of the existing LG/webOS kernel.

That means the realistic research path is:

```text
webOS bootloader
  -> LG O20 kernel
    -> webOS userspace remains or is partially reused
      -> Android/Linux userspace experiment
```

Important Android kernel features to check:

```text
binder / binderfs
ashmem or memfd compatibility
cgroups
namespaces
tmpfs
devtmpfs
overlayfs
fuse
loop
tun
netfilter
dmabuf / dma-heap / ion
SELinux support or workaround
```

Recommended config check:

```sh
cd ~/disk/kernel/lg-c1-src/o20n/o20n_mt7921/linux-rockhopper_build_o20

grep -E \
'CONFIG_ANDROID|CONFIG_ANDROID_BINDER|CONFIG_ASHMEM|CONFIG_MEMFD|CONFIG_ION|CONFIG_DMABUF|CONFIG_DMA_SHARED|CONFIG_CGROUP|CONFIG_NAMESPACES|CONFIG_PID_NS|CONFIG_NET_NS|CONFIG_USER_NS|CONFIG_SECCOMP|CONFIG_OVERLAY_FS|CONFIG_FUSE|CONFIG_LOOP|CONFIG_TUN|CONFIG_BRIDGE|CONFIG_NETFILTER|CONFIG_SELINUX|CONFIG_TMPFS|CONFIG_DEVTMPFS' \
.config include/config/auto.conf
```

Recommended runtime checks on the TV:

```sh
ls -l /dev/binder /dev/vndbinder /dev/hwbinder 2>/dev/null || true
find /dev /sys -maxdepth 4 2>/dev/null | grep -i binder || true

cat /proc/filesystems | grep -E 'overlay|fuse|tmpfs|squashfs|ext4'

ls -la /proc/self/ns
mount | grep -E 'cgroup|binder|ashmem|tmpfs'
```

Practical conclusion:

```text
Android with a custom kernel:
  blocked for now.

Android userspace on LG/webOS kernel:
  possible research direction.

Full Android UI with graphics/audio/video:
  likely hard due to LG proprietary display/audio/video stack.
```

---

## 20. Recommended repo structure

Suggested repository layout:

```text
LG-C1-kernel/
├── README.md
├── docs/
│   ├── kexec-research.md
│   ├── dump-mmcoops.md
│   ├── lg-opensource-analysis.md
│   ├── partition-map.md
│   ├── android-on-webos-kernel.md
│   └── bootloader-risk.md
├── scripts/
│   ├── parse_partinfo.py
│   ├── scan_lg_image_calls.py
│   ├── resolve_calls.py
│   ├── collect-kexec-baseline.sh
│   ├── collect-post-reboot.sh
│   └── check-android-kernel-features.sh
├── evidence/
│   ├── faultmanager/
│   ├── p52-panic-test/
│   ├── p52-kexec-test/
│   ├── lg-source/
│   └── disasm/
└── userspace/
    ├── linux-chroot/
    └── android-userspace/
```

---

## 21. Key evidence files produced during research

Important generated files/directories:

```text
faultmanager-poweron-reasons.txt
faultmanager-poweron-context.txt
faultmanager-interesting.txt

deeper-dump/dump-partition/partition-start-size.txt
deeper-dump/dump-partition/tiny-parts/p2.bin
deeper-dump/dump-partition/tiny-parts/p3.bin
deeper-dump/dump-partition/real-dump/p52-dump-full.bin

deeper-dump/mmcoops-panic-test/p52-before-panic.bin
deeper-dump/mmcoops-panic-test/p52-after-panic.bin
deeper-dump/mmcoops-panic-test/p52-after-panic-interesting.txt

kexec-vs-dump-test/<run>/before/p52-before-kexec.bin
kexec-vs-dump-test/<run>/after/p52-after-kexec.bin
kexec-vs-dump-test/<run>/after/p52-after-interesting.txt
kexec-vs-dump-test/<run>/after/faultmanager-after-interesting.txt

lg-c1-src/o20n/o20n_mt7921/linux-rockhopper_build_o20/.config
lg-c1-src/o20n/o20n_mt7921/linux-rockhopper_build_o20/System.map
lg-disasm/asm/machine_kexec.S
lg-disasm/machine_kexec-calls-resolved.txt
```

---

## 22. Useful commands

### 22.1 Dump current kernel state

```sh
cat /proc/cmdline
cat /proc/iomem
cat /sys/kernel/kexec_loaded 2>/dev/null
cat /sys/kernel/kexec_crash_loaded 2>/dev/null
cat /sys/kernel/kexec_crash_size 2>/dev/null
```

### 22.2 Confirm dump partition

```sh
dd if=/dev/mmcblk0p52 of=p52-dump.bin bs=1M count=10
strings p52-dump.bin | head
```

### 22.3 Trigger controlled panic

```sh
echo 1 > /proc/sys/kernel/sysrq
sync
echo c > /proc/sysrq-trigger
```

After reboot:

```sh
dd if=/dev/mmcblk0p52 of=p52-after-panic.bin bs=1M count=10

strings p52-after-panic.bin | grep -i -E \
'panic|oops|sysrq|crash|cpu|watchdog|wdt|mmcoops|wdtlog|dump|Call trace|Kernel panic|Unable to handle'
```

### 22.4 kexec load/execute

```sh
cd /tmp/usb/sda/sda1/lgc1-kexec

CMDLINE='console=ttyAMA0,115200n81 earlycon=pl011,mmio32,0xfe000000 ignore_loglevel loglevel=8 rdinit=/init root=/dev/ram0 rw maxcpus=1 nr_cpus=1 reset_devices irqpoll'

./ld-linux-aarch64.so.1 --library-path . ./kexec -l ./Image \
  --dtb=./lgc1-running.dtb \
  --initrd=./initramfs.cpio.gz \
  --append="$CMDLINE" \
  --debug

cat /sys/kernel/kexec_loaded

sync
./ld-linux-aarch64.so.1 --library-path . ./kexec -e
```

### 22.5 Compare dump before/after kexec

```sh
dd if=/dev/mmcblk0p52 of=p52-before-kexec.bin bs=1M count=10

# run kexec -e, allow TV to reboot normally

dd if=/dev/mmcblk0p52 of=p52-after-kexec.bin bs=1M count=10

cmp -s p52-before-kexec.bin p52-after-kexec.bin \
  && echo "P52_UNCHANGED" \
  || echo "P52_CHANGED"
```

### 22.6 Search LG source for important symbols

```sh
cd ~/disk/kernel/lg-c1-src/o20n/o20n_mt7921/linux-rockhopper_build_o20

grep -E \
'CONFIG_KEXEC|CONFIG_CRASH_DUMP|CONFIG_PSTORE|CONFIG_MMCOOPS|CONFIG_DPM_WATCHDOG|CONFIG_ARM_SP805_WATCHDOG|CONFIG_WATCHDOG_NOWAYOUT|CONFIG_PANIC' \
.config include/config/auto.conf

grep -E \
' machine_kexec$| wdt_log_kexec_setup$| wdt_log_save$| wdt_log_ready$| wdt_detect_proc$| diag_panic_event$| diag_invoke_reset$| mmcoops_init$| wdt_log_init$' \
System.map
```

---

## 23. Current final conclusion

The strongest conclusion from the whole project so far:

```text
The LG C1/O20 kernel has working LG mmcoops/dump infrastructure,
and /dev/mmcblk0p52 is the real persistent dump partition.

A controlled Linux panic writes a useful panic log to p52.

However, failed kexec -e attempts leave p52 unchanged and the next boot is
classified by webOS/MICOM as cpuAbnormal.

Static analysis of machine_kexec() shows no direct call to wdt_log_kexec_setup(),
wdt_log_save(), or wdt_log_ready().

Therefore the kexec failure path is not captured by LG's Linux panic/dump path.
The failure likely occurs after control leaves the running kernel, or through
a platform/MICOM watchdog/reset path outside normal Linux panic handling.
```

Practical implication:

```text
A custom kernel via the current kexec path is not viable right now.

A near-term Android/Linux port should assume the existing LG/webOS kernel
continues to run, unless a separate bootloader/flash/UART recovery path is
developed.
```

---

## 24. Next steps

Recommended next steps:

```text
1. Add UART.
   This is the single most valuable debugging upgrade.

2. Preserve all evidence and scripts in the repo.

3. Investigate Android userspace feasibility on the LG kernel:
   binder, ashmem/memfd, cgroups, namespaces, graphics, input, audio.

4. Analyze lxboot/kernel verification before considering any flash attempt.

5. Do not flash kernel/rootfs/tvservice partitions without recovery.
```

Possible advanced direction:

```text
Disassemble more of Image.lg:
- wdt_log_kexec_setup
- wdt_log_save
- diag_panic_event
- diag_invoke_reset
- wdt_detect_proc

Search for indirect callers or init registration.
```

But this will not by itself make `kexec -e` boot a custom kernel; it will only explain LG's proprietary logging path more deeply.

---

## 25. Short version

```text
Can we kexec a custom kernel today?
  No, not reliably.

Did normal kexec load?
  Yes.

Did kexec execute successfully?
  No. It reboots as cpuAbnormal.

Does LG dump work?
  Yes, for normal Linux panics.

Does kexec failure write LG dump?
  No.

Does machine_kexec call LG wdtlog setup?
  No direct call observed.

Is the correct LG source package found?
  Yes: OLED65C17LB / webOS 6.0 ON 1.0.

Is the published source complete?
  Not for the critical LG-specific files.

Best next path?
  Android/Linux userspace on webOS kernel, or UART + bootloader/flash research.
```

