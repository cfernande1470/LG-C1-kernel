# LG-C1-kernel

Documentación de los experimentos de arranque alternativo en una LG OLED C1 / webOS usando `kexec`.

> **Estado actual:** `kexec -l` funciona y carga correctamente el kernel LG original extraído, el DTB y el initramfs.  
> **Bloqueo actual:** `kexec -e` no consigue arrancar limpiamente un segundo kernel. Incluso al hacer kexec hacia el propio webOS original, la TV se congela y reinicia a los ~10 segundos con `PowerOnReason=cpuAbnormal`.

---

## ⚠️ Aviso importante

No sobrescribir particiones internas de la TV.

En particular, **no escribir nunca** en:

- kernel
- rootfs
- tvservice
- particiones `mmcblk0*`
- bootloader
- particiones de actualización A/B

Todo el trabajo descrito aquí se ha hecho leyendo desde la eMMC y escribiendo únicamente en:

- USB stick montado en `/tmp/usb/sda/sda1`
- `/media` / `/media/internal` para pruebas de marcador
- repo de desarrollo en la NanoPi

La TV muestra este aviso al entrar por SSH:

```text
NEVER EVER OVERWRITE SYSTEM PARTITIONS LIKE KERNEL, ROOTFS, TVSERVICE.
Your TV will be bricked, guaranteed!
```

---

## Hardware / sistema probado

TV:

```text
Modelo: LG OLED C1
modelName=OLED65C17LB
chip=O20B0
webOS TV 6.x
kernel=4.4.84-229.1.kavir.2
arch=aarch64
```

Kernel en ejecución:

```sh
uname -a
```

Resultado observado:

```text
Linux LGwebOSTV 4.4.84-229.1.kavir.2 #1 SMP PREEMPT Mon Jan 17 08:08:42 UTC 2022 aarch64 GNU/Linux
```

Cmdline original observada:

```text
root=/dev/mmcblk0p27 ro rootfstype=squashfs vmalloc=1216M hma=856M@0x047800000;360M@0x0a9800000 lg1k.use_vmap=1 hma.use_vmap=1 use_vmap=1 cma_on ethaddr=... rev=1 console=ttyAMA0,115200n81 chip=O20B0 sver=4.05.48 bver=4.05.48 quiet loglevel=0 snapshot resume=/dev/mmcblk0p51 devtmpfs.mount=1 rootwait emmc_size=0x1d2000000 ... modelName=OLED65C17LB ...
```

---

## Objetivo inicial

Arrancar Android o Linux alternativo sin tocar particiones internas.

Diseño deseado:

```text
webOS arranca lo mínimo
↓
hook Homebrew / SSH manual
↓
kexec carga kernel LG original + DTB + initramfs desde USB
↓
webOS queda reemplazado
↓
initramfs monta Android/Linux desde /media/internal
```

Diseño de almacenamiento previsto:

```text
USB stick:
  /lgc1-kexec/
    kexec-run
    kexec
    ld-linux-aarch64.so.1
    libc.so.6
    libz.so.1
    Image
    initramfs.cpio.gz
    lgc1-running.dtb
    wdctl-lgc1

Interno:
  /media/internal/android-root/
```

---

## Resultado resumido

### Funciona

- Acceso SSH root.
- USB montado automáticamente.
- DTB extraído desde `/sys/firmware/fdt`.
- Kernel LG original localizado en `/dev/mmcblk0p21`.
- Kernel LG desempaquetado desde formato `LZ4P`.
- `Image` resultante reconocido como `Linux kernel ARM64 boot executable Image`.
- `kexec-tools` funciona en la TV usando wrapper con linker/librerías desde USB.
- `kexec -l` devuelve `0`.
- Carga en memoria alta funciona con `--mem-min=0x80000000 --mem-max=0xa9000000`.
- Se puede desactivar el watchdog Linux normal con `WDIOC_SETOPTIONS / WDIOS_DISABLECARD`.

### No funciona todavía

- `kexec -e` no arranca limpiamente el segundo kernel.
- El initramfs no llega a dejar marcadores.
- Incluso kexec hacia el propio webOS original falla.
- La TV congela imagen y reinicia a los ~10 segundos.
- Logs posteriores muestran `PowerOnReason=cpuAbnormal`.
- El reset persiste aunque `/dev/watchdog` y `/dev/watchdog0` aceptan `WDIOS_DISABLECARD`.

Conclusión actual:

```text
El problema no está en Android, initramfs ni /media/internal.
El problema está en la transición kexec / machine_kexec / firmware / DPM watchdog / plataforma LG O20.
```

---

## Layout de particiones relevante

USB:

```text
/dev/sda1 -> /tmp/usb/sda/sda1
LABEL="WEBOSBOOT"
TYPE="vfat"/tfat
```

Root webOS activo:

```text
/dev/mmcblk0p27
TYPE=squashfs
```

Datos internos:

```text
/dev/mmcblk0p56
TYPE=ext4
Montado por webOS bajo /media
/media/internal existe dentro de esa partición
```

Kernel LG extraído:

```text
/dev/mmcblk0p21
24 MiB
Cabecera: LZ4P
Contiene ARMd y strings 4.4.84-229.1.kavir.2
```

Snapshot/resume:

```text
/dev/mmcblk0p51
150 MiB
cmdline original: snapshot resume=/dev/mmcblk0p51
Contiene strings Linux / LGwebOSTV / 4.4.84
```

Metadatos boot:

```text
/dev/mmcblk0p4
Contiene strings:
  kernel
  kernel.lz4
  bootargs
  decomp_kernel
  setup_kernel_params
```

---

## Preparar USB en la TV

Ruta usada:

```sh
USB=/tmp/usb/sda/sda1
BOOT="$USB/lgc1-kexec"
mkdir -p "$BOOT"
```

Extraer DTB real:

```sh
cp /sys/firmware/fdt "$BOOT/lgc1-running.dtb"
```

Guardar datos diagnósticos:

```sh
cat /proc/cmdline > "$BOOT/cmdline-webos.txt"
uname -a > "$BOOT/uname.txt"
cat /proc/cpuinfo > "$BOOT/cpuinfo.txt"
cat /proc/meminfo > "$BOOT/meminfo.txt"
cat /proc/partitions > "$BOOT/partitions.txt"
dmesg > "$BOOT/dmesg-before-kexec.txt"
sync
```

---

## Localizar y extraer el kernel original

Escaneo básico de particiones:

```sh
USB=/tmp/usb/sda/sda1
BOOT="$USB/lgc1-kexec/extracted"
mkdir -p "$BOOT"

for p in 4 21 51 2 3; do
  echo "=== copying /dev/mmcblk0p$p ==="
  dd if=/dev/mmcblk0p$p of="$BOOT/mmcblk0p$p.bin" bs=1M
  sync
  ls -lh "$BOOT/mmcblk0p$p.bin"
done
```

Resultado clave:

```text
mmcblk0p21.bin:
  24 MiB
  empieza por LZ4P
  contiene ARMd
  contiene Linux version 4.4.84-229.1.kavir.2
```

---

## Desempaquetar `LZ4P` en NanoPi

La NanoPi usada:

```text
pi@192.168.2.200
```

Directorio de trabajo:

```sh
mkdir -p ~/disk/LG-C1-kernel
cd ~/disk/LG-C1-kernel
```

Script `scripts/unpack_lg_lz4p.py`:

```python
#!/usr/bin/env python3
import struct
import sys
from pathlib import Path

try:
    import lz4.block
except Exception as e:
    print("ERROR: falta python3-lz4 o pip install lz4:", e)
    sys.exit(1)

if len(sys.argv) != 3:
    print(f"Uso: {sys.argv[0]} input.LZ4P output.bin")
    sys.exit(1)

inp = Path(sys.argv[1])
outp = Path(sys.argv[2])
data = inp.read_bytes()

if data[:4] != b"LZ4P":
    print("No empieza por LZ4P")
    sys.exit(1)

uncomp_size, comp_size, chunk_size, block_count = struct.unpack_from("<IIII", data, 4)

print(f"uncompressed size: 0x{uncomp_size:x} ({uncomp_size})")
print(f"compressed size-ish: 0x{comp_size:x} ({comp_size})")
print(f"chunk size: 0x{chunk_size:x} ({chunk_size})")
print(f"block count: {block_count}")

sizes_off = 0x20
sizes = list(struct.unpack_from("<" + "I" * block_count, data, sizes_off))
table_end = sizes_off + 4 * block_count

candidates = [
    table_end,
    (table_end + 0x0f) & ~0x0f,
    (table_end + 0xff) & ~0xff,
    (table_end + 0x1ff) & ~0x1ff,
    (table_end + 0x3ff) & ~0x3ff,
    0x400,
    0x800,
    0x1000,
]

last_error = None

for data_off in candidates:
    print(f"\n[*] Probando data offset 0x{data_off:x}")
    pos = data_off
    chunks = []
    ok = True

    for i, sz in enumerate(sizes):
        if sz == 0:
            chunks.append(b"\x00" * chunk_size)
            continue

        block = data[pos:pos + sz]
        pos += sz

        try:
            dec = lz4.block.decompress(block, uncompressed_size=chunk_size)
        except Exception as e1:
            try:
                dec = lz4.block.decompress(block)
            except Exception as e2:
                last_error = (i, sz, e1, e2)
                print(f"    fallo bloque {i}, size={sz}: {e1} / {e2}")
                ok = False
                break

        chunks.append(dec)

    if not ok:
        continue

    out = b"".join(chunks)[:uncomp_size]
    outp.write_bytes(out)

    print(f"[+] OK: escrito {outp} ({len(out)} bytes)")
    print("[+] Buscando ARM64 magic ARMd...")
    idx = out.find(b"ARMd")
    print(f"    ARMd offset: {idx if idx >= 0 else 'not found'}")
    idx2 = out.find(b"Linux version")
    print(f"    Linux version offset: {idx2 if idx2 >= 0 else 'not found'}")
    sys.exit(0)

print("\nNo se pudo desempaquetar con offsets probados.")
print("Último error:", last_error)
sys.exit(2)
```

Uso:

```sh
python3 scripts/unpack_lg_lz4p.py mmcblk0p21.bin p21-unpacked.bin
```

Resultado observado:

```text
uncompressed size: 0x1da3600 (31077888)
compressed size-ish: 0xf13c77 (15809655)
chunk size: 0x40000 (262144)
block count: 119

[*] Probando data offset 0x1fc
[+] OK: escrito p21-unpacked.bin (31077888 bytes)
ARMd offset: 56
Linux version offset: 14024832
```

Comprobación:

```sh
file p21-unpacked.bin
strings p21-unpacked.bin | grep -E -m20 'Linux version|4\.4\.84|kavir|ARMd|bootargs|LGwebOSTV'
```

Resultado:

```text
p21-unpacked.bin: Linux kernel ARM64 boot executable Image, little-endian, 4K pages
Linux version 4.4.84-229.1.kavir.2 ...
```

Como `ARMd` estaba en offset `56`, el `Image` empieza en `0`:

```sh
cp p21-unpacked.bin Image.lg
scp Image.lg root@192.168.2.121:/tmp/usb/sda/sda1/lgc1-kexec/Image
```

---

## kexec-tools en la TV

La TV no tenía `kexec` instalado.

Se copió `kexec` aarch64 desde la NanoPi, junto con sus librerías:

```text
kexec
ld-linux-aarch64.so.1
libc.so.6
libz.so.1
```

Como el binario era dinámico, ejecutarlo directamente daba:

```text
-sh: ./kexec: not found
```

Solución: wrapper `kexec-run`.

`tv/kexec-run`:

```sh
#!/bin/sh

DIR="$(cd "$(dirname "$0")" && pwd)"

PRELOAD=/etc/ld.so.preload
EMPTY=/tmp/empty-ld-so-preload
BOUND=0

# webOS tiene /etc/ld.so.preload apuntando a una libSegFault ELF32.
# Nuestro kexec usa glibc aarch64, así que el loader intenta precargarla y avisa.
# Montamos temporalmente un preload vacío solo durante esta ejecución.
if [ -f "$PRELOAD" ]; then
  : > "$EMPTY"
  if mount --bind "$EMPTY" "$PRELOAD" 2>/dev/null; then
    BOUND=1
  fi
fi

"$DIR/ld-linux-aarch64.so.1" --library-path "$DIR" "$DIR/kexec" "$@"
RC=$?

if [ "$BOUND" = "1" ]; then
  umount "$PRELOAD" 2>/dev/null
fi

exit "$RC"
```

Prueba:

```sh
cd /tmp/usb/sda/sda1/lgc1-kexec
./kexec-run --version
```

Resultado:

```text
kexec-tools 2.0.28
```

---

## Initramfs de prueba

Estructura del repo:

```text
LG-C1-kernel/
  initramfs/rootfs/
  scripts/
  tv/
  tools/
  out/
```

Script de build `scripts/build-initramfs.sh`:

```sh
#!/bin/sh
set -eu

REPO="$(cd "$(dirname "$0")/.." && pwd)"
ROOT="$REPO/initramfs/rootfs"
OUT="$REPO/out/initramfs.cpio.gz"

mkdir -p "$REPO/out"

if [ ! -x /bin/busybox ]; then
  echo "ERROR: falta /bin/busybox."
  echo "Instala busybox-static:"
  echo "sudo apt install busybox-static"
  exit 1
fi

cp -L /bin/busybox "$ROOT/bin/busybox"
chmod +x "$ROOT/bin/busybox"

for app in sh mount cat echo sleep dmesg ls mkdir sync reboot grep cp find; do
  ln -sf /bin/busybox "$ROOT/bin/$app"
done

cd "$ROOT"
find . -print0 | cpio --null -ov --format=newc | gzip -9 > "$OUT"

ls -lh "$OUT"
```

Script de copia `scripts/push-to-tv.sh`:

```sh
#!/bin/sh
set -eu

TV="${TV:-root@192.168.2.121}"
DEST="${DEST:-/tmp/usb/sda/sda1/lgc1-kexec}"
REPO="$(cd "$(dirname "$0")/.." && pwd)"

scp "$REPO/out/initramfs.cpio.gz" "$TV:$DEST/initramfs.cpio.gz"
scp "$REPO/tv/kexec-run" "$TV:$DEST/kexec-run"
scp "$REPO/tv/kexec" "$TV:$DEST/kexec"
scp "$REPO/tv/ld-linux-aarch64.so.1" "$TV:$DEST/ld-linux-aarch64.so.1"
scp "$REPO/tv/libc.so.6" "$TV:$DEST/libc.so.6"
scp "$REPO/tv/libz.so.1" "$TV:$DEST/libz.so.1"

ssh "$TV" "cd '$DEST' && chmod +x kexec-run kexec ld-linux-aarch64.so.1"

echo "Copiado initramfs + kexec bundle a $TV:$DEST"
```

---

## Prueba `kexec -l`

Primera prueba con initramfs:

```sh
cd /tmp/usb/sda/sda1/lgc1-kexec

CMDLINE="$(cat ./cmdline-webos.txt) rdinit=/init loglevel=8"

./kexec-run -l ./Image \
  --initrd=./initramfs.cpio.gz \
  --dtb=./lgc1-running.dtb \
  --append="$CMDLINE"

echo "kexec_load_rc=$?"
```

Resultado:

```text
Can't open (/proc/kcore).
Warning, can't get the VA_BITS from kcore
Can't open (/proc/kcore).
kexec_load_rc=0
```

El warning de `/proc/kcore` no bloquea la carga.

---

## Mapa de memoria

`/proc/iomem` relevante:

```text
00000000-337fffff : System RAM
  00088000-017bbfff : Kernel code
  01838000-023eefff : Kernel data
47800000-7cffffff : System RAM
80000000-bfffffff : System RAM
fd200000-fd200fff : /amba/watchdog@fd200000
fe000000-fe000fff : /amba/serial@fe000000
```

Reserved memory desde device-tree:

```text
kdrv_buffer1:
  base 0x33800000
  size 0x4c800000
  no-map

kdrv_buffer2:
  base 0xa9800000
  size 0x16800000
  no-map
```

Interpretación:

```text
RAM baja:
  0x00000000 - 0x337fffff

Reservado / HMA:
  0x33800000 - 0x7fffffff

RAM alta usable:
  0x80000000 - 0xa97fffff

Reservado / HMA:
  0xa9800000 - 0xbfffffff
```

Carga original en memoria baja:

```text
kernel: 0x00088000
initrd: 0x023ef000
dtb:    0x024e5000
```

Carga high2 usada:

```sh
./kexec-run -l ./Image \
  --mem-min=0x80000000 \
  --mem-max=0xa9000000 \
  --initrd=./initramfs.cpio.gz \
  --dtb=./lgc1-running.dtb \
  --append="$CMDLINE"
```

Resultado debug:

```text
image_arm64_load: kernel_segment: 0000000080000000
kexec_load: entry = 0x824ea680 flags = 0xb70000

segment[0].mem = 0x80088000  kernel
segment[1].mem = 0x823ef000  initrd
segment[2].mem = 0x824e5000  dtb
segment[3].mem = 0x824ea000  purgatory
```

---

## Watchdog

Config del kernel:

```text
CONFIG_DPM_WATCHDOG=y
CONFIG_DPM_WATCHDOG_TIMEOUT=5
CONFIG_WATCHDOG=y
CONFIG_WATCHDOG_CORE=y
# CONFIG_WATCHDOG_NOWAYOUT is not set
CONFIG_ARM_SP805_WATCHDOG=y
```

Dispositivos:

```text
/dev/watchdog
/dev/watchdog0
```

Herramienta `wdctl-lgc1`:

```c
#include <errno.h>
#include <fcntl.h>
#include <linux/watchdog.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>

static int try_dev(const char *dev) {
    int fd = open(dev, O_RDWR | O_CLOEXEC);
    if (fd < 0) {
        printf("%s: open failed: %s\n", dev, strerror(errno));
        return 1;
    }

    printf("%s: opened\n", dev);

    int timeout = 120;
    if (ioctl(fd, WDIOC_SETTIMEOUT, &timeout) == 0) {
        printf("%s: WDIOC_SETTIMEOUT ok, timeout=%d\n", dev, timeout);
    } else {
        printf("%s: WDIOC_SETTIMEOUT failed: %s\n", dev, strerror(errno));
    }

    int get_timeout = 0;
    if (ioctl(fd, WDIOC_GETTIMEOUT, &get_timeout) == 0) {
        printf("%s: WDIOC_GETTIMEOUT=%d\n", dev, get_timeout);
    } else {
        printf("%s: WDIOC_GETTIMEOUT failed: %s\n", dev, strerror(errno));
    }

    int flags = WDIOS_DISABLECARD;
    if (ioctl(fd, WDIOC_SETOPTIONS, &flags) == 0) {
        printf("%s: WDIOS_DISABLECARD ok\n", dev);
    } else {
        printf("%s: WDIOS_DISABLECARD failed: %s\n", dev, strerror(errno));
    }

    if (write(fd, "V", 1) == 1) {
        printf("%s: magic close V written\n", dev);
    } else {
        printf("%s: magic close V failed: %s\n", dev, strerror(errno));
    }

    close(fd);
    printf("%s: closed\n", dev);
    return 0;
}

int main(void) {
    int rc0 = try_dev("/dev/watchdog0");
    int rc1 = try_dev("/dev/watchdog");
    return (rc0 && rc1) ? 1 : 0;
}
```

Compilación:

```sh
gcc -O2 -Wall -static -o tv/wdctl-lgc1 tools/wdctl-lgc1.c || gcc -O2 -Wall -o tv/wdctl-lgc1 tools/wdctl-lgc1.c
```

Resultado en TV:

```text
/dev/watchdog0: opened
/dev/watchdog0: WDIOC_SETTIMEOUT ok, timeout=43
/dev/watchdog0: WDIOC_GETTIMEOUT=43
/dev/watchdog0: WDIOS_DISABLECARD ok
/dev/watchdog0: magic close V written
/dev/watchdog0: closed
/dev/watchdog: opened
/dev/watchdog: WDIOC_SETTIMEOUT ok, timeout=43
/dev/watchdog: WDIOC_GETTIMEOUT=43
/dev/watchdog: WDIOS_DISABLECARD ok
/dev/watchdog: magic close V written
/dev/watchdog: closed
wdctl_rc=0
```

A pesar de ello, `kexec -e` sigue reiniciando a los ~10 segundos.

Conclusión:

```text
El reset no parece venir del watchdog Linux normal expuesto como /dev/watchdog.
Probablemente interviene DPM watchdog, firmware/PMU/TEE o reset externo de plataforma.
```

---

## Pruebas realizadas con `kexec -e`

### 1. Kernel LG + initramfs mínimo

Resultado:

```text
pantalla congelada
sonido petado / ruido
apagado o reset
sin marcadores
```

### 2. Kernel LG + initramfs con reboot automático

Resultado:

```text
pantalla congelada
reset a ~10 s
SSH vuelve tras reinicio
sin marcadores
```

### 3. Kernel LG cargado en RAM alta `0x80000000`

Resultado:

```text
pantalla congelada
reset a ~10 s
SSH vuelve
sin marcadores
```

### 4. Watchdog desactivado antes de `kexec -e`

Comando:

```sh
./wdctl-lgc1
sync
./kexec-run -e
```

Resultado:

```text
pantalla congelada
reset a ~10 s
sin marcadores
```

### 5. Control: kexec hacia webOS original

Se cargó:

```sh
CMDLINE="$(cat ./cmdline-webos.txt)"

./kexec-run -l ./Image \
  --mem-min=0x80000000 \
  --mem-max=0xa9000000 \
  --dtb=./lgc1-running.dtb \
  --append="$CMDLINE"
```

Luego:

```sh
./wdctl-lgc1
sync
./kexec-run -e
```

Resultado:

```text
También reset a ~10 s.
```

Esto demuestra que el problema no está en Android/initramfs.

---

## Logs post-reset

`bootd.log`:

```text
PowerOnReason = {"reason":"cpuAbnormal","returnValue":true}
```

`legacy-log`:

```text
dpm_watchdog timeout : 5
dev:vdec, message : [    0.205133] 0x000000000000-0x000000001000 : "reset vector"
```

Otros datos:

```text
/mnt/lg/cmn_data/var/pbs_reset_check = temp
/mnt/lg/cmn_data/*.dcd aparece tras las pruebas
```

El `.dcd` observado no contenía strings útiles.

---

## Diagnóstico actual

El estado actual se resume así:

```text
El kernel LG original se puede extraer y cargar con kexec.
El DTB real se puede pasar a kexec.
kexec-tools funciona.
kexec_load() devuelve éxito.
La ubicación de memoria high2 es válida.
Pero el segundo kernel no arranca limpiamente.
Incluso webOS original no arranca mediante kexec.
```

Hipótesis principales:

```text
1. machine_kexec de LG/O20 no deja la plataforma en estado válido.
2. Hay un DPM watchdog o monitor externo no controlable desde /dev/watchdog.
3. Firmware/TEE/PMU reinicia la CPU al detectar estado anómalo.
4. Algún coprocesador/audio/video/DMA queda activo y provoca reset.
5. El segundo kernel muere antes de UART/init, pero sin consola serie no se ve.
```

---

## Próximos pasos recomendados

### Opción A — UART serie

Necesaria para depuración real.

La cmdline ya usa:

```text
console=ttyAMA0,115200n81 earlycon ignore_loglevel loglevel=8
```

Buscar pads UART 3.3 V en la placa y capturar salida durante `kexec -e`.

Preguntas que UART debe responder:

```text
- ¿El purgatory entra?
- ¿El segundo kernel imprime algo?
- ¿Muere en early boot?
- ¿Hay panic antes de initramfs?
- ¿Hay reset externo sin logs?
```

### Opción B — Recompilar/instrumentar kernel LG

El kernel original ya falla por kexec, así que recompilarlo solo no basta.  
Pero sí permitiría instrumentar:

```text
machine_kexec
cpu reset path
watchdog/DPM
early printk
reboot notifiers
device shutdown paths
```

### Opción C — Evitar kexec

Mantener webOS y usar Android userspace/chroot/namespaces sobre el kernel vivo:

```text
webOS kernel vivo
↓
binder.ko cargado
↓
Android userspace parcial
```

No sería Android 100% como sistema principal, pero evita el punto que falla: `kexec -e`.

---

## Comandos útiles de seguridad

Descargar imagen kexec cargada:

```sh
cd /tmp/usb/sda/sda1/lgc1-kexec
./kexec-run -u 2>/dev/null || true
```

Desactivar hooks automáticos:

```sh
rm -f /var/lib/webosbrew/init.d/00kexec_usb_gate
rm -f /var/lib/webosbrew/init.d/00pause_boot_probe
rm -f /var/lib/webosbrew/init.d/00kexec_probe
rm -f /var/lib/webosbrew/init.d/99kexec_dryrun
```

Desactivar intento desde USB:

```sh
rm -f /tmp/usb/sda/sda1/lgc1-kexec/ENABLE_KEXEC
touch /tmp/usb/sda/sda1/lgc1-kexec/DISABLE_KEXEC
sync
```

No usar:

```sh
systemctl stop webapp-mgr.service
```

En esta TV congeló/tumbó la sesión SSH.

---

## Git

Añadir documentación:

```sh
git add README.md
git commit -m "Document LG C1 kexec experiments and current blocker"
git push
```

Si se añaden scripts:

```sh
git add scripts/ initramfs/ tv/kexec-run tools/
git commit -m "Add LG C1 kexec tooling"
git push
```

No commitear dumps propietarios o binarios extraídos:

```text
extracted/*.bin
Image
*.dtb
tv/kexec
tv/ld-linux-aarch64.so.1
tv/libc.so*
tv/libz.so*
out/*.cpio.gz
```

---

## Estado final de esta sesión

```text
Hito conseguido:
  kernel LG original extraído y cargable con kexec.

Bloqueo:
  kexec -e provoca cpuAbnormal/reset a ~10 s incluso al intentar arrancar webOS original.

Siguiente paso real:
  UART serie o instrumentación de kernel/machine_kexec.
```
