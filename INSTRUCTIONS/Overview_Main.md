# Enroot — Overview

## What this is -

Enroot is an attempt to get a Chiral-style Linux userland (built LFS/BLFS-methodology, cross-compiled) running **inside a PRoot namespace on Android**, with no reboot, no root, and no real kernel handoff. It picks up where the LFS Cross Edition stops — that project cross-compiles a temp toolchain and then reboots into the target on real hardware. Enroot needs a different ending, because there's no "target hardware" to reboot into: just Termux's PRoot sandbox on top of Android's existing kernel.

The goal isn't "port LFS to ARM" (that's just an architecture flag). The goal is "make an LFS-built system usable when the thing running it can never own the kernel, never load modules, and never has real root."

## Why this is hard, specifically

Everything below assumes normal LFS/BLFS behavior and breaks under PRoot:

- **No real root** — PRoot fakes root via ptrace, but you can't load kernel modules, can't mknod real device nodes in most cases, can't touch raw hardware.
- **No systemd** — systemd expects to be PID 1, manage cgroups, and control the real kernel. None of that is available under PRoot. This is the single biggest architectural fork from mainline LFS/BLFS.
- **No reboot into target** — every "verify by rebooting" step in the LFS/Cross Edition book has to be replaced with "verify by re-entering the PRoot session."
- **No direct display access** — no DRM/KMS, no framebuffer. Anything graphical (XFCE) has to go through a VNC server instead of a normal X session on real hardware.
- **D-Bus/login1 dependents** — XFCE and a lot of BLFS desktop packages assume `org.freedesktop.login1` exists, which normally comes from systemd-logind. Under Enroot this has to come from elogind instead, and it's not yet confirmed everything XFCE needs is actually satisfied by elogind alone.

## Approach

1. **Reuse, don't reinvent, the toolchain phase.** Cross-compiling the temporary toolchain works the same whether the target is real hardware or a PRoot namespace — CPU is CPU. This phase follows the Cross Edition book near-verbatim, credited, not rewritten.
2. **Fork hard at the "boot into target" step.** This is the actual divergence point Enroot exists to document. Instead of writing a bootloader/kernel and rebooting, Enroot packages the built system as a rootfs image that Termux/PRoot can `proot` into directly.
3. **Replace systemd with elogind + minimal init.** No PID-1 systemd. A lightweight init (or just a shell-driven startup) handles session setup, and elogind provides the D-Bus login1 interface that desktop packages expect.
4. **Get XFCE running headless-to-VNC.** No real display hardware is reachable, so the desktop target is XFCE running under a VNC server, accessed from Android's own display.
5. **Document every blocker as it's found**, since this is the part with zero prior art — the value of this repo is largely in the trail of "here's what broke and how it was fixed," not just a working end state.

## Honest feasibility estimate

| Component | Confidence it's achievable | Why |
|---|---|---|
| Cross-compiled toolchain (ARM64) | ~95% | Already done once for the ARM64 build order — known-working territory |
| Base LFS system running under PRoot | ~80% | PRoot-hosted Linux userlands are a well-trodden pattern (this is what Termux itself, and projects like Andronix, already rely on) |
| elogind replacing systemd-logind fully | ~50–60% | Works for many D-Bus/login1 consumers in other minimal setups, but XFCE's exact dependency surface hasn't been verified yet — this is the open unknown |
| XFCE fully usable via VNC | ~60% | Desktop environments run under PRoot+VNC elsewhere (proven pattern via Termux:X11 and similar projects), but full XFCE session stability (panel, session manager, thunar) under this stack is unverified for this specific build |
| NetworkManager / full networking stack under PRoot | ~30–40% | PRoot can't touch real network interfaces directly — this will likely need to proxy through Android's existing networking rather than manage it directly, which may mean dropping NetworkManager entirely for something much thinner |
| Overall: "usable daily-driver Chiral-on-Android" | ~40–50% | Not because any single piece is impossible, but because the compounding of PRoot restrictions across every subsystem means something in the chain will likely need a workaround that isn't a straight LFS/BLFS install |

**Bottom line:** the toolchain and base system are close to a sure thing. The desktop environment and networking layers are where this could genuinely stall — and that's fine, because even a partial result (working base system + documented blockers) is the useful output of this repo, not just a fully working GUI environment.

## What "done" looks like

Not necessarily a polished daily-driver desktop. A realistic finish line:

- A scripted, repeatable build from cross-compiled toolchain to bootable-under-PRoot base system
- elogind + minimal init working, documented, with exact XFCE dependency findings written up
- XFCE reachable via VNC, even if some components (panel applets, certain thunar plugins) are dropped
- A clear, honest writeup of what didn't work and why — this is genuinely new information nobody else has published
