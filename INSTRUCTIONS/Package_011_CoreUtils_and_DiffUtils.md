# Package 11 — Coreutils & Diffutils
 
**Status: ✅ COMPLETE** (Diffutils' fix also changes your permanent build environment — see the end of this file.)
 
Grouped together because Diffutils' fix directly affects how every package *after* it gets built, so it's worth reading through both before moving on.
 
---
 
## Part A — Coreutils (Section 1, step 15)
 
**Why:** `ls`, `cat`, `cp`, and the rest of the everyday command-line utilities — needed almost immediately by later build steps, not just the final system.
 
### 1. Fetch source, check for the required patch
 
```bash
cd $LFS/sources
wget https://ftp.gnu.org/gnu/coreutils/coreutils-9.10.tar.xz
tar xf coreutils-9.10.tar.xz
cd coreutils-9.10
```
