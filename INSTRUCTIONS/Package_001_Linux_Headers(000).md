# Package 001 — Linux Kernel Headers

**Why:** Not compiled — just extracted and installed into the build tree. The cross-compiler needs to know the target's syscall interface before anything else can be built against it.

Env used in this build (note: different from earlier draft, this is what was actually run):

```bash
export LFS=/mnt/arm64lfs
export LFS_TGT=aarch64-linux-gnu
export PATH=$LFS/tools/bin:/usr/bin:/bin
mkdir -pv $LFS/{tools,sources}
```

Save this as `~/arm64-build-env.sh` and `source` it in every new terminal used for this project — it's intentionally not persisted automatically, to keep the real system environment untouched.

## 1. Get the source into the isolated tree

Copied from the existing x86_64 LFS `/sources`, not re-downloaded, since it was already on hand:

```bash
mkdir -p $LFS/sources
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.10.tar.xz
ls $LFS/sources/
```

## 2. Extract and generate headers

```bash
cd $LFS/sources
tar xf linux-6.18.10.tar.xz
cd linux-6.18.10

make mrproper
make ARCH=arm64 headers
```

`ARCH=arm64` is what makes this generate aarch64 syscall headers instead of the host's own architecture — this is the one flag in this whole step that actually matters.

## 3. Clean up and install into the build tree

```bash
find usr/include -name '.*' -delete
rm usr/include/Makefile
mkdir -pv $LFS/usr
cp -rv usr/include $LFS/usr
```

The `find ... -delete` and `rm Makefile` steps strip out build-system leftovers that aren't real headers — only the actual include files should end up in `$LFS/usr/include`.

## Verify

```bash
ls $LFS/usr/include | head
```

Should show real kernel headers (`asm`, `linux`, `asm-generic`, etc.) — if this directory is empty or missing, the `make ARCH=arm64 headers` step didn't run cleanly and needs to be redone before touching Binutils.

## Commit checkpoint

1. `docs: add linux kernel headers build notes (aarch64)`
2. `build: linux 6.18.10 headers generated for ARCH=arm64`
3. `install: kernel headers copied to $LFS/usr/include`

## Next

`packages/02-binutils-pass1.md`
