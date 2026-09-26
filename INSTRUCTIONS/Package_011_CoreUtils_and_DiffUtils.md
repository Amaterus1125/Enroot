# Package 11 — Coreutils & Diffutils
 
**Status: ✅ COMPLETE** (Diffutils' fix also changes your permanent build environment — see the end of this file.)
 
Grouped together because Diffutils' fix directly affects how every package *after* it gets built, so it's worth reading through both before moving on.
 
---
 
## Part A — Coreutils (Section 1, step 15)
 
**Why:** `ls`, `cat`, `cp`, and the rest of the everyday command-line utilities — needed almost immediately by later build steps, not just the final system.
 
### 1. Fetch source, check for the required patch
 
```bash
cd $LFS/sources
wget https://ftp.gnu.org/gnu/coreutils/coreutils-9.10.tar.xz
tar xf coreutils-9.10.tar.xz
cd coreutils-9.10
```
Coreutils needs `coreutils-9.10-i18n-1.patch` per the build-order file. LFS-book patches are version-specific, so check for the exact match rather than grabbing any i18n patch found online:
```bash
find / -iname "coreutils-9.10-i18n*" 2>/dev/null
```
If found in your original x86_64 BLFS `/sources` (not inside `$LFS`), copy it in and apply:
 
```bash
cp /sources/coreutils-9.10-i18n-1.patch $LFS/sources/
cd $LFS/sources/coreutils-9.10
patch -Np1 -i ../coreutils-9.10-i18n-1.patch
```
Confirm clean application — should show `patching file ...` lines with **no** `FAILED` or `.rej` mentions anywhere in the output.
### 2. Configure
 
```bash
mkdir -v build
cd build
 
../configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../build-aux/config.guess) \
    --enable-install-program=hostname \
    --enable-no-install-program=kill,uptime \
    gl_cv_macro_MB_CUR_MAX_good=yes
```

**Flag notes:**
- `--enable-install-program=hostname` — Coreutils doesn't install `hostname` by default; this turns it on.
- `--enable-no-install-program=kill,uptime` — these two overlap with versions provided by other packages later in the build (procps-ng), so they're deliberately excluded here to avoid a conflict.
- `gl_cv_macro_MB_CUR_MAX_good=yes` — a cross-compile cache override, same category of fix as the ones seen in Diffutils below (a runtime-behavior check `configure` can't actually execute during a cross build, so the known-good answer is asserted directly).
### 3. Build and install
 
```bash
make -j2
make DESTDIR=$LFS install
```
