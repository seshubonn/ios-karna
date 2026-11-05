# ios-karna

**Apple's iOS 16 on the POCO X3 — a board port of [ios-sweet](https://github.com/dipa-studio/ios-sweet).**

`karna` is the POCO X3 India (M2007J20CI); `surya` is the POCO X3 NFC. Same board, same
Qualcomm **SM7150-AC / Snapdragon 732G** as the Redmi Note 10 Pro (`sweet`) that ios-sweet
already boots — so the XNU kernel, the boot shim, the UFS root and the entire iOS 16.7.16
(iPhone X) userland carry over unchanged. This repo adds the **board layer**:

| | sweet (upstream) | karna (this port) | verified from |
| --- | --- | --- | --- |
| cont_splash framebuffer | `0x9c000000`, 0x1700000 | `0x9c000000`, **0x1000000** | LineageOS `sm6150.dtsi`: `cont_splash_region@9c000000 0x0f00000` + `dfps_data_region@9cf00000 0x100000` |
| Panel | 1080×2400 DSC **video mode** | 1080×2400 **video mode** (Huaxing/Tianma NT36672C FHD+) | `dsi-panel-nt36672c-{huaxing,tianma}-fhd-video.dtsi` — same continuous-splash trick, no refresh kick |
| UFS host | `0x1d84000` / 0x3000 | **identical** | `sm6150.dtsi` `ufshc@1d84000` |
| GIC | `0x17a00000` / `0x17a60000` | **identical** | `sm6150.dtsi` `intc@17a00000` |
| DDR banks | low `0x80000000`+2G, high `0x100000000`+0xfcc00000 | **identical** (6 GiB unit) | same platform; confirm against live FDT after unlock |
| Root partition | GPT name `userdata` (sda21) | GPT name `userdata` (sda16) — **no kernel change**, the driver matches by name | live `by-name` map, 5 Nov 2026 |
| Touch | Goodix GT9896, I²C QUP SE7 | **Novatek NT36672C, SPI** — Goodix driver disabled (`sweettouch=0`) until a Novatek SPI driver exists | live input stack: `NVTCapacitiveTouchScreen`; downstream DTS: reset GPIO 8, IRQ GPIO 9, vddio L18A, 8 MHz SPI |
| Identity | `xiaomi,sweet` | `xiaomi,karna|xiaomi,surya|` + `xiaomi,sweet` (SweetPlatformExpert still matches) | — |

Status: **prepared, not yet hardware-tested** — the target unit's bootloader is locked
(Xiaomi server-side unlock, see the runbook), so the first `fastboot boot` can only happen
after the unlock. Everything that can be built and verified before that is done here.

## Build

The kernel is **byte-identical to ios-sweet's** (same SoC, same `MACHINE_CONFIG=SWEET`,
same patch series — board values live in the ADT, not the kernel), but there are no upstream
release artifacts, so build it once via GitHub Actions (public repos get free macOS minutes):

```sh
gh workflow run xnu.yml          # or fork and use the GitHub web UI
gh run watch && gh run download RUN_ID -n xnu-kernel -D xnu/out
```

Then (needs `aarch64-linux-gnu-gcc`, `mkbootimg`, and the stock `boot.img` DTB from *your*
phone — see `docs/KARNA-PORT.md`):

```sh
python3 tools/mkadt.py shim/out/karna.adt --board karna --highmem
python3 tools/mkpayload.py xnu/out/kernel.development.sweet shim/out/karna.adt shim/out/payload.bin \
    --cmdline 'debug=0xc serial=3 cpus=6 keepsyms=1 -enable_kprintf_spam -noprogress rd=ufsroot0 unentitled_ios_sim_launch=1 ignition_level=1 wdt_test=no_user_checkin ufsrw=1 rootrw=1 launchd_unsecure_cache=1 wdt=-1 sweetfb_noconsole=1 sweetfb_hz=30 sweettouch=0'
shim/build.sh shim/out/payload.bin     # -> shim/out/shim-boot.img
```

The only differences from sweet's boot line are `--board karna` and `sweettouch=0`.

iOS root image (`userdata`), log pulling, crash symbolication: unchanged from upstream —
see `docs/UPSTREAM-sweet-README.md`.

## Boot (after the bootloader is unlocked)

```sh
fastboot boot shim/out/shim-boot.img   # RAM-only; nothing flashed, Power+VolDown to leave
```

First hardware milestone: kernel console on the panel (the persistent log lands in the last
4 MiB of `userdata`; read it from OrangeFox with `tools/sweet-log-pull.sh`).

## What is still missing on karna

- **First-hardware-boot verification** (blocked on the Mi Unlock 168-hour server wait).
- **Touch**: the Novatek NT36672C SPI driver (Phase 4 in `docs/KARNA-PORT.md`): reset/IRQ
  GPIOs 8/9, vddio L18A; borrow the SPI transport shape from HackiOS `hakios.goodix.c` and
  report parsing from UniXNU `ShengNovatekTouch` (`NvtProtocol.c`); the downstream
  `nt36672c` driver has the register map.
- Confirm the live FDT values (esp. DDR banks and the touch QUP SE base) once readable.

Credits: everything here stands on [ios-sweet](https://github.com/dipa-studio/ios-sweet)
(dipa-studio) and Apple's open-source XNU. iOS is Apple's software; nothing here
distributes it.
