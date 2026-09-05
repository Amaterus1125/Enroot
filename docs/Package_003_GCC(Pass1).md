# Package 02 — GCC (Pass 1)

**Why:** Builds a cross-compiler targeting `aarch64-linux-gnu`, without a C library yet — just enough to compile Glibc next. A second GCC pass happens later once Glibc exists.

Env: same `arm64-build-env.sh` (`LFS=/mnt/arm64lfs`, `LFS_TGT=aarch64-linux-gnu`).

## 1. Get sources into the tree

```bash
cd $LFS/sources
cp /sources/gmp-6.3.0.tar.xz $LFS/sources/ 2>/dev/null
cp /sources/mpfr-4.2.2.tar.xz $LFS/sources/ 2>/dev/null
cp /sources/mpc-1.3.1.tar.xz $LFS/sources/ 2>/dev/null
cp /sources/gcc-15.2.0.tar.xz $LFS/sources/ 2>/dev/null
ls $LFS/sources/
```

## 2. Extract GCC, then bundle GMP/MPFR/MPC inside it

```bash
cd $LFS/sources
wget https://ftp.gnu.org/gnu/gcc/gcc-15.2.0/gcc-15.2.0.tar.xz
tar xf gcc-15.2.0.tar.xz
cd gcc-15.2.0

wget https://ftp.gnu.org/gnu/mpfr/mpfr-4.2.2.tar.xz
tar xf ../mpfr-4.2.2.tar.xz
mv -v mpfr-4.2.2 mpfr

wget https://ftp.gnu.org/gnu/gmp/gmp-6.3.0.tar.xz
tar xf ../gmp-6.3.0.tar.xz
mv -v gmp-6.3.0 gmp

wget https://ftp.gnu.org/gnu/mpc/mpc-1.3.1.tar.gz
tar xf ../mpc-1.3.1.tar.xz
mv -v mpc-1.3.1 mpc

cd ..
```

**What's actually happening here, and why it has to be exact:**

The downloaded tarballs (`gmp-6.3.0.tar.xz`, etc.) stay put in `$LFS/sources/` — they're never moved. What goes *inside* `gcc-15.2.0/` is the **extracted contents**, renamed to the specific bare names (`gmp`, `mpfr`, `mpc`) GCC's build system automatically looks for. That renaming isn't tidiness — GCC's `configure` script scans its own source root for subdirectories with exactly those names, and if found, builds and statically links them into the same GCC build with no extra `--with-gmp=` style flags needed.

Result:
```
$LFS/sources/
├── mpc-1.3.1.tar.xz          ← tarball stays here, untouched
├── gcc-15.2.0/
│   ├── mpc/                  ← extracted + renamed, lives inside gcc's tree
│   ├── gmp/
│   ├── mpfr/
│   ├── configure
│   └── ...(rest of gcc source)
```

**On the `../` in `tar xf ../mpc-1.3.1.tar.xz`:** `..` always means "up one directory from wherever you currently are" — it's purely relative to your current location, not a fixed path. Standing inside `gcc-15.2.0/`, `../` resolves to `sources/`, which is exactly where the tarball lives, so `../mpc-1.3.1.tar.xz` correctly reaches it. This is the same `..` symbol used in Binutils' `../binutils-2.46.0/configure` — same rule, different starting point, coincidentally landing on the same `sources/` folder both times because both build/source dirs sit one level deep in the same tree.

## 3. Out-of-tree build directory (same pattern as Binutils)

```bash
mkdir -v gcc-build
cd gcc-build

../gcc-15.2.0/configure \
    --target=$LFS_TGT \
    --prefix=$LFS/tools \
    --with-glibc-version=2.43 \
    --with-sysroot=$LFS \
    --with-newlib \
    --without-headers \
    --enable-default-pie \
    --enable-default-ssp \
    --disable-nls \
    --disable-shared \
    --disable-multilib \
    --disable-threads \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libquadmath \
    --disable-libssp \
    --disable-libvtv \
    --disable-libstdcxx \
    --enable-languages=c,c++
```

**Flag notes:**
- `--with-glibc-version=2.43` — tells GCC what Glibc version to expect on target; must match whatever Glibc gets built next, not left as a hardcoded number.
- `--without-headers` — no target C library headers exist yet at this stage.
- `--disable-multilib` — required on aarch64; this target doesn't support multilib configs.
- `--disable-shared` — static libs only this pass; `libgcc_eh.a` specifically is needed later for Glibc's build.
- `--disable-libstdcxx` — C++ standard library isn't built until Pass 2, so a separate `--disable-libstdcxx-pch` flag would be redundant here — don't add it, it doesn't apply to a pass that already skips libstdc++ entirely.
- No `gobject-introspection` flag exists for GCC — that's an unrelated GTK-stack package that comes up much later; nothing to configure here.
- `--enable-languages=c,c++` — only what's needed for the temp toolchain.

## 4. Build and install

```bash
make -j$(nproc)
make install
```

This is typically the longest single build in the whole toolchain phase (commonly 20–60+ minutes depending on CPU) — worth checking `configure` output carefully for anything failing before letting `make` run unattended for that long.

## 5. Verify

```bash
$LFS/tools/bin/aarch64-linux-gnu-gcc --version
```

Then confirm actual cross-compilation, not just a version string:

```bash
echo 'int main(){return 0;}' > /tmp/test.c
$LFS/tools/bin/aarch64-linux-gnu-gcc /tmp/test.c -c -o /tmp/test.o
file /tmp/test.o
```

`file` should report the object as ARM aarch64 — that's the real proof the cross-compiler works, not just that it runs.

## Commit checkpoint

1. `docs: add gcc pass 1 build notes (aarch64, gmp/mpfr/mpc bundled)`
2. `build: gcc 15.2.0 cross-compiled for aarch64-linux-gnu`
3. `install: gcc pass 1 installed to $LFS/tools, cross-compile output verified as aarch64`

## Next

`packages/03-glibc.md`
