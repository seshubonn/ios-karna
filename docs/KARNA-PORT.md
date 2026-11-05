# Porting ios-sweet to the POCO X3 (`karna` / `surya`) — engineering log and runbook

Created 5 Nov 2026. Target unit: POCO X3 India, M2007J20CI, 6/128 GB, MIUI 13 (Android 12),
serial `654e4145`. Pre-port device report: `../device/karna-20261105-182514/` (workspace).

## Why this port is cheap

The X3 is the same SM7150-AC / sm6150 platform as the Redmi Note 10 Pro that ios-sweet
already boots. In ios-sweet's architecture every board value the kernel reads comes from
the **generated ADT** (`tools/mkadt.py`), not from the kernel binary, and the UFS root
device is found by **GPT partition name** (`ufsroot=`, default `userdata`). So a same-SoC
board port is: new ADT values + one boot-arg. The kernel, shim and userland are unchanged.

## Verified board values

| Value | karna | Source |
| --- | --- | --- |
| cont_splash | `0x9c000000`, `0x0f00000` (+ dfps `0x9cf00000` `0x100000`) | LineageOS `android_kernel_xiaomi_surya` @ lineage-22.2, `arch/arm64/boot/dts/qcom/sm6150.dtsi` |
| Panel | 1080×2400 **video mode**, Huaxing/Tianma NT36672C | `dsi-panel-nt36672c-huaxing-fhd-video.dtsi`, `…-tianma-…` |
| UFS host | `ufshc@1d84000` size `0x3000` | same file |
| GICv3 | GICD `0x17a00000`/0x10000, GICR `0x17a60000`/0x100000 | same file |
| DDR | low `0x80000000` 2G + high `0x100000000` `0xfcc00000` (6 GiB) | same platform as sweet; **confirm from live FDT after unlock** |
| `userdata` | GPT name, `/dev/block/sda16`, f2fs | live by-name map |
| Touch | Novatek NT36672C, **SPI**, reset GPIO 8, IRQ GPIO 9 (`0x2001`), vddio `L18A`, 8 MHz | downstream `touchscreen.dtsi` ("Xiaomi J20 672C TM 6.67" module criteria); input stack reports `NVTCapacitiveTouchScreen` |
| Buttons | PM6150 PON + PM6150L GPIO (same PMIC pair as sweet) | platform |

Everything not listed is identical to sweet by construction (same SoC): PSCI, 19.2 MHz
timebase, kryo470 cores (0–5 A55, 6–7 A76), UFSHCI 2.1 with LUN0 GPT.

Risk register (first hardware boot):

1. **FB size mismatch** — if ABL's splash window on karna is not `0x9c000000..0x9d000000`,
   console writes go to wrong RAM. Mitigation: verify against the live FDT (`cont_splash`
   node) from OrangeFox before the first boot; regenerate the ADT if it differs.
2. **Upper-bank XPU** — sweet notes unknown runtime TZ/XPU protection in the upper DDR
   bank would show as an instant reset; fall back to the low-memory payload (no
   `--highmem`) which uses `0xb0000000`.
3. **Video-mode assumption** — the huaxing/tianma panel dtsi say video mode, like sweet's;
   if the unit turns out command-mode, pixels need a DSI refresh kick (see Pixel 4/6a
   ports).
4. **Goodix driver probing** — disabled via `sweettouch=0` on karna (wrong controller and
   bus); without it the I²C probe on SE7 could hang early boot.

## Runbook — day 0 (do now; starts the 7-day clock)

The unit is bootloader-locked (`unlocked: no`, `secure: yes`, `get_unlock_ability: 0`,
`sys.oem_unlock_allowed=0`). Xiaomi's token is server-signed after a Mi account is bound
to the device for 168 h; there is no local bypass (all fastboot unlock variants return
`Token Verify Failed`; EDL/test-point routes risk a hard brick and are out of scope).

1. Create a Mi account (global.account.xiaomi.com) and sign in on the phone.
2. Insert a SIM, disable Wi-Fi (binding wants mobile data).
3. Developer options → **OEM unlocking** ON.
4. Developer options → Mi Unlock status → **Add account and device** → must say
   "Added successfully". This starts the 168 h wait.
5. Keep the phone charged; do not factory-reset (it can reset the binding timer).

## Runbook — day 7 (unlock and first boot)

```sh
# 1. Unlock (wipes userdata). Mi Unlock on Windows, or on this Mac:
#    PATH=/opt/homebrew/bin:$PATH .venv-miunlock/bin/miunlock   (same server API)
# 2. Back up calibration partitions (unique per unit!) and dump the live board:
adb reboot bootloader
fastboot boot OrangeFox-R11.0_A12-surya.img      # temporary, nothing flashed
#   in recovery:  ./tools/collect_board.sh       -> persist/modemst/fsg/fsc/boot/dtbo/vbmeta + live FDT
# 3. Verify the ADT assumptions against the live FDT:
#    cont_splash reg, memory node, ufshc reg. If anything differs from the table
#    above, fix tools/mkadt.py and regenerate.
# 4. Unpack the stock DTB for the shim image header:
python3 tools/unpack_bootv2.py backup/boot.img dtb/boot_unpacked
# 5. Build (kernel from CI, see README) and boot:
fastboot boot shim/out/shim-boot.img
```

Milestones after first boot (Phase 2+ of the research plan):

1. Kernel console on panel / persistent log reachable → board values correct.
2. `rd=ufsroot0` after flashing the iOS root into `userdata` (built from the iPhone X
   16.7.16 IPSW per upstream docs) → Apple logo.
3. Lock screen (panel copy already works if 1 worked).
4. Novatek SPI touch driver (the real new engineering): QUP SE base from the live FDT,
   GPIOs 8/9, L18A rail via RPMh (borrow sweet's 0062 pattern), report parsing from
   UniXNU `NvtProtocol.c`. Boot-arg `sweettouch=0` stays until it exists.

## Artifact map (not in git)

- `shim/out/karna.adt`, `shim/out/payload.bin`, `shim/out/shim-boot.img` — built locally
- `xnu/out/kernel.development.sweet` — CI artifact
- `backup/`, `dtb/boot_unpacked/`, `research/ipsw-x/` — per-device, never commit
