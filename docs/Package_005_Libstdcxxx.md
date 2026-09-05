# Package 005 — Libstdc++

**Why:** Reuses the same `gcc-15.2.0` source tree already extracted for GCC Pass 1 — this is just a second, separate build pass against it now that Glibc exists for it to link against.

**Naming note:** the library's real, official name genuinely is `libstdc++` (GNU's C++ standard library, distinct from LLVM's `libc++`). The build directory below is named `libstdcxx-build` (not `libstdc++-build`) purely as a shell-safety convention — `+` isn't strictly dangerous in bash, but can misbehave in some regex/glob/Makefile-substitution contexts, so LFS/CLFS convention spells it `cxx` in directory and script names only. The actual source folder inside GCC (`libstdc++-v3`) and the compiled library itself keep the real `++` name.

## 1. Out-of-tree build directory

```bash
cd $LFS/sources
mkdir -v libstdcxx-build
cd libstdcxx-build
```

## 2. Configure — against the `libstdc++-v3` subdirectory, NOT the top-level GCC configure

```bash
../gcc-15.2.0/libstdc++-v3/configure \
    --host=$LFS_TGT \
    --build=$(../gcc-15.2.0/config.guess) \
    --prefix=/usr \
    --libdir=/usr/lib \
    --disable-multilib \
    --disable-nls \
    --disable-libstdcxx-pch \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/15.2.0
```

## Why this must target `libstdc++-v3/configure` specifically

Running `../gcc-15.2.0/configure` (the top-level GCC configure) here fails with a **"C compiler cannot create executables"** error, and it's easy to misread as a broken sysroot. The real cause: top-level configure tries to set up *every* GCC subcomponent it finds, including `libcody` and `c++tools` — internal helpers for C++20 modules support, unrelated to what's actually needed. Those two specifically require a working **native** C++ compiler capable of producing runnable executables, which doesn't exist yet in a cross-bootstrap. It's a chicken-and-egg trap, not a real problem with the sysroot.

The fix is configuring directly against the subdirectory instead, which never touches `libcody`/`c++tools` at all — this sidesteps the problem entirely rather than fixing anything. That's why the command above points at `../gcc-15.2.0/libstdc++-v3/configure`, not `../gcc-15.2.0/configure`.

If you've already run the top-level configure once and hit this, clear the build directory before retrying:
```bash
cd $LFS/sources/libstdcxx-build
rm -rf * .* 2>/dev/null    # safe — only clears this empty build dir, not the GCC source next door
```

**Watch for this exact line in the configure tail:**
```
checking whether the C compiler works... yes
```
If it says `no`, something upstream (Glibc's install) isn't actually in place yet — go back and confirm Glibc actually installed (see the `find` checks in `04-glibc.md`) before proceeding.

## 3. Build and install

```bash
make -j$(nproc)
make DESTDIR=$LFS install
```

## 4. Known issue — libraries landing in `lib64` instead of `lib`

Even with `--disable-multilib` set, GCC's `aarch64-linux-gnu` target configuration can still default to `lib64` for this specific decision — a known quirk, not a mistake. The `--libdir=/usr/lib` flag included above should prevent this on a fresh build, but if you're troubleshooting an existing one and find libraries under `lib64`:

```bash
ls -la $LFS/usr/lib64/libstdc++.so*    # confirm they're really there
cd $LFS/usr/lib
ln -sv ../lib64/libstdc++.so.6* .
ln -sv ../lib64/libstdc++.so .
```

This makes both paths resolve to the same files, so anything later searching the "normal" `/usr/lib` path finds it too, without duplicating anything.

## 5. Verify

```bash
find $LFS/usr/lib -name "libstdc++.so*"
```

Should resolve directly under `$LFS/usr/lib` (via `--libdir` or the symlink fallback above).

## Commit checkpoint

1. `docs: add libstdc++ build notes, libstdc++-v3 subdirectory fix, lib64 symlink note`
2. `build: libstdc++ compiled against gcc-15.2.0/libstdc++-v3, aarch64-linux-gnu`
3. `install: libstdc++.so verified under $LFS/usr/lib`

## Next

`packages/06-binutils-pass2.md`
