# Package 09 — Ncurses
 
**Status: ✅ COMPLETE — built, installed, and verified. `libncursesw` confirmed in place, 1861 terminfo entries installed.**
 
**Why:** Section 1, step 13 — provides the terminal-handling library that later packages (starting with Bash, next up) link against for interactive terminal I/O.
 
## Source

Confirmed real URL for `ncurses-6.6`, verified via multiple sources: `https://ftp.gnu.org/pub/gnu/ncurses/ncurses-6.6.tar.gz`.
 
## 1. Fetch, extract
 
```bash
cd $LFS/sources
wget https://ftp.gnu.org/pub/gnu/ncurses/ncurses-6.6.tar.gz
tar xf ncurses-6.6.tar.gz
cd ncurses-6.6
```
## 2. Build a native `tic` first
 
Ncurses needs a small trick for cross-compiles — it has to build a native (host) copy of its own code-generation tool first, since it can't run the ARM64 one during its own build:
```bash
mkdir -v build-aux
pushd build-aux
    ../configure
    make -C include
    make -C progs tic
popd
```
## 3. Configure the real cross-compiled build (no manpages)
 
Manpage generation disabled on every package going forward, per standing preference:
 
```bash
mkdir -v build
cd build
 
../configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../config.guess) \
    --without-manpages \
    --without-progs \
    --without-normal \
    --with-shared \
    --with-cxx-shared \
    --without-debug \
    --without-ada \
    --disable-stripping \
    --with-build-cc=gcc \
    --with-build-cpp=cpp
```
 
## 4. Build
 
```bash
make -j2
```
 
## 5. Install
 
Note the `TIC_PATH` on install — this points the install step at the **native** `tic` binary built in `build-aux/`, so it can generate terminfo data during install without trying to run the ARM64 binary on the x86_64 host:
 
```bash
make DESTDIR=$LFS TIC_PATH=$(pwd)/../build-aux/progs/tic install
```
 
## Verify
 
```bash
find $LFS/usr/lib -iname "*ncurses*"
find $LFS/usr/bin -name "tic"
```
 
Confirmed output: `libncursesw.so.6`, `libncursesw.so.6.6`, and the dev symlink `libncursesw.so` all landed in `$LFS/usr/lib`. 1861 terminfo entries installed. Clean finish.
## Known note — built as `libncursesw`, not `libncurses`
 
This is the wide-character/unicode variant, not the plain name. Some later packages expect the plain `libncurses` name — if that comes up, it's a simple symlink fix, not a real problem. Flagging now so it's not confusing if it surfaces in a future package's configure step.
 
## Commit checkpoint
 
1. `build: ncurses-6.6 native build-aux tic built for host`
2. `build: ncurses-6.6 cross-compiled for aarch64-linux-gnu (manpages disabled)`
3. `install: ncurses installed to $LFS/usr/lib via TIC_PATH, verified libncursesw + terminfo`
## Next
 
Move to Bash (Section 1, step 14) — confirm `m4` and `ncurses` are both installed before starting, since Bash's configure will detect and link against `libncursesw` from this step.
 
