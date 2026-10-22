# Firmware Usage:
1. Flash your B800 to latest official firmware, wait for camera back on
2. Flash the customized firmware, wait for camera back to online. This step should takes longer, ETA 5 mins.
3. Create camera password if you didn't, then use admin + password created for ONVIF or RTSP. Enjoy!

# Reolink B800 (v2) Firmware Mod

Reverse-engineering and rebuilding the firmware of a Reolink B800 v2 IP camera to add
**ONVIF**, **RTSP**, and **corridor mode** — features the stock firmware ships but keeps
locked on this standalone model — while keeping the device stable.

> **Disclaimer.** This documents work done on my own hardware. Flashing modified firmware
> can brick your camera and voids any warranty. Nothing here is endorsed by Reolink or
> Novatek. Proceed only if you understand the recovery path for your device.

---

## Result

| Feature | Status | Notes |
| --- | --- | --- |
| ONVIF | Working | Port 8000, device discoverable and connectable |
| RTSP | Working | Port 554 |
| Corridor mode | Working | `support_corridor_mode="1"` + corridor AI model added |
| Intelligent H.265+ | Working | EVBR + MAQ rate control ported from the RP-PCB8MZ via a boot-time script |
| Auto-reboot / crash | Fixed | Was caused by a Linux/HDAL RAM-split that starved Linux |
| Debug backdoor | Removed in final build | telnet (`bindshell`) and the `trace.so` tracer are diagnostic-only |

All features were confirmed working on-device after flashing.

---

## Hardware & firmware

| Item | Detail |
| --- | --- |
| SoC | Novatek NT98566MQG (internal codename **NA51089**) |
| RAM | 128 MB total; stock device tree splits it **41 MB Linux / 87 MB HDAL** |
| Flash | 16 MB |
| Firmware container | Classic Reolink PAK: magic `0x32725913`, section table at offset `0xc`, `0x40`-byte stride, with blank placeholder slots interleaved between real sections |
| rootfs | squashfs 4.0 (xz, 256 KB blocks) |
| rootfs partition limit | 13,107,200 bytes |

**HDAL** is Novatek's proprietary media/ISP/AI/video-encode subsystem. Its memory is
reserved in the device tree (FDT) and is invisible to Linux. Reference firmware from the
RLC-810A and CX810 (both UBI/UBIFS, magic `0x23494255`) was used only for string/config
comparison — their rootfs format differs from the B800's squashfs, so files could not be
lifted directly.

---

## Problems & root causes

| Symptom | Root cause |
| --- | --- |
| Device crashes / auto-reboots a few minutes after ONVIF+RTSP are added; processes fail `pthread_create()` with `ENOMEM`/`EAGAIN` | The FDT hard-codes a 41 MB / 87 MB Linux/HDAL split out of 128 MB. 41 MB is too little once the main app plus ONVIF/RTSP all run. |
| ONVIF/RTSP processes run but never `listen()` on their ports | A three-layer gate: `dvr.xml` capability flags, the `dvr.xml` `standalone` attribute, and — the real switch — a persistent config file **outside** the rootfs. |
| Debug shell unusable (Ctrl+C drops the connection, multi-line input garbles) | The original `bindshell` used `dup2()` onto the socket with no pty, so there was no terminal line discipline. |
| Killing a running `onvif`/`rtsp` process reboots the device | `router` (which holds `/dev/watchdog`) monitors its registered children and reboots if one dies unexpectedly. Un-registered diagnostic copies can be killed safely. |

---

## Solutions & key findings

### 1. RAM re-split via an FDT patch

The real fix for the crashing/rebooting was the memory layout, not any software flag. A
from-scratch Python FDT (`.dtb`) parser/patcher (`dtc` was unavailable offline) rewrites
just the two 8-byte `reg` cells for `memory` and `hdal-memory/media`, shifting **8 MB from
HDAL to Linux** (41 MB/87 MB → **49 MB/79 MB**), keeping the file size and structure
otherwise byte-identical. After this, ONVIF/RTSP no longer hit `ENOMEM` and the device
stops auto-rebooting.

### 2. The real ONVIF/RTSP switch lives outside the rootfs

With the `dvr.xml` capability flags and `standalone="1"` all set correctly, the ports still
would not open. A live `LD_PRELOAD` trace of `socket`/`bind`/`listen` showed the processes
only ever doing UDP localhost binds, never a TCP `listen()` on the public port.

The actual on/off switch is in **`/mnt/para/service.cfg`** — a plaintext XML file on a
JFFS2 partition (`/dev/mtdblock7`, mounted at `/mnt/para`) that is **completely separate
from the rootfs** and untouched by reflashing:

```xml
<onvif enable="0" port="8000" discovery="0" />
<rtsp  port="554" enable="0" />
```

`router` reads this file at boot and, only when `enable="1"`, signals the `onvif`/`rtsp`
modules to proceed from their idle "wait probe" state into actually calling `listen()`.
Flipping both to `"1"` (plus `discovery="1"`) and rebooting opens the ports. ONVIF reuses
the camera's main admin account — no separate ONVIF user is needed, but a password must
exist on the device first.

> Because this file lives outside the rootfs, reflashing firmware never changes it — but a
> **factory reset restores its defaults** (everything disabled).

### 3. Boot-time self-heal

So the switch survives a factory reset, the final build adds a check to
`/etc/init.d/start_app`, right after `/mnt/para` is mounted and before `router` starts:
every boot it verifies the ONVIF/RTSP `enable`/`discovery` flags in `service.cfg` and
rewrites them to `"1"` if needed (and skips the write when they are already correct, to
avoid needless flash writes).

### 4. Diagnostic tooling (not in the final build)

- **`bindshell2`** — a `forkpty()`-based replacement for the original `bindshell`, giving a
  real pty so Ctrl+C, line editing, and job control work over a raw `nc` connection.
- **`trace.so`** — an `LD_PRELOAD` tracer for `socket`/`bind`/`listen`. The first version
  used `dlsym(RTLD_NEXT, …)` and failed on-device with `GLIBC_2.34 not found` (the modern
  cross-compiler's glibc vs. the device's much older libc); rewriting it to call
  `syscall(SYS_socket/bind/listen, …)` directly — needing only `syscall@GLIBC_2.4` — fixed
  it.

### 5. Intelligent H.265+ ported from a different model, in userspace

The B800's `device` binary has no "Intelligent H.265+" code — its static HDAL build lacks
`EVBR_AUTO` and `OUT_H26X_MAQ_DIFF`, and no app knows `support_smart_265`, so there is no UI
for it. But the B800 **kernel drivers do** expose the needed knobs from shell:

- `/proc/hdal/venc/cmd` — `vdoenc setevbr <path> <21 ints>` sets EVBR rate control live.
- `/proc/kdrv_vdocdc/diff_maq` — `<ch> <en> <str> <idx1> <idx2>` sets the MAQ diff, whose
  struct is identical to the RP-PCB8MZ's.
- `vdoenc encinfo <p>` dumps per-path Codec / RC Mode / W / H / BitRate / Fps to dmesg.

So instead of patching `device`, the feature is a small boot script (`/mnt/app/smart265.sh`)
that waits for the encoders to come up, auto-detects the main stream path (largest W×H),
and — only for H.265 — applies EVBR (with per-level GOP, QP, MotionAQ, StillFrameCnd and
MotionRatioThd) plus the MAQ diff, matching the RP-PCB8MZ's values. It re-checks every 30 s
and re-applies if `device` resets the encoder (e.g. after a settings change in the app). JND
is not ported — the kernel supports it but there is no simple interface. Config is an
optional `/mnt/para/smart265.cfg` (`ENABLE`, `LEVEL` 0/1/2, `GOP_SEC`, `CODEC`, `PATH_ID`,
`INTERVAL`) on the persistent partition; defaults are `ENABLE=1 LEVEL=1`.

### 6. AI models (`ppvd.bin`) are not portable across camera models

Swapping in a `ppvd` AI model from a different Reolink camera makes the device fail to
initialize entirely. The two models are compiled by **different generations of Novatek's NN
toolchain** — the stock model uses operator type `Conv`, input tensor `images`, and `_Y`
layer suffixes, while the borrowed model uses `Convolution`, `input.in0`/`.out0`, and an
extra `ILIF` section. The B800's AI runtime (`kdrv_ai.ko` / `kflow_ai.ko`, tied to kernel
4.19.91 and this NPU revision) cannot parse the newer IR, and because AI init is part of the
main pipeline bring-up, the whole system fails to come up. Using such a model would require
porting the source camera's entire AI runtime, and even then the NPU/ISP generation must
match. This is a format/toolchain incompatibility, not a memory or size limit.

---

## Final build

Version `4075_2410211800`. rootfs 12,845,056 bytes (limit 13,107,200 — ~256 KB free).

Two PAK sections are modified; the kernel, uboot, and loader are untouched:

- **FDT** (section 1) — the 8 MB RAM re-split. Its `version.json` entry has `forbid_ol_up=0`
  and `ver` bumped to `5772` so it is guaranteed to flash over whatever FDT is installed.
- **rootfs** (section 6) — ONVIF/RTSP enabled + boot self-heal, corridor mode + its AI model
  (`/mnt/src/CNNLib/para/ppvd_corridor.bin`) with `support_corridor_mode="1"`, and the
  Intelligent H.265+ script (`/mnt/app/smart265.sh`) launched from `start_app`. The telnet
  backdoor is removed; `trace.so` was never included.

### Build & flash

A custom pure-Python squashfs 4.0 (xz) rebuilder applies the rootfs changes; `pakler`
assembles the PAK. From the stock PAK:

```sh
# 1. replace the FDT section with the memory-re-split one
pakler -r -n 1 -f B800_fdt_8mshift.bin -o step1.pak \
    B800_IPC_5158M8M_V2_4057_2409132131_B800_8MP_REOLINK.pak

# 2. replace the rootfs section
pakler -r -n 6 -f B800_SMART265_4075_2410211800.bin \
    -o B800_SMART265_4075.pak step1.pak
```

> The firmware is flashed through the device's own upgrade mechanism, which honors the
> per-section `forbid_ol_up` flag in `version.json`. The FDT section defaults to
> `forbid_ol_up=1` (skipped on upgrade); set it to `0` **and** bump the section `ver` above
> what is currently installed for the FDT to actually be written. Each later build must keep
> incrementing the FDT `ver` (5773, 5774, …) to keep overwriting.

### Verify after flashing

```sh
cat /mnt/app/version_file      # expect 4075_2410211800
free                           # ~45/49 MB confirms the re-split FDT took (not stock 41 MB)
ps  | grep -E "onvif|rtsp|smart265"  # running; bindshell absent in the final build
netstat -tlnp | grep -E "554|8000"   # both LISTEN
cat /proc/vencrc/info          # main stream RC Mode -> EVBR (Intelligent H.265+ active)
cat /proc/hdal/comm/info       # HDAL free memory — watch this when corridor AI runs
```

---

## Known limits & notes

- **Partition headroom is nearly gone** — ~256 KB free on the rootfs. The corridor AI model
  consumed most of the space freed by removing the telnet backdoor; little room remains for
  further additions.
- **HDAL free memory is tight** (~5.5 MB with ONVIF/RTSP running, measured before corridor
  was added). Corridor-mode AI inference needs more HDAL, not less, so after enabling it,
  check `/proc/hdal/comm/info`; if free memory runs out, reduce the FDT shift (e.g. 8 MB →
  4 MB) to give HDAL more room, watching that Linux stays above ~41 MB where ONVIF/RTSP
  start to fail.
- **Factory-reset behavior is assumed, not fully verified** — the self-heal assumes a reset
  restores `service.cfg` defaults rather than deleting the file. If a reset deletes it and
  `router` recreates it, the self-heal would need adjusting.
- **A future official OTA** may reflash the FDT (its `forbid_ol_up` allows it) and revert the
  RAM split.

---

## Tooling built for this project

- A standalone FDT (`.dtb`) parser and byte-exact `reg`-cell patcher (no `dtc` needed).
- A pure-Python squashfs 4.0 (xz, 256 KB block) reader/rebuilder that preserves inode
  numbers, permissions, ownership, mtimes, and symlinks, and a verifier that diffs two
  images and checks xz-stream parameters against the kernel's squashfs constraints.
- `smart265.sh` — the userspace Intelligent H.265+ port driving the EVBR/MAQ `/proc` knobs.
- `bindshell2` (pty shell) and `trace.so` (`LD_PRELOAD` syscall tracer) for live diagnosis.
- ARM32 disassembly via `llvm-objdump --triple=armv7-linux-gnueabihf` (the distro
  `objdump` lacked ARM support), plus manual PLT/GOT resolution for the stripped binaries.
