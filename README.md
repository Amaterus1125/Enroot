# Enroot

**Cross-compiled Linux From Scratch, adapted for PRoot targets (no reboot, no root).**

Enroot picks up where mainline Cross Linux From Scratch and the [LFS Cross Edition](https://www.linuxfromscratch.org/~xry111/lfs/view/clfs-ng-systemd/) leave off. Both assume you can reboot into the target system once the temporary toolchain is built. On Android under Termux/PRoot, that assumption breaks — there's no kernel handoff, no bootloader, no root. Enroot documents and scripts the path from "cross-compiled toolchain" to "usable userland running under PRoot," which nothing else currently covers.

## Why this exists

- The original CLFS project (2006–2017) is archived and inactive.
- Pierre Labastie's Cross Edition modernized cross-compilation for LFS 13.0, but its endpoint is "reboot into the target hardware" — not applicable to a sandboxed, rootless Android environment.
- No public project currently documents building an LFS-methodology system that ends in a **PRoot namespace instead of a bootable target.**

## Relationship to upstream LFS

Enroot is not a fork of the LFS book — it's a **continuation** for a different target environment.

| Phase | Source | Notes |
|---|---|---|
| Toolchain bootstrap, temp system cross-compile | LFS 13.0 Cross Edition (credited, unmodified methodology) | Standard cross-compile-before-chroot approach |
| Target boot | **Diverges here** | Cross Edition: reboot into target. Enroot: no reboot — build for PRoot execution instead |
| Init / session management | Custom | No systemd-under-PRoot; elogind in place of systemd-logind |
| Desktop environment | Custom | XFCE, D-Bus `org.freedesktop.login1` contract satisfied via elogind |

## Status / known blockers

- [ ] Verify what XFCE actually requires without `systemd-logind` (in progress)
- [ ] elogind build + integration for PRoot userland
- [ ] VNC server for display output (no direct framebuffer/DRM access under PRoot)
- [ ] systemd-under-PRoot — primary anticipated blocker, likely resolved by *not* using systemd at all in this target (elogind-only init)
- [ ] Document divergence point precisely (which LFS chapter/step Enroot's instructions replace)

## Repo structure (proposed)

```
enroot/
├── README.md
├── docs/
│   ├── 00-overview.md          # why this exists, how it differs from Cross Edition
│   ├── 01-toolchain.md         # cross-compile phase, references upstream LFS steps
│   ├── 02-target-divergence.md # exact point where Enroot deviates from reboot-based CLFS
│   ├── 03-proot-userland.md    # building the PRoot-executable rootfs
│   ├── 04-init-elogind.md      # elogind setup, D-Bus session handling
│   └── 05-xfce-under-proot.md  # desktop environment specifics, known breakage
├── scripts/
│   ├── build-toolchain.sh
│   ├── build-temp-system.sh
│   ├── build-proot-rootfs.sh
│   └── verify-hostreqs.sh
├── configs/
│   └── (elogind, D-Bus, XFCE session configs)
└── CREDITS.md                  # attribution to LFS project, Pierre Labastie's Cross Edition
```

## Credits

Built on the methodology of:
- [Linux From Scratch](https://www.linuxfromscratch.org/) (Gerard Beekmans et al.)
- [LFS Cross Edition](https://www.linuxfromscratch.org/~xry111/lfs/view/clfs-ng-systemd/) (Pierre Labastie)

Enroot does not reproduce book content verbatim — it documents the additional/divergent steps needed for a PRoot target.

## License

GPL-3.0
