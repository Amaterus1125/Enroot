# Package 10 — Bash
 
**Status: ✅ COMPLETE — built, installed, and verified as an aarch64 binary.**
 
**Why:** Section 1, step 14 — the shell for the target system, built directly after Ncurses since Bash's configure detects and links against the `libncursesw` just installed.
 
## 1. Fetch, extract, configure
 
```bash
cd $LFS/sources
wget https://ftp.gnu.org/gnu/bash/bash-5.3.tar.gz
tar xf bash-5.3.tar.gz
mkdir -v bash-build
cd bash-build
 
../bash-5.3/configure \
    --prefix=/usr \
    --host=$LFS_TGT \
    --build=$(../bash-5.3/support/config.guess) \
    --without-bash-malloc \
    --disable-nls \
    --without-man
```
