# Package 007 — GCC (Pass 2, final cross-compiler)

**Why:** The final compiler — once this installs cleanly, the toolchain phase (Section 0) is done, and every subsequent package compiles using this compiler instead of the bootstrap Pass 1 one.

Reuses the same `gcc-15.2.0` source tree, with `gmp`/`mpfr`/`mpc` still bundled inside from Package 03.

## 1. Out-of-tree configure

```bash
cd $LFS/sources
mkdir -v gcc-build2
cd gcc-build2

# Explicitly export these BEFORE configure — see the known issue below for why
export AS_FOR_TARGET=$LFS/tools/bin/aarch64-linux-gnu-as
export LD_FOR_TARGET=$LFS/tools/bin/aarch64-linux-gnu-ld

../gcc-15.2.0/configure \
    --build=$(../gcc-15.2.0/config.guess) \
    --host=$LFS_TGT \
    --target=$LFS_TGT \
    --prefix=/usr \
    --libdir=/usr/lib \
    --with-sysroot=$LFS \
    --enable-languages=c,c++ \
    --enable-default-pie \
    --enable-default-ssp \
    --disable-nls \
    --disable-multilib \
    --disable-libatomic \
    --disable-libgomp \
    --disable-libquadmath \
    --disable-libsanitizer \
    --disable-libssp \
    --disable-libvtv \
    --disable-fixincludes
```

## Critical flag: `--target=$LFS_TGT` must be present, not just `--host`/`--build`

This is the single most important line in this whole file. If `--target=$LFS_TGT` is missing (only `--host` and `--build` set), GCC's target-tool auto-detection breaks, and the build silently falls back to using the **host's native `as`** instead of the cross-assembler — even though the correct `aarch64-linux-gnu-as` binary exists and works fine on its own.

**Symptom this causes:** `make` fails deep inside `libgcc/config/aarch64/lse.S` with errors like:
```
Error: no such instruction: 'adrp x16,__aarch64_have_lse_atomics'
Error: unknown pseudo-op: '.inst'
```
These look like the assembler doesn't support AArch64/LSE instructions at all — but it's a red herring. The real assembler supports everything; the build just isn't calling it.

**How to confirm this is what's happening, if you hit this error:**
```bash
grep '^AS_FOR_TARGET' gcc-build2/Makefile
```
If this prints `AS_FOR_TARGET=as` (bare `as`, no path, no target prefix), that confirms the wrong assembler is being invoked.

**Fix — don't patch around it, reconfigure clean:**
```bash
rm -rf gcc-build2   # stale Makefile/config.cache can't just be patched
mkdir -v gcc-build2
cd gcc-build2

export AS_FOR_TARGET=$LFS/tools/bin/aarch64-linux-gnu-as
export LD_FOR_TARGET=$LFS/tools/bin/aarch64-linux-gnu-ld

# re-run the full configure above, making sure --target=$LFS_TGT is present this time
```

Verify the fix took before running `make` again:
```bash
grep '^AS_FOR_TARGET' Makefile
```
Should now show the correct resolved path to `aarch64-linux-gnu-as`.

**Sanity check if you're unsure whether your cross-assembler itself is even the problem:** test it directly, bypassing GCC entirely:
```bash
cat > /tmp/lsetest.s << 'EOF'
.arch armv8-a+lse
.inst 0xd503201f
EOF
$LFS/tools/bin/aarch64-linux-gnu-as /tmp/lsetest.s -o /tmp/lsetest.o
echo "exit code: $?"
```
Exit code `0` confirms the real assembler is fine and the problem is purely which one GCC's build is calling — not a capability gap.

## 2. Resource constraints (relevant on modest hardware)

On a 4GB RAM / 4-core machine, this build realistically takes ~2 hours and can hit swap thrashing. Before running `make` at full parallelism:
```bash
free -h    # check available RAM/swap headroom
top        # watch for swap thrashing once make starts
```
If memory pressure is visible, drop parallelism rather than fight it:
```bash
make -j1     # or -j2
```
Adding a swapfile ahead of time if none exists is cheap insurance against an OOM-killed build hours in.

## 3. Build

```bash
make 2>&1 | tail -80
```

Piping through `tail -80` on any failure (rather than scrolling hundreds of lines manually) is the standard way to actually find the real error line instead of guess-fixing blind.

## 4. Install

```bash
make DESTDIR=$LFS install
```

## Critical: `--prefix=/usr` only means the right thing with `DESTDIR=$LFS`

`--prefix=/usr` in the configure step above is only correct **because** install is run with `DESTDIR=$LFS` — that combination redirects the install target into `$LFS/usr/...`. If `make install` is ever run **without** `DESTDIR=$LFS` on this same build (outside a chroot, on the host), `--prefix=/usr` points at the **host's real `/usr`**, and the install will either:
- fail with `Permission denied` (the expected, safe outcome for a non-root user — this is the system correctly refusing to let a cross-build touch the real OS), or
- if accidentally run as root, actually overwrite parts of the real host system.

This exact `Permission denied` (`mkdir: cannot create directory '/usr/libexec/gcc/aarch64-linux-gnu/15.2.0'`) is the correct, expected behavior when `DESTDIR` is missing — it is not a bug to work around, it's confirmation the safety mechanism is working. The fix is always to add `DESTDIR=$LFS`, never to `sudo` past the permission error.

## 5. Verify — this closes out the entire toolchain phase

```bash
find $LFS/usr/bin -name "*gcc*"
find $LFS/usr/bin -iname "*g++*"

echo 'int main(){}' > /tmp/dummy.c
$LFS/usr/bin/aarch64-linux-gnu-gcc /tmp/dummy.c -o /tmp/dummy
readelf -l /tmp/dummy | grep ': /lib'
```

The last line should show `/lib/ld-linux-aarch64.so.1` — confirming the compiler produces real, correctly-linked aarch64 executables against the target dynamic linker.

**Once this all checks out, the cross-toolchain (Packages 00–07) is fully complete.** Every package after this compiles using this now-finished compiler instead of any bootstrap tool.

## Commit checkpoint

1. `docs: add gcc pass 2 build notes, missing --target flag root cause, DESTDIR safety note`
2. `build: gcc 15.2.0 pass 2 compiled with explicit AS_FOR_TARGET/LD_FOR_TARGET, --target=$LFS_TGT`
3. `install: gcc pass 2 installed via DESTDIR=$LFS, cross-compiled dummy binary verified aarch64`
4. `milestone: cross-toolchain phase complete`

## Next

Toolchain phase is done — move to the temporary tools / chroot-entry section of the build order file (M4 and the rest of the temp-tools packages that were deferred from `04-glibc.md`).
