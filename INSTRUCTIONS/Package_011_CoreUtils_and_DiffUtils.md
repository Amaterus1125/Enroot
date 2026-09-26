# Package 11 — Coreutils & Diffutils
 
**Status: ✅ COMPLETE** (Diffutils' fix also changes your permanent build environment — see the end of this file.)
 
Grouped together because Diffutils' fix directly affects how every package *after* it gets built, so it's worth reading through both before moving on.
 
---
 
## Part A — Coreutils (Section 1)
 
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
### 4. Verify
 
```bash
find $LFS/usr/bin -name "ls"
find $LFS/usr/bin -name "cat"
```
 
---
 
## Part B — Diffutils (Section 1)
**Why:** `diff`, `cmp`, and friends.
 
### 1. Fetch, extract, configure
 
```bash
cd $LFS/sources
wget https://ftp.gnu.org/gnu/diffutils/diffutils-3.12.tar.xz
tar xf diffutils-3.12.tar.xz
mkdir -v diffutils-build
cd diffutils-build
 
../diffutils-3.12/configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../diffutils-3.12/build-aux/config.guess)
```
### ⚠️ Known issue 1 — `strcasecmp` cross-compile check fails
 
**Symptom:** `configure` errors out because it needs to actually *run* a small compiled test program to check `strcasecmp`'s runtime behavior — impossible during cross-compilation, for the same root reason an ARM64 binary can't execute on the x86_64 host doing the building.
 
**Fix — assert the known-good answer directly**, skipping the runtime test:
 
```bash
rm -rf *
../diffutils-3.12/configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../diffutils-3.12/build-aux/config.guess) \
    gl_cv_func_strcasecmp_works=yes
```
 
Safe to assume `yes` here: this check only exists to catch broken libc implementations on obscure/legacy platforms, and the target here is a standard, modern glibc (2.43) — well past any risk this test is meant to catch.
 
Diffutils sometimes surfaces 2-3 of these `gl_cv_*` cross-compile checks in a row — check the full configure tail for any others before moving on, not just the first one hit.
 
### ⚠️ Known issue 2 — `PATH_MAX` undeclared (same recurring root cause as Binutils Pass 2 and M4)
 
**Symptom:** build fails referencing `PATH_MAX` undeclared, in a diffutils source file this time.
 
**Do not** hardcode `4096` directly into the specific source file that failed — that's fragile and only patches one occurrence; this exact symptom already showed up in Binutils Pass 2 and M4, confirming it's a **recurring symptom of one root cause**: this sysroot's `limits.h` header chain still doesn't reliably expose `PATH_MAX` everywhere it's expected.
 
**Better fix — solve it at the compiler-flag level for this whole package:**
 
```bash
rm -rf *
../diffutils-3.12/configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../diffutils-3.12/build-aux/config.guess) \
    gl_cv_func_strcasecmp_works=yes \
    CPPFLAGS="-include linux/limits.h"
 
make -j2
make DESTDIR=$LFS install
```
