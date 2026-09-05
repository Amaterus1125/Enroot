# Package 01 — Binutils (Pass 1)

**Why first (after headers):** Both Glibc and GCC test the available linker/assembler during their own configure steps to decide which features to enable. Binutils has to exist before either can be built.

Env (same as headers step):
```bash
source ~/arm64-build-env.sh   # LFS=/mnt/arm64lfs, LFS_TGT=aarch64-linux-gnu
```

## 1. Get source into the tree

```bash
cd $LFS/sources
wget https://ftp.gnu.org/gnu/binutils/binutils-2.46.0.tar.xz
ls $LFS/sources/
```

## 2. Extract and set up an out-of-tree build directory

```bash
tar xf binutils-2.46.0.tar.xz
mkdir -v binutils-build
cd binutils-build
```

**Why out-of-tree (build directory as a sibling, not nested inside the source):**

Binutils (and GCC, next) explicitly recommend never building directly inside the extracted source folder. Reasons this actually matters, not just convention:

1. **Keeps pristine source clean** — `configure`/`make` generate a large volume of object files, generated headers, and cache files. Building in-tree tangles all of that into the same folder as the actual source.
2. **Cheap to nuke and restart** — if a build goes wrong, `rm -rf binutils-build` and start over with a guaranteed-clean slate, while `binutils-2.46.0/` stays untouched.
3. **Same source, multiple configurations** — this source gets reused later for **Binutils Pass 2** with different flags. Keeping build output separate from source means no need to re-extract the tarball a second time — just create a fresh `binutils-build-pass2/` pointed at the same clean source.

Correct layout — sibling directories, not nested:
```
$LFS/sources/
├── binutils-2.46.0/        ← pristine extracted source, stays clean
├── binutils-2.46.0.tar.xz  ← original tarball
├── binutils-build/         ← empty; build happens HERE
├── linux-6.18.10/
└── linux-6.18.10.tar.xz
```

`../binutils-2.46.0/configure` works because you're standing inside `binutils-build/` (sibling to the source) and reaching up-and-over via `../` into the neighboring source folder. If `binutils-build` were nested inside the source folder instead, that relative path breaks and the separation is defeated.

## 3. Configure

```bash
../binutils-2.46.0/configure \
    --prefix=$LFS/tools \
    --with-sysroot=$LFS \
    --target=$LFS_TGT \
    --disable-nls \
    --disable-werror
```

**Flag notes:**
- `--prefix=$LFS/tools` — installs into the temp toolchain location.
- `--with-sysroot=$LFS` — cross build looks inside `$LFS` for target libraries, not the host's own.
- `--target=$LFS_TGT` (`aarch64-linux-gnu`) — makes this a cross-linker/assembler, not native.
- `--disable-nls` — skips internationalization, unneeded for temp tools.
- `--disable-werror` — stops the host compiler's warnings from aborting the build.

## 4. Build and install

```bash
make -j$(nproc)
make install
```

## 5. Verify

```bash
$LFS/tools/bin/aarch64-linux-gnu-ld --version
```

Should print a version string confirming an `aarch64` target linker — if it reports the host's own architecture instead, something in `--target`/`$LFS_TGT` didn't take.

## Commit checkpoint

1. `docs: add binutils pass 1 build notes (aarch64, out-of-tree)`
2. `build: binutils 2.46.0 cross-compiled for aarch64-linux-gnu`
3. `install: binutils pass 1 installed to $LFS/tools, cross-ld verified`

## Next

`packages/03-gcc-pass1.md`
