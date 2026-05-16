LG C1 / webOS kexec research notes

This document summarizes the current state of the `LG-C1-kernel` research effort: attempting to boot a custom kernel on an LG C1 webOS TV using `kexec`, identifying LG's crash/dump mechanisms, and assessing whether an Android port would need to run on top of the stock webOS kernel.
> **Safety warning**
>
> Do **not** overwrite internal eMMC partitions such as kernel, rootfs, tvservice, bootloader, partinfo, or system partitions. The commands documented here are intended to be read-only unless explicitly marked otherwise. A wrong `dd of=/dev/mmcblk0pXX` can brick the TV.
---
Device tested
Observed device and firmware state:
```text
Model: OLED65C17LB
SoC:   LG O20B0
webOS: 4.05.48
Kernel: 4.4.84-229.1.kavir.2
Arch:  AArch64
```
Relevant stock kernel command line:
```text
root=/dev/mmcblk0p27 ro rootfstype=squashfs vmalloc=1216M \
hma=856M@0x047800000;360M@0x0a9800000 \
lg1k.use_vmap=1 hma.use_vmap=1 use_vmap=1 cma_on \
console=ttyAMA0,115200n81 chip=O20B0 sver=4.05.48 bver=4.05.48 \
quiet loglevel=0 snapshot resume=/dev/mmcblk0p51 devtmpfs.mount=1 rootwait \
emmc_size=0x1d2000000 sbkey=0x7d0e0000 portProtection \
mmcoops=dump wdtlog=dump@1M \
modelName=OLED65C17LB debugMode=5 cmdEnd
```
Important parameters:
```text
console=ttyAMA0,115200n81
mmcoops=dump
wdtlog=dump@1M
snapshot resume=/dev/mmcblk0p51
```
No `crashkernel=` parameter is present.
---
Test files used
The working USB directory contained:
```text
Image
initramfs.cpio.gz
lgc1-running.dtb
kexec
ld-linux-aarch64.so.1
libc.so.6
libz.so.1
wdctl-lgc1
cmdline-webos.txt
reserved-memory.txt
iomem.txt
partitions.txt
meminfo.txt
cpuinfo.txt
uname.txt
dmesg-before-kexec.txt
kexec-debug-load*.txt
kexec-probe.log
pause-boot-probe.log
```
The local `kexec` binary and its libraries were supplied from USB. The retail firmware did not provide `/usr/sbin/kexec`.
---
Initial kexec status
Normal kexec load
`kexec -l` succeeds with the supplied image, DTB, and initramfs.
Representative command:
```sh
cd /tmp/usb/sda/sda1/lgc1-kexec

CMDLINE='console=ttyAMA0,115200n81 earlycon=pl011,mmio32,0xfe000000 ignore_loglevel loglevel=8 rdinit=/init root=/dev/ram0 rw maxcpus=1 nr_cpus=1 reset_devices irqpoll'

./ld-linux-aarch64.so.1 --library-path . ./kexec -l ./Image \
  --dtb=./lgc1-running.dtb \
  --initrd=./initramfs.cpio.gz \
  --append="$CMDLINE" \
  --debug

cat /sys/kernel/kexec_loaded
```
After loading, `/sys/kernel/kexec_loaded` reports `1`.
Executing kexec
Executing the loaded kernel:
```sh
sync
./ld-linux-aarch64.so.1 --library-path . ./kexec -e
```
Observed behavior:
```text
kexec -e starts the transition.
The current SSH session dies.
The TV reboots after roughly 10-15 seconds.
After reboot, webOS starts normally again.
```
No output is visible through SSH after `kexec -e`.
---
kexec -p / kdump status
`kexec -p` is not usable on this boot because no standard crashkernel memory is reserved.
Observed state:
```sh
tr ' ' '\n' < /proc/cmdline | grep -i crash || true
cat /sys/kernel/kexec_crash_loaded
cat /sys/kernel/kexec_crash_size
```
Result:
```text
no crashkernel parameter
kexec_crash_loaded = 0
kexec_crash_size   = 0
```
`/proc/iomem` showed System RAM regions but no standard crashkernel reservation.
Kernel config findings:
```text
CONFIG_KEXEC=y
CONFIG_CRASH_DUMP is not set
CONFIG_PSTORE is not set
CONFIG_DPM_WATCHDOG=y
CONFIG_ARM_SP805_WATCHDOG=y
CONFIG_WATCHDOG_NOWAYOUT is not set
```
There is a custom LG memory report:
```text
/proc/kdump_mem = 124M@0x56c00000
```
However, this is not recognized by upstream `kexec -p` as standard crashkernel memory.
Conclusion:
```text
The stock retail boot does not provide a usable upstream kdump path.
```
---
LG kdump service investigation
`kdump.service` exists:
```ini
[Service]
Type=simple
EnvironmentFile=-/var/systemd/system/env/kdump.env
ExecStart=/lib/systemd/system/scripts/kdump.sh
RemainAfterExit=yes
```
The service reports `active (exited)` because the script exits successfully. It does not mean a crash kernel is armed.
The script checks for `crashkernel` in `/proc/cmdline` and then attempts something equivalent to:
```sh
/usr/sbin/kexec -i -p /boot/kdImage-1.0.14-162 \
  --command-line="root=... init=/sbin/init.kdump maxcpus=1 reset_devices mem=$(cat /proc/kdump_mem) wdtlog=dump@1M secondkernel"
```
On this retail firmware, the required files do not exist:
```text
/boot/kdImage-1.0.14-162  -> missing
/sbin/init.kdump          -> missing
/usr/sbin/kexec           -> missing
```
A wider search did not find alternate `kdImage`, `init.kdump`, or `/usr/sbin/kexec` pieces.
Conclusion:
```text
LG kdump infrastructure is partially present, but non-functional on this retail firmware.
```
---
UART / framebuffer / pstore observability
UART
The kernel console is configured for:
```text
console=ttyAMA0,115200n81
```
However, no physical UART was available during these tests. Therefore, anything printed after `kexec -e` would not be visible over SSH.
Framebuffer
Framebuffer devices exist:
```text
/dev/fb0
/dev/fb1
/dev/fb2
/dev/fb3
```
`/proc/fb` showed:
```text
0 osd0_fb
1 osd1_fb
2 osd2_fb
3 crsr_fb
```
Observed framebuffer characteristics included:
```text
fb0 osd0_fb: 1920x2160, 32 bpp, stride 7680
fb1 osd1_fb: 512x4320, 32 bpp, stride 2048
fb2 osd2_fb: 128x4, 32 bpp
fb3 crsr_fb: 256x512, 32 bpp
```
Writes to `/dev/fb0` and `/dev/fb1` did not produce visible on-screen output. Therefore framebuffer could not be used as a reliable post-kexec debug channel.
pstore
`CONFIG_PSTORE` is not set. No usable `/sys/fs/pstore` logs were available.
Conclusion:
```text
No reliable post-kexec output channel is currently available without physical UART.
```
---
Watchdog / reset investigation
Relevant watchdog devices and symbols were found:
```text
/sys/bus/amba/devices/fd200000.watchdog
/sys/bus/amba/drivers/sp805-wdt
/sys/devices/platform/wdt_detect
/dev/watchdog
/dev/watchdog0
```
Relevant runtime parameters:
```text
dpm_watchdog_timeout = 5
dpm_watchdog_timeout_userresume = 20
stmmac watchdog = 5000
```
Tests performed:
Changed DPM watchdog timeout from `5` to `120`.
Changed DPM userresume timeout from `20` to `120`.
Attempted `/dev/watchdog` magic close with `V`.
Re-ran `kexec -e`.
Result:
```text
The TV rebooted after the same ~10-15 second interval.
```
Conclusion:
```text
The reset is not controlled by the sysfs DPM timeout parameter or the regular /dev/watchdog path tested here.
```
The suspected reset path is platform/MICOM/firmware-specific or occurs after the running Linux kernel has already left control.
---
faultmanager evidence: PowerOnReason=cpuAbnormal
webOS faultmanager logs were found under:
```text
/var/spool/faultmanager/crash/
  crash_fault_logs.tar.gz
  crash_fault_logs.0.tar.gz
  crash_fault_logs.1.tar.gz
  crash_fault_logs.2.tar.gz
```
These logs record different power-on reasons, including normal reasons and the abnormal reset reason.
Observed after failed `kexec -e` attempts:
```text
PowerOnReason=cpuAbnormal
reason":"cpuAbnormal"
power on reason string:cpuAbnormal, enum:14
```
The logs also show other normal boot reasons:
```text
remoteKey
micomPower
```
This is important because `cpuAbnormal` is not simply a generic/default value. webOS/MICOM distinguishes it from normal boot causes.
Conclusion:
```text
Failed kexec-e attempts reboot the platform and are recorded by webOS/MICOM as cpuAbnormal.
```
---
Kernel symbols and LG-specific dump path
`/proc/kallsyms` showed standard and LG-specific symbols relevant to this work.
Standard kexec/panic symbols:
```text
machine_kexec_prepare
machine_kexec
machine_kexec_cleanup
machine_crash_shutdown
kernel_kexec
crash_kexec
sys_kexec_load
kexec_image
kexec_crash_image
```
LG-specific watchdog/dump/MICOM symbols:
```text
wdt_log_kexec_setup
wdt_log_save
wdt_log_ready
wdt_detect_isr
wdt_detect_proc
wdt_detect_probe
mmcoops_init
wdt_log_init
wdt_detect_init
sys_proc_read_kdump_meminfo
diag_panic_event
diag_invoke_reset
get_micom_disable
set_micom_disable
o20_ucom_FuncChipReset
```
The kernel image also contains strings indicating LG's custom dump mechanism:
```text
mmcoops:Oops/panic log will be stored at %s partition (offset: 0x%llx)
wdtlog:pmlog/legacy log will be stored at %s partition (offset: 0x%llx)
wdtlog: Start second kernel
wdtlog: file copy completed: %u bytes
mmcoops=
wdtlog=
```
This matches the stock command line:
```text
mmcoops=dump
wdtlog=dump@1M
```
Conclusion:
```text
LG has a proprietary dump/logging path using a logical partition named dump. It is separate from upstream pstore and crash_dump.
```
---
Partition map and identifying the dump partition
Linux exposes 56 eMMC partitions:
```text
/dev/mmcblk0p1 ... /dev/mmcblk0p56
```
`blkid` only identifies some partitions as `squashfs` or `ext4`; many are raw/unknown.
Important partition observations:
```text
p27 = active rootfs squashfs
p51 = snapshot/resume partition, from cmdline resume=/dev/mmcblk0p51
p54/p55/p56 = ext4 data-style partitions
p38-p42 = likely backup/update mirror of p27-p31 layout
```
p4
`p4` contains `lxboot` / LG bootloader code and strings, including:
```text
PARTINFO : Loaded O.K
showpart
display partinfo
emmc dump offset|partition size
mmcoops=dump
wdtlog=dump@1M
kernel.lz4
rootfs.squashfs
```
This is bootloader/parser code, not the dump log partition.
p21
`p21` starts with:
```text
LZ4P
ARMd
```
and contains many kernel strings including `machine_kexec_prepare`, `wdtlog`, `MMCOOPS`, etc. It appears to be a kernel or kernel-like firmware image.
p2 and p3
`p2` and `p3` contain the actual LG `partinfo` table and its backup. They include strings such as:
```text
h13_emmc
secureboot
partinfo
PART.INFO
mapbak
boot
kernel
rootfs
dbboot
dump
hist
reserved
```
A parser was written to match partinfo offsets/sizes against Linux partition start/size information.
Result:
```text
FOUND_DUMP_NAME=dump
FOUND_DUMP_OFFSET=0xe7980000
FOUND_DUMP_SIZE=0xa00000
FOUND_DUMPDEV=/dev/mmcblk0p52
```
Therefore:
```text
dump = /dev/mmcblk0p52
offset = 0xe7980000
size   = 0x00a00000 = 10 MiB
```
---
Proving mmcoops works
Before a controlled panic, `/dev/mmcblk0p52` was all zeroes:
```text
p52 is all zero
```
A controlled Linux panic was triggered:
```sh
echo 1 > /proc/sys/kernel/sysrq
sync
echo c > /proc/sysrq-trigger
```
After reboot, `/dev/mmcblk0p52` contained a persistent panic/oops log.
Observed strings from `p52-after-panic.bin`:
```text
sysrq: SysRq : Trigger a crash
Unable to handle kernel NULL pointer dereference
Internal error: Oops: 96000046 [#1] PREEMPT SMP
CPU: 3 PID: ... Comm: sh
PC is at sysrq_handle_crash+0x14/0x20
Call trace:
Kernel panic - not syncing: Fatal exception
CPU0: stopping
CPU1: stopping
CPU2: stopping
--- diagnosis panic event ----
```
Conclusion:
```text
LG mmcoops/dump works for normal Linux panics.
```
This is an important control test: the dump partition is real and functional.
---
kexec-e vs dump partition test
A clean before/after comparison was performed around a failed `kexec -e` attempt.
Procedure:
Copy `/dev/mmcblk0p52` before `kexec -e`.
Run `kexec -e`.
Let the TV reboot normally. Do not unplug it.
SSH back in.
Copy `/dev/mmcblk0p52` again.
Compare the two images.
Result:
```text
P52_UNCHANGED
```
`cmp` showed no byte differences, and no relevant strings appeared in the post-kexec dump image.
Observed after reboot:
```text
kexec_loaded       = 0
kexec_crash_loaded = 0
kexec_crash_size   = 0
```
Faultmanager still recorded the boot reason as:
```text
cpuAbnormal
```
Conclusion:
```text
A normal Linux panic writes to /dev/mmcblk0p52.
A failed kexec-e does not write to /dev/mmcblk0p52.
```
Therefore, the `kexec -e` failure does not appear to go through the normal Linux panic/oops/mmcoops path.
Most likely explanations:
```text
1. Control leaves the running webOS kernel during machine_kexec.
2. The destination kernel either never starts or does not start far enough to log anything.
3. A platform/MICOM/firmware watchdog/reset path reboots the system.
4. The reset is classified by webOS/MICOM as cpuAbnormal.
```
What this test cannot determine without UART:
```text
A. Whether machine_kexec jumps to the destination image at all.
B. Whether the destination kernel executes any instructions.
C. Whether it reaches early decompression/start_kernel.
D. Whether a platform watchdog kills it after entry.
```
---
Summary of findings
Confirmed
```text
CONFIG_KEXEC=y.
kexec -l succeeds.
kexec -e causes a reboot after roughly 10-15 seconds.
After reboot, webOS/faultmanager reports PowerOnReason=cpuAbnormal.
LG dump partition is /dev/mmcblk0p52.
/dev/mmcblk0p52 is used by LG mmcoops/dump.
A controlled Linux panic writes a persistent panic log to p52.
kexec -e does not modify p52.
LG kdump service exists but is non-functional in this retail firmware.
No upstream pstore support is compiled.
No upstream crash dump support is compiled.
No standard crashkernel memory is reserved.
Framebuffer devices exist but are not useful as visible output.
DPM watchdog timeout changes do not alter kexec-e reboot behavior.
/dev/watchdog magic close does not alter kexec-e reboot behavior.
```
Not confirmed
```text
Whether the destination kernel receives control.
Whether the destination kernel starts but dies early.
Whether the platform reset is caused by MICOM, SP805, firmware, secure monitor, or hardware state left by machine_kexec.
Whether a patched stock kernel could make kexec work.
```
---
Current interpretation
The current `kexec` path is not usable for reliably booting a custom kernel on this LG C1 firmware.
The most defensible statement is:
```text
kexec_load works, but kexec_execute results in a platform-level cpuAbnormal reboot without entering the normal Linux panic/oops/mmcoops path.
```
This suggests the reset happens either:
```text
- after the stock webOS kernel has already left control, or
- through a low-level platform/MICOM/firmware path that Linux cannot log, or
- before/inside the destination kernel so early that no output or persistent dump is produced.
```
Without UART, there is no direct visibility into the transition.
---
Android port implications
The practical consequence is:
```text
Do not currently rely on kexec to boot an Android kernel.
```
A native Android port using a custom kernel would require one of:
```text
- bootloader control,
- working kexec,
- physical UART to debug the failed transition,
- LG O20 kernel sources/patches sufficient to fix machine_kexec/watchdog/reset handling,
- or another bootchain/exploit path.
```
The more realistic near-term path is Android or Linux userspace on top of the stock webOS kernel.
Possible stack:
```text
LG stock webOS kernel
  +
webOS userspace still booting normally
  +
chroot/container/proot Linux userspace
  +
experimental Android userspace components
```
Important Android kernel features to check next:
```text
CONFIG_ANDROID_BINDER_IPC
CONFIG_ANDROID_BINDERFS
CONFIG_ASHMEM or memfd compatibility
CONFIG_CGROUPS
CONFIG_NAMESPACES
CONFIG_SECCOMP
CONFIG_OVERLAY_FS
CONFIG_EXT4_FS
CONFIG_FUSE_FS
CONFIG_TMPFS
CONFIG_VETH
CONFIG_TUN
CONFIG_BRIDGE
CONFIG_NETFILTER
CONFIG_SECURITY_SELINUX
CONFIG_DMA_SHARED_BUFFER
ION / DMA-BUF / graphics memory support
```
Suggested command:
```sh
zcat /proc/config.gz 2>/dev/null | grep -i -E \
'ANDROID|BINDER|ASHMEM|MEMFD|ION|DMABUF|DMA_SHARED|CGROUP|NAMESPACE|SECCOMP|OVERLAY|SQUASHFS|EXT4|LOOP|VETH|TUN|BRIDGE|NETFILTER|SELINUX|DEVTMPFS|TMPFS|FUSE'
```
If Binder is missing, a real Android userspace becomes much harder. If Binder exists, Android userspace on top of the LG kernel becomes more plausible.
---
Useful commands kept for reference
Load and execute kexec
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
Inspect kexec state
```sh
cat /sys/kernel/kexec_loaded 2>/dev/null || true
cat /sys/kernel/kexec_crash_loaded 2>/dev/null || true
cat /sys/kernel/kexec_crash_size 2>/dev/null || true
```
Inspect crashkernel presence
```sh
tr ' ' '\n' < /proc/cmdline | grep -i crash || true
grep -i -E 'crash|reserved|System RAM' /proc/iomem
```
Copy dump partition
```sh
cd /tmp/usb/sda/sda1/lgc1-kexec
mkdir -p dump-copy

dd if=/dev/mmcblk0p52 of=dump-copy/p52-dump.bin bs=1M count=10
md5sum dump-copy/p52-dump.bin
strings dump-copy/p52-dump.bin | head -200
```
Controlled panic test
```sh
echo 1 > /proc/sys/kernel/sysrq
sync
echo c > /proc/sysrq-trigger
```
After reboot:
```sh
dd if=/dev/mmcblk0p52 of=p52-after-panic.bin bs=1M count=10
strings p52-after-panic.bin | grep -i -E 'panic|oops|sysrq|Call trace|Kernel panic|Unable to handle|diagnosis'
```
Compare dump before/after kexec
```sh
cd /tmp/usb/sda/sda1/lgc1-kexec
mkdir -p kexec-vs-dump-test

RUN="run-$(date +%Y%m%d-%H%M%S)"
echo "$RUN" > kexec-vs-dump-test/current-run.txt
mkdir -p "kexec-vs-dump-test/$RUN/before" "kexec-vs-dump-test/$RUN/after"

dd if=/dev/mmcblk0p52 of="kexec-vs-dump-test/$RUN/before/p52-before-kexec.bin" bs=1M count=10
md5sum "kexec-vs-dump-test/$RUN/before/p52-before-kexec.bin" > "kexec-vs-dump-test/$RUN/before/md5-before.txt"
sync

# Run kexec-e here, then SSH back in after reboot.

RUN="$(cat kexec-vs-dump-test/current-run.txt)"
dd if=/dev/mmcblk0p52 of="kexec-vs-dump-test/$RUN/after/p52-after-kexec.bin" bs=1M count=10
md5sum "kexec-vs-dump-test/$RUN/after/p52-after-kexec.bin" > "kexec-vs-dump-test/$RUN/after/md5-after.txt"

if cmp -s "kexec-vs-dump-test/$RUN/before/p52-before-kexec.bin" \
          "kexec-vs-dump-test/$RUN/after/p52-after-kexec.bin"; then
  echo "P52_UNCHANGED"
else
  echo "P52_CHANGED"
fi | tee "kexec-vs-dump-test/$RUN/after/p52-change-result.txt"
```
---
Recommended next steps
Highest value
```text
1. Obtain physical UART.
2. Capture serial output during kexec-e.
3. Determine whether execution reaches the destination kernel.
4. Acquire LG GPL/O20 kernel sources matching this firmware.
5. Inspect machine_kexec.c, wdt_log_kexec_setup, wdt_detect, mmcoops, MICOM reset logic.
```
Android/userspace path
```text
1. Check Android kernel feature support in /proc/config.gz.
2. Test Binder availability.
3. Test cgroups/namespaces/seccomp/mount behavior.
4. Start with Linux chroot/container on webOS.
5. Attempt minimal Android userspace only after confirming Binder and graphics/input/audio feasibility.
```
Do not spend more time on, unless new visibility is available
```text
- Repeating blind kexec-e attempts.
- Instrumenting purgatory without UART.
- Writing to internal partitions.
- Reading random /proc or /sys driver nodes that may block or hang the TV.
```
---
Final conclusion
Current status:
```text
Custom kernel via current kexec path: blocked / not reliable.
Native Android kernel boot: not currently viable through this method.
Android/Linux userspace on top of the stock webOS kernel: most realistic next research direction.
```
The key evidence is:
```text
panic -> /dev/mmcblk0p52 receives persistent mmcoops log
kexec-e -> reboot + cpuAbnormal, but /dev/mmcblk0p52 unchanged
```
Therefore, failed `kexec -e` is not behaving like a Linux panic. It is most likely a low-level platform reset during or after the kexec transition.
