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
