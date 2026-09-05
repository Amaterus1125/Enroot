# Package 006 — Binutils (Pass 2)

**Why:** Rebuilds Binutils now that a real target Glibc/libstdc++ exist, so it links against the actual target C library instead of the bootstrap-only setup Pass 1 used.

**Key difference from Pass 1:** Pass 1 was built *to run on the host* (x86_64) while *producing* aarch64 output — a cross-tool. Pass 2 is built *to run on aarch64 itself* natively. That's why the flags change from `--target=$LFS_TGT` (Pass 1) to `--host=$LFS_TGT` (Pass 2) — this is the actual technical distinction between the two passes, not just a naming difference.

## 1. Out-of-tree configure

```bash
cd $LFS/sources
mkdir -v binutils-build2
cd binutils-build2

../binutils-2.46.0/configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../binutils-2.46.0/config.guess) \
    --with-sysroot=$LFS \
    --libdir=/usr/lib \
    --disable-nls \
    --disable-werror \
    --enable-shared \
    --enable-64-bit-bfd
```

**Flag notes:**
- `--prefix=/usr` (not `$LFS/tools` like Pass 1) — this installs into the real target sysroot, since Pass 2 becomes part of the actual ARM64 system.
- `--libdir=/usr/lib` — forces libraries into `/usr/lib` rather than risking the `lib64` default seen in the libstdc++ step.

## 2. Build and install

```bash
make -j$(nproc)
make DESTDIR=$LFS install
```

##  Known issue: `PATH_MAX` undefined error in `ld/ldmain.c`

**Symptom:** build fails around `ld/ldmain.c` with `PATH_MAX` undeclared.

**Do not** just add `#include <limits.h>` to `ldmain.c` — in this build, that header was already present from the start; the file just doesn't chain to the definition on this particular sysroot. Adding a duplicate include shifts the error line number but doesn't fix anything, and adding `<linux/limits.h>` directly into `ldmain.c` treats the symptom, not the cause.

**Real root cause:** the target sysroot's `linux/limits.h` — which should have come from the Package 01 (Linux Kernel Headers) step — never actually landed, or landed incomplete. `limits.h` → `bits/posix1_lim.h` → `linux/limits.h` is the real chain, and if the last link is missing, `PATH_MAX` is never defined anywhere reachable.

**Diagnose directly:**
```bash
find $LFS/usr/include -path "*/linux/limits.h"
cat $LFS/usr/include/linux/limits.h 2>/dev/null
```

If this file is missing or empty, that confirms it — re-copy it from the kernel source used in Package 01:
```bash
cd $LFS/sources/linux-6.18.10
cp -rv usr/include/linux/limits.h $LFS/usr/include/linux/
cat $LFS/usr/include/linux/limits.h    # should show: #define PATH_MAX 4096
```

**Then clean-retry the whole Binutils Pass 2 build** (don't patch `ld/ldmain.c` at all — the fix belongs in the sysroot headers, not the binutils source):
```bash
cd $LFS/sources/binutils-build2
rm -rf *
../binutils-2.46.0/configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../binutils-2.46.0/config.guess) \
    --with-sysroot=$LFS \
    --libdir=/usr/lib \
    --disable-nls \
    --disable-werror \
    --enable-shared \
    --enable-64-bit-bfd

make -j$(nproc)
```

**If you'd already patched `ldmain.c` directly** while chasing this (adding `#include <limits.h>` or `<linux/limits.h>` manually), revert it once the sysroot fix is confirmed — leaving stray manual patches in third-party source is exactly the kind of thing that causes confusing repeat errors on a future rebuild:
```bash
cd $LFS/sources/binutils-2.46.0/ld
sed -n '1,20p' ldmain.c    # inspect before removing anything blindly
```

This exact bug (PATH_MAX undefined in a cross-compile sysroot) is a documented, known issue on aarch64 cross-builds generally — it isn't specific to a bad setup, it's a real gap that appears whenever the kernel headers copy step misses this one file.

## 3. Verify

```bash
find $LFS/usr/lib -iname "*bfd*"
find $LFS/usr/bin -name "*ld*"
```

Should show real target binaries/libraries: `$LFS/usr/bin/ld`, `ld.bfd`, `ldd`, `pldd`, and `libbfd-*.so`/`.a`, `libctf-nobfd.so*` landing in `$LFS/usr/lib` (not `lib64`, confirming `--libdir=/usr/lib` took effect).

## Commit checkpoint

1. `docs: add binutils pass 2 build notes, PATH_MAX/linux-headers root cause`
2. `fix: recopy linux/limits.h into sysroot (Package 01 header copy was incomplete)`
3. `build: binutils 2.46.0 pass 2 compiled + installed for aarch64-linux-gnu native`

## Next

`packages/07-gcc-pass2.md`
