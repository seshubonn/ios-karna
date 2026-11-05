# Working on ios-karna (POCO X3 board port of ios-sweet)

- Read `NOTES.md` (karna handoff at the top, upstream sweet history below) and
  `docs/KARNA-PORT.md` before starting: verified board values, risk register, runbook.
- The kernel, shim and userland are inherited from ios-sweet unchanged; board values live
  in the ADT (`tools/mkadt.py --board karna`). Do not patch the kernel for board data —
  change the ADT.
- Update the karna handoff in `NOTES.md` after each milestone. Record commits, CI run IDs,
  exact build/boot results. Distinguish observations from guesses.
- Commit completed changes. Use `gh` for GitHub Actions kernel builds and logs.
- Device bring-up uses temporary `fastboot boot` images. Never flash boot/recovery/modem/
  persist; the iOS root goes into `userdata` only. The unit is bootloader-locked until the
  Mi Unlock wait completes — nothing on the phone can be tested before that.
- Keep device dumps, serial numbers and downloaded Apple artifacts out of git.
