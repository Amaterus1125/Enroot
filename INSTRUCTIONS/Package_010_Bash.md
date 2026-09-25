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
## Known issue — `unrecognized options: --without-man` warning (harmless, not an error)
 
**Symptom:**
```
configure: WARNING: unrecognized options: --without-man
```
Bash's `configure` script doesn't have that specific flag — not every package uses the same flag names for disabling manpages. It's silently ignored, and configure still completes all the way through (`creating Makefile`, `creating config.h`, ... `executing stamp-h commands`). Configure succeeded despite the warning.
 
Also confirms `using libncursesw` in the configure tail — Ncurses from the previous step is being found correctly by this build.
 
Manpages won't build by default in this minimal `make install` anyway, since `make install-doc` (or similar) is never run separately — the outcome is the same with or without the flag.
 
## 2. Build
