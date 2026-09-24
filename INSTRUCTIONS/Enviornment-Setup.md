# Package 00 — Build Environment Setup (Loopback Image + Partition + Sources)

Everything that has to exist *before* Package 01 (Linux Headers) can run. This is the actual sequence used to set up `/mnt/arm64lfs`.

## 1. Create the loopback image file

Instead of a real disk partition, the ARM64 build lives inside a single disposable file — a loopback image. This keeps the entire build state as one portable file, and means a broken build never touches the real system disk.

```bash
# 40GB is enough for the full cross-toolchain + temp system, with headroom
dd if=/dev/zero of=~/arm64build.img bs=1M count=40000 status=progress
```

`bs=1M count=40000` writes the file in 1MB chunks, 40,000 times, giving exactly 40GB (40000 × 1MB). `status=progress` just shows a live progress readout since this step alone can take a while depending on disk speed.

## 2. Format the image with a filesystem

The image file is currently just 40GB of zeroes — it needs an actual filesystem written into it before it can be mounted and used like a real disk:

```bash
mkfs.ext4 ~/arm64build.img
```

`mkfs.ext4` works directly on the file (not a device node) because Linux loopback devices let a regular file be treated as a block device once it's mounted with `-o loop`.

## 3. Set up the loopback device and mount point

```bash
sudo mkdir -pv /mnt/arm64lfs
sudo mount -o loop ~/arm64build.img /mnt/arm64lfs
sudo chown $(whoami):$(whoami) /mnt/arm64lfs
```

- `-o loop` tells `mount` to attach the image file to a loopback device automatically (no need to manually `losetup` it first — `mount -o loop` handles that internally).
- The `chown` step matters: without it, everything under `/mnt/arm64lfs` is owned by root, and the whole point of building as a non-root `lfs`-style user (or your own regular user, in this session's case) breaks immediately.

## 4. Directory structure inside the mounted image

```bash
export LFS=/mnt/arm64lfs
mkdir -pv $LFS/{tools,sources}
```

- `$LFS/sources` — every tarball gets extracted here.
- `$LFS/tools` — the temporary cross-toolchain (Binutils, GCC Pass 1) installs here, kept fully separate from what will eventually become the target root filesystem.

## 5. The environment script

This is what actually gets sourced at the start of every build session — save it once, reuse every time:

```bash
cat > ~/arm64-build-env.sh << 'EOF'
export LFS=/mnt/arm64lfs
export LFS_TGT=aarch64-linux-gnu
export PATH=$LFS/tools/bin:/usr/bin:/bin
mkdir -pv $LFS/{tools,sources}
EOF

source ~/arm64-build-env.sh
echo $LFS
echo $LFS_TGT
```

You should see `/mnt/arm64lfs` and `aarch64-linux-gnu` printed back. This is deliberately **not** persisted automatically in `.bashrc` — every new terminal for this project needs `source ~/arm64-build-env.sh` run explicitly. That's intentional: it keeps this cross-build environment from silently leaking into your normal shell sessions on the real system.

## 6. Source tarballs needed for Packages 01–03

These are copied from the existing x86_64 LFS `/sources` where already present. If any aren't already downloaded, use the commented `wget` lines below — uncomment the one you need and run it.

```bash
cd $LFS/sources

# Linux Kernel (Package 01 — headers)
# wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.10.tar.xz
cp /sources/linux-6.18.10.tar.xz $LFS/sources/ 2>/dev/null

# Binutils (Package 02 — pass 1)
# wget https://ftp.gnu.org/gnu/binutils/binutils-2.46.0.tar.xz
cp /sources/binutils-2.46.0.tar.xz $LFS/sources/ 2>/dev/null

# GCC (Package 03 — pass 1)
# wget https://ftp.gnu.org/gnu/gcc/gcc-15.2.0/gcc-15.2.0.tar.xz
cp /sources/gcc-15.2.0.tar.xz $LFS/sources/ 2>/dev/null

#  GCC's required math libraries (bundled inside gcc-15.2.0/ in Package 03)
# wget https://ftp.gnu.org/gnu/gmp/gmp-6.3.0.tar.xz
cp /sources/gmp-6.3.0.tar.xz $LFS/sources/ 2>/dev/null

# wget https://www.mpfr.org/mpfr-4.2.2/mpfr-4.2.2.tar.xz
cp /sources/mpfr-4.2.2.tar.xz $LFS/sources/ 2>/dev/null

# wget https://ftp.gnu.org/gnu/mpc/mpc-1.3.1.tar.gz
cp /sources/mpc-1.3.1.tar.xz $LFS/sources/ 2>/dev/null

ls $LFS/sources/
```

Pin every version above to whatever's actually current when you run this — these are the exact versions used in this build session, not guaranteed to be latest.

## 7. Verify everything before starting Package 01

```bash
echo $LFS              # /mnt/arm64lfs
echo $LFS_TGT           # aarch64-linux-gnu
mount | grep arm64lfs   # confirms the loopback image is actually mounted
ls $LFS/sources/        # should list all tarballs copied above
df -h /mnt/arm64lfs     # confirms ~40GB is available
```

If all four checks come back clean, the environment is ready.

## Unmounting / remounting later

```bash
sudo umount /mnt/arm64lfs
```

To resume a later session:

```bash
sudo mount -o loop ~/arm64build.img /mnt/arm64lfs
sudo chown $(whoami):$(whoami) /mnt/arm64lfs
source ~/arm64-build-env.sh
```

Everything built so far persists inside `arm64build.img` — it's one file that can be copied, backed up, or deleted and restarted from scratch without ever touching the real system.

## Commit checkpoint

1. `docs: add arm64 loopback image + partition setup`
2. `env: arm64build.img created (40GB), mounted at /mnt/arm64lfs, directory structure + env script in place`
3. `sources: linux/binutils/gcc/gmp/mpfr/mpc tarballs staged in $LFS/sources`

## Next

`packages/01-linux-headers.md`
