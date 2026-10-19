# Asahi USB-C Display Handoff

A handoff for getting HDMI out of a USB-C hub on Fedora Asahi Remix, on a MacBook Pro 14" (2023, M2 Max, Mac14,5 / J414c).

The stock Asahi kernel can't output video over USB-C yet. Asahi's experimental [`fairydust`](https://github.com/AsahiLinux/linux/tree/fairydust) kernel branch can, on one port. This repo holds the research and a review of the third-party [asahi-fairydust-display](https://github.com/bharambetejas/asahi-fairydust-display) build script, written up in [`CLAUDE.md`](CLAUDE.md) so that Claude Code running on Asahi can do the build, checks and rollback step by step.

> fairydust is unsupported by the Asahi team and meant for developers. Expect quirks: hot-plug is unreliable, only one USB-C display works, and some monitors get wrong colours or missing modes.

## On Asahi

1. Install Claude Code:
   ```bash
   curl -fsSL https://claude.ai/install.sh | bash
   ```
   If `claude` isn't found, open a new terminal. If it crashes on start (Asahi uses 16K memory pages), install the Node version instead: `sudo dnf install -y nodejs npm && sudo npm install -g @anthropic-ai/claude-code`.
2. Clone this repo and start Claude Code in it. It reads `CLAUDE.md` automatically.
   ```bash
   git clone https://github.com/sys-skye/asahi-usbc-display-handoff ~/fairydust-work && cd ~/fairydust-work && claude
   ```
3. Send: **Read CLAUDE.md and start Phase 1.**
4. Keep a second terminal window open. Claude can't type your sudo password, so it hands you root commands to run there; paste the output back.
5. After the reboot it asks for, pick the `-fairydust` entry in GRUB, then continue the same conversation:
   ```bash
   cd ~/fairydust-work && claude --continue
   ```

## Which port

Only one port carries display output: on the **left side, the USB-C port nearer the front**. The devicetree calls it "USB-C Left Front".

```
 left side of the MacBook Pro 14"
 back (hinge)                                         front (trackpad)
 ┌──────────────────────────────────────────────────────────────────┐
 │  [ MagSafe ]   [USB-C]   [USB-C]                       ( o )     │
 └──────────────────────────────────────────────────────────────────┘
                              ▲ plug the hub here     headphones
```

Booting with the hub already plugged in is more reliable than hot-plugging. The hub must use DisplayPort Alt Mode, as most simple USB-C hubs with HDMI do. DisplayLink hubs and Thunderbolt docks won't work this way.

## What's in here

- [`CLAUDE.md`](CLAUDE.md): the agent brief. It covers the hardware facts, the script review (pinned to commit `833444e0` with SHA-256 checksums), five phases (pre-flight, build, verify, reboot and test, make it stick), what the agent must ask before doing, and a rollback plan.
- `.gitignore`: keeps the machine-specific files created along the way (bootloader backup, pre-flight notes, script clone) out of git.

## Sources

- [Asahi Linux: Progress Report, Linux 6.19](https://asahilinux.org/2026/02/progress-report-6-19/)
- [AsahiLinux/linux, fairydust branch](https://github.com/AsahiLinux/linux/tree/fairydust)
- [bharambetejas/asahi-fairydust-display](https://github.com/bharambetejas/asahi-fairydust-display)
- [AsahiLinux/asahi-scripts: update-m1n1](https://github.com/AsahiLinux/asahi-scripts/blob/main/update-m1n1)
