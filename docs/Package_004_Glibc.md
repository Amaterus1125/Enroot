# Package 04 — Glibc

**Why:** The actual C library everything else in the ARM64 system links against. GCC Pass 1 was deliberately built minimal (`--without-headers`, `--with-newlib`, static-only) specifically so it could compile this next.

**Correction to older build-order notes:** M4 is *not* a build dependency for Glibc itself — that was a mistake in an earlier version of the package list. M4 gets picked up later, in the temporary-tools section.

## 1. Get source + patch

```bash
source ~/arm64-build-env.sh
cd $LFS/sources
wget https://ftp.gnu.org/gnu/glibc/glibc-2.43.tar.xz
cp /sources/glibc-fhs-1.patch $LFS/sources/ 2>/dev/null
ls $LFS/sources/ | grep glibc
```

Glibc patches are version- and book-edition-specific — if `glibc-fhs-1.patch` isn't already on hand from a prior BLFS build, confirm which LFS book edition it needs to come from before grabbing one blindly; a mismatched patch version can silently apply wrong.

## 2. Extract and patch

```bash
cd $LFS/sources
tar xf glibc-2.43.tar.xz
cd glibc-2.43
patch -Np1 -i ../glibc-fhs-1.patch
```

## 3. Out-of-tree configure

```bash
cd $LFS/sources
mkdir -v glibc-build
cd glibc-build

echo "rootsbindir=/usr/sbin" > configparms

../glibc-2.43/configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../glibc-2.43/scripts/config.guess) \
    --enable-kernel=4.19 \
    --with-headers=$LFS/usr/include \
    libc_cv_slibdir=/usr/lib
```

Glibc's configure is notoriously picky — check the last ~20 lines of output for anything indicating `--build` mis-detected or headers not found, before running `make`.

## 4. Build and install

```bash
make -j$(nproc)
make DESTDIR=$LFS install
```

## Critical: `DESTDIR=$LFS` is not optional, and depends on your shell actually having `$LFS` set

This is the single most dangerous step in the whole toolchain phase. If you run this in a **fresh terminal that never sourced `arm64-build-env.sh`**, `$LFS` is empty, and:

```bash
make DESTDIR=$LFS install
```

silently becomes:

```bash
make DESTDIR= install
```

— which is identical to plain `make install`, and will attempt to write glibc directly into your **real host system's `/usr`**. On most systems this fails with a permission error (regular users can't write to `/usr`), which is actually the safety net catching the mistake. But always verify before and after:

```bash
# BEFORE running make DESTDIR=$LFS install, confirm $LFS is actually set:
echo $LFS               # must print /mnt/arm64lfs, not blank

# AFTER, confirm your REAL host libc was never touched:
file /usr/lib/libc.so.6           # must say x86_64
```

If `file /usr/lib/libc.so.6` ever reports `aarch64`, stop immediately — that means the real host's C library got overwritten with an incompatible build. This has not happened in this project so far, but the check costs nothing and the failure mode is severe enough to check every single time this pattern (`DESTDIR=$LFS`) appears in later packages too.

## 5. Verify install landed in the right place

```bash
find $LFS/usr/lib -name "libc.so*"
find $LFS/usr/lib -name "crt1.o"
find $LFS/usr/lib -name "crti.o"
find $LFS/usr/lib -name "crtn.o"
```

All four should resolve under `$LFS/usr/lib` — **not** `$LFS/lib` (no `/lib` symlink exists at this stage, that's expected, not an error) and **not** left behind only in `$LFS/sources/glibc-build/` (that would mean it was compiled but never actually installed).

## Commit checkpoint

1. `docs: add glibc build notes, DESTDIR safety warning`
2. `build: glibc 2.43 configured + compiled for aarch64-linux-gnu`
3. `install: glibc installed to $LFS/usr via DESTDIR, host libc verified untouched`

## Next

`packages/05-libstdcxx.md`
