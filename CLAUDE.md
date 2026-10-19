# Task: get HDMI out of my USB-C hub on Fedora Asahi Remix (MacBook Pro 14" M2 Max)

You are Claude Code running on my Fedora Asahi Remix install. An earlier Claude session (on macOS, which you cannot reach) already researched this and reviewed the build script. Everything it found is below, so don't redo the research. Work through the phases in order, check each step, and explain results to me in plain language. Stop and ask before anything listed under "Ask first".

This file is CLAUDE.md in the handoff repo, cloned to `~/fairydust-work`, which is your working directory. Begin at Phase 1 when I say so. The script clone, preflight.txt and the boot.bin backup also go in this folder; .gitignore keeps them out of git. Don't commit or push to this repo unless I ask.

## The machine
- MacBook Pro 14-inch (2023), Apple M2 Max, model Mac14,5. Asahi devicetree target: J414c, file `apple/t6021-j414c.dtb`. Confirm: `tr -d '\0' < /proc/device-tree/compatible` should contain `apple,j414c`.
- Dual boot with macOS. Internal disk: Asahi EFI system partition (~0.5 GB, holds the m1n1 bootloader `m1n1/boot.bin`, mounted at /boot/efi), Linux /boot (~1.1 GB), Linux root (~80 GB). Never touch any APFS/macOS partition.
- macOS on this Mac currently cannot mount the Asahi EFI partition (its FAT driver fails), so bootloader backups and restores have to be done from Linux.

## Background (already researched)
- The stock Asahi kernel has no USB-C DisplayPort Alt Mode. The Mac's built-in HDMI port (right side) already works on the stock kernel; it's the fallback if this fails.
- Asahi's experimental `fairydust` branch of https://github.com/AsahiLinux/linux adds DP Alt Mode. It was announced in the Asahi "Linux 6.19" progress report (Feb 2026) as unsupported and meant for developers. The branch was last updated 2026-09-08.
- It only works on ONE "blessed" port per machine. For j314/j316/j414/j416 (14/16-inch MacBook Pro, M1/M2 Pro/Max) that is the **front-left USB-C port**: on the left side, the USB-C port nearer the front edge and the headphone jack, away from MagSafe. Source: fairydust commit "arm64: dts: apple: t60xx: j[34]1[46]: Add dp-altmode hacks" (Janne Grunau).
- Devicetree details (arch/arm64/boot/dts/apple/t600x-j314-j316.dtsi, included by t602x-j414-j416.dtsi, with `#define ENABLE_DCPEXT_TYPEC`): connector `typec1` ("USB-C Left Front", CD321x PD controller `hpm1` at i2c0 0x3f) has `displayport = <&dcpext1>`. `dcpext1` uses `atcphy1` in DP mode through `atcphy1_xbar` (mux-index 2), `apple,dptx-phy = <1>`, connector-type "DP". The HDMI port stays on `dcpext0` via `atcphy3`.
- Four hardware blocks are involved: DCP (display coprocessor), DPXBAR (display crossbar), ATCPHY (Apple Type-C PHY), ACE/CD321x (USB-PD controller, driver `tipd`/tps6598x).
- Known quirks: hot-plug is unreliable (booting with the hub already plugged in works better), some displays get wrong or oversaturated colours or missing modes, and only one USB-C display works.
- **How the devicetree reaches Linux:** the m1n1 bootloader bundles devicetrees into `/boot/efi/m1n1/boot.bin` and hands one to U-Boot, then GRUB, then Linux. Every kernel, stock or fairydust, boots with whatever devicetrees are inside boot.bin. So the fairydust kernel only gets DP Alt Mode if boot.bin contains the fairydust DTBs. That is the most important thing to verify.
- The hub must use DisplayPort Alt Mode, as most simple USB-C hubs with HDMI do. DisplayLink hubs (in `lsusb` as vendor 17e9) won't work this way, and neither will Thunderbolt-dock display tunnelling. MST multi-display hubs give at most one display.

## The build script (already reviewed)
- Repo: https://github.com/bharambetejas/asahi-fairydust-display (third-party, unofficial). Reviewed at commit `833444e03dbd51329344895943e765637ed908c1`.
- SHA-256 at that commit:
  - asahi-fairydust-build.sh: `e309f93c8ee7df99d49326201f3283c1f882bfb0460bcb9757f4682cc4f498c0`
  - asahi-fairydust-uninstall.sh: `6717864b15417397f51f1d6d466a7714c58f1f679273fdc63219d9d2e70e66a1`
  - patches/0001-drm-apple-reconnect-DP2HDMI-output-on-resume.patch: `ceb8617cdb4a196ed88045a77008c14093536bb9a47d51575f0ed30c4cf5853a`
- Verdict: not malicious. It downloads only from GitHub (AsahiLinux/linux) and the Fedora repos, plus rustup only if Fedora's rust-src package is missing. It installs the new kernel alongside the stock one.
- What it does, in order: dnf installs build deps; clones fairydust (blobless) to ~/linux-fairydust; applies the patch; copies /boot/config-$(uname -r); enables RUST, DRM_ASAHI, DRM_APPLE and TYPEC_DP_ALTMODE; disables module signing; sets LOCALVERSION=-fairydust; runs `make -j$(nproc)`; runs `sudo make modules_install dtbs_install vdso_install install`; symlinks /usr/lib/modules/$KVER/dtb to /boot/dtbs/$KVER and /usr/src/linux to the tree; **repoints /boot/dtb to /boot/dtbs/$KVER and runs `sudo update-m1n1`**; makes the GRUB menu visible with a 5 s timeout and runs grub2-mkconfig; writes /etc/modules-load.d/fairydust-typec.conf. The display autostart/udev step is commented out in main().
- The patch changes drivers/gpu/drm/apple/dcp.c so dcp_platform_resume() calls dcp_enable_dp2hdmi_hpd() instead of a bare enable_irq(). The earlier session checked it against the fairydust dcp.c: that function exists and calls enable_irq itself, so the IRQ enable/disable count stays balanced. It only affects the built-in HDMI port after resume. Keep it, or build with SKIP_PATCHES=1 to leave the branch untouched.
- Problems found in the script and README (account for these):
  1. The README says the original kernel is "untouched". Not fully true: boot.bin's devicetrees are shared by all kernels.
  2. Whether the script's /boot/dtb trick works depends on how update-m1n1 resolves DTBS on this install (Phase 1, step 3).
  3. A later `dnf upgrade` that installs a new stock kernel probably re-runs update-m1n1 and may put stock DTBs back into boot.bin, which silently breaks DP on the fairydust kernel.
  4. Its 15 GB free-space check is too low for Fedora's full config with debug info. Aim for about 30 GB free on /, and check that /boot (1.1 GB) has room for one more kernel and initramfs.
  5. The uninstall script leaves the GRUB/BLS entry behind, points /boot/dtb at a "dtb-<kver>" target that may not exist, and overwrites the DTBS setting. Don't run it as-is; use the Rollback section.
  6. The README's xrandr/DP-1 checks only work on X11. On KDE Wayland use `kscreen-doctor -o`; on anything, read `/sys/class/drm/*/status`.
  7. The script is interactive (y/N prompts) and uses sudo throughout, including after the roughly 1-hour build.

## How we work with sudo
Your Bash tool can't answer sudo password prompts. For anything that needs root, give me the exact commands (batched, so I run as few as possible). I'll run them in a separate terminal window and paste the output back. Run everything that doesn't need root yourself. Note /boot/efi is usually readable only by root.

## Ask first
Ask me before: running the build script; any write to /boot, /boot/efi, /etc or /usr; changing update-m1n1 config; installing or removing packages; rebooting; deleting anything; committing or pushing to this repo.
Never: touch macOS/APFS partitions, use dd/mkfs/parted/fdisk/wipefs, modify firmware, disable SELinux, or run the repo's uninstall script unmodified.

## Phase 1: pre-flight (read-only, except the backup)
1. Confirm the machine and OS: compatible string (apple,j414c), `uname -r`, `/etc/os-release`, `getconf PAGESIZE` (expect 16384).
2. Disk: `df -h / /boot` (and /boot/efi). Need about 30 GB free on /. If there's less, stop and give me options: free space, or (with my approval) build with debug info turned off.
3. Work out exactly which DTB files update-m1n1 bundles into boot.bin:
   - `command -v update-m1n1`, `rpm -qf "$(command -v update-m1n1)"`, then read the whole script.
   - Read every config it sources: /etc/default/update-m1n1 and/or /etc/sysconfig/update-m1n1, and /etc/m1n1.conf.
   - `ls -l /boot/dtb; ls /boot/dtbs/`. Find what runs it on kernel install: look in /usr/lib/kernel/install.d/ and /etc/kernel/install.d/ and grep them for m1n1.
   - Conclude in plain language: after the script points /boot/dtb at the fairydust DTBs and runs update-m1n1, will boot.bin contain the fairydust DTBs? If not, propose the smallest change that makes it so, and wait for my approval.
4. Record the current state in ~/fairydust-work/preflight.txt: `uname -r`, `readlink -f /boot/dtb`, the full contents of the update-m1n1 config file(s), `ls -l /boot`, `ls /usr/lib/modules`, `rpm -q kernel-16k kernel`, and the default boot entry (`sudo grubby --default-kernel`, which I'll run).
5. Clone the reviewed script and verify it:
   ```
   cd ~/fairydust-work
   git clone https://github.com/bharambetejas/asahi-fairydust-display
   cd asahi-fairydust-display && git checkout 833444e03dbd51329344895943e765637ed908c1
   sha256sum asahi-fairydust-build.sh asahi-fairydust-uninstall.sh patches/*.patch
   ```
   The hashes must match the ones above. If they don't, stop.
6. Bootloader backup (root, I run it):
   ```
   sudo cp -p /boot/efi/m1n1/boot.bin /boot/efi/m1n1/boot.bin.stock && sudo cp /boot/efi/m1n1/boot.bin ~/fairydust-work/boot.bin.stock && sudo chown "$USER": ~/fairydust-work/boot.bin.stock && sudo sha256sum /boot/efi/m1n1/boot.bin /boot/efi/m1n1/boot.bin.stock ~/fairydust-work/boot.bin.stock
   ```
   All three hashes must match.

## Phase 2: build (I run it, you watch)
- Summarise the plan and get my OK.
- I run this in a separate terminal, with the laptop on power, answering y to its prompts:
  `cd ~/fairydust-work/asahi-fairydust-display && NO_REBOOT=1 ./asahi-fairydust-build.sh`
  It takes about 60-90 minutes and may ask for my sudo password again at the install step.
- You: check `~/fairydust-build.log` now and then (`tail -n 40`), tell me how it's going, and diagnose any failure from the log. Likely failures: rustc and rust-src version mismatch (see `make rustavailable`), disk full, the patch not applying (rerun with SKIP_PATCHES=1 only with my OK), or `make install`/dracut errors.
- If it fails after the compile finished, don't start from scratch: the tree at ~/linux-fairydust is reused and `make` is incremental. Prefer running only the missing install steps by hand (from the script's own commands) over rerunning the whole script.

## Phase 3: verify before rebooting
1. `KVER=$(make -C ~/linux-fairydust -s kernelrelease)`. Check that /boot/vmlinuz-$KVER, /boot/initramfs-$KVER.img, /usr/lib/modules/$KVER and /boot/dtbs/$KVER/apple/t6021-j414c.dtb exist, and that a boot entry for $KVER exists in /boot/loader/entries/ (may need root).
2. Confirm the installed j414c DTB is the DP Alt Mode one: decompile it with `dtc -I dtb -O dts` (package `dtc`; ask before installing) and check that the "USB-C Left Front" connector has a `displayport` property and that dcpext1 is `okay`.
3. Confirm boot.bin now contains that exact DTB. I run: `sudo cp /boot/efi/m1n1/boot.bin /tmp/boot.bin.new && sudo chmod a+r /tmp/boot.bin.new`. Then you run:
   ```
   python3 -c "import sys;d=open(sys.argv[1],'rb').read();b=open(sys.argv[2],'rb').read();print('fairydust j414c DTB inside boot.bin:', d in b)" /boot/dtbs/$KVER/apple/t6021-j414c.dtb /tmp/boot.bin.new
   ```
   Upstream update-m1n1 concatenates the raw .dtb files (`cat "$M1N1" $DTBS`), so a byte match is a reliable check. If this install's update-m1n1 works differently (from your Phase 1 reading), adapt the check. If the result is False, stop, fix how DTBS resolves (with my approval), rerun update-m1n1 and check again.
4. Tell me what to expect after the reboot, then ask before I reboot.

## Phase 4: reboot and test
- I reboot, pick the GRUB entry containing "-fairydust", log in, and restart you with `cd ~/fairydust-work && claude --continue`.
- Without the hub:
  - `uname -r` contains -fairydust
  - `ls /proc/device-tree/aliases | grep dcpext` shows dcpext0 and dcpext1
  - `find /proc/device-tree -path '*connector*' -name displayport` finds the property
  - `lsmod | grep -iE 'typec|tps6598|tipd|atcphy|apple|asahi'` (report what's loaded)
  - `glxinfo -B | grep -i renderer` must not say llvmpipe (llvmpipe means the GPU driver wasn't built)
  - `for c in /sys/class/drm/card*-*; do echo "$(basename "$c"): $(cat "$c/status")"; done`
- Then I plug the hub, with the monitor attached, into the front-left USB-C port. Check: a connector flips to "connected"; `journalctl -k -b | grep -iE 'dcp|atcphy|tipd|cd321|typec|dptx|xbar|hpd' | tail -60`; `ls /sys/class/typec/`; partner alt modes `grep -H . /sys/class/typec/port*-partner/*/svid` (DisplayPort SVID is ff01); `lsusb` (DisplayLink is 17e9).
- If there's no picture, try in this order: (a) reboot with the hub already plugged in; (b) confirm it's the front-left port; (c) connect the HDMI cable to the hub before plugging the hub in; (d) try a plain USB-C to HDMI or USB-C to DP adapter to separate hub problems from kernel problems; (e) check the hub isn't DisplayLink. Report what you see; don't guess.
- If colours look off or a resolution is missing, that's a known DCP issue. Try another mode in display settings.

## Phase 5: make it stick
- Offer (ask first) to make it the default: `sudo grubby --set-default /boot/vmlinuz-$KVER`.
- Explain that after any `dnf upgrade` that installs a new kernel, DP Alt Mode may stop working on the fairydust kernel because boot.bin may get stock DTBs again. The check is `ls /proc/device-tree/aliases | grep dcpext1`. The fix is to point /boot/dtb (or DTBS, depending on what Phase 1 found) at /boot/dtbs/$KVER, run `sudo update-m1n1` and reboot. Offer to write ~/fairydust-work/restore-fairydust-dtbs.sh that does the check and fix (ask first).
- To update fairydust later: review the new commits, `git -C ~/linux-fairydust pull`, then rebuild.

## Rollback
- Something is off but it boots: choose the stock kernel in the GRUB menu.
- The stock kernel misbehaves with the fairydust devicetrees: restore the bootloader from Linux with `sudo cp /boot/efi/m1n1/boot.bin.stock /boot/efi/m1n1/boot.bin` and reboot. update-m1n1 also leaves a boot.bin.old.
- Remove the fairydust kernel (from the stock kernel, ask first): first check how it was registered (BLS entry). Then `sudo kernel-install remove $KVER`; `sudo rm -rf /boot/dtbs/$KVER /usr/lib/modules/$KVER`; `sudo rm -f /etc/modules-load.d/fairydust-typec.conf`; repoint /boot/dtb to the original target from preflight.txt; restore the update-m1n1 config if it was changed; `sudo update-m1n1`; `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`.
- If Linux won't boot at all (no GRUB): hold the power button at startup and choose macOS. macOS's `diskutil mount` currently fails on the Asahi EFI partition; the FSKit route `sudo mount -F -t msdos -o rdonly /dev/disk0s4 /private/tmp/esp` hasn't been tried. Tell me this risk honestly before Phase 2. It's low, because update-m1n1 only re-bundles the same m1n1 and U-Boot with different devicetrees.

## References
- Asahi Linux 6.19 progress report: https://asahilinux.org/2026/02/progress-report-6-19/
- fairydust branch: https://github.com/AsahiLinux/linux/tree/fairydust (devicetree: arch/arm64/boot/dts/apple/t600x-j314-j316.dtsi and t602x-j414-j416.dtsi; display driver: drivers/gpu/drm/apple/)
- Upstream update-m1n1: https://github.com/AsahiLinux/asahi-scripts/blob/main/update-m1n1
- Build script repo: https://github.com/bharambetejas/asahi-fairydust-display
