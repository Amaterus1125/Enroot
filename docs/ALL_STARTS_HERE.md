# Getting Started — What You Need Before Touching Any of This

This project involves building a Linux system completely from source code — not installing a distro, but actually compiling the pieces that make one up. That means the setup requirements are a bit different from normal software projects. This doc explains what you need and why, in plain terms.

## 1. A real laptop or desktop — not just a VM on a low-power machine

You need an actual computer you can install Linux on, not just a phone or tablet. Why this matters:

- Compiling software (especially GCC, which shows up a lot in this project) uses real CPU and RAM. It's not like running a pre-built app — the computer is doing genuine, heavy work turning source code into working programs.
- Some steps take a long time (an hour or more isn't unusual for the bigger packages). This is normal and expected, not a sign something's broken.

**Minimum realistic specs:**
- 4GB RAM (works, but expect some steps to be slow — see the swap note below)
- 8GB+ RAM if you can manage it — noticeably smoother
- At least **50GB of free disk space** — the build environment alone uses a 40GB image file, plus space for the OS itself and downloaded source code

If your machine only has 4GB RAM, it's still doable — just add a swapfile (a chunk of disk space the system can use as overflow memory) before starting the bigger builds. This gets mentioned again later in the actual package build files where it matters most.

## 2. A Linux operating system installed on that machine

This is the operating system your terminal commands actually run on — the "host." You're not building Linux *in* Windows or macOS directly; you need Linux itself as the base you're working from.

**What's being used for this project: Debian, installed via the "netinst" image.**

### What "netinst" means, in plain terms

A normal Linux install disk usually comes packed with thousands of programs already included, so it can be several gigabytes to download. **netinst** ("network install") flips that around — it's a small image (under a gigabyte) containing just enough to get a basic system running, and it downloads everything else you actually choose to install directly from the internet during setup.

**Why this is a good fit for this specific project**, not just a random choice:
- It starts you with a genuinely minimal system — no extra desktop bloat, no pre-installed programs you didn't ask for. That mindset (start minimal, add only what you need, understand what's actually installed and why) matches exactly what this whole project is about.
- Smaller, faster initial download.
- You get to choose exactly what goes on the system during install, rather than un-installing things you never wanted later.

### How to get it

1. Go to the official Debian download page: **https://www.debian.org/distrib/**
2. Look for the "Small CDs or USB sticks" section — this is the netinst image.
3. Download the version matching your computer (almost certainly **amd64** unless you specifically know otherwise).
4. Write it to a USB stick using a tool like **Rufus** (Windows), **balenaEtcher** (Windows/Mac/Linux), or `dd` (if you're already on Linux/Mac).
5. Boot your computer from that USB stick (usually by pressing F12, F2, Esc, or Del right when the computer turns on — varies by manufacturer — to get a boot menu).
6. Follow the Debian installer prompts. When asked about "software selection" partway through, you can safely uncheck desktop environment options if you want to keep things minimal (a "standard system utilities" + SSH server selection is enough to get going) — though installing a desktop is fine too if you'd rather have a normal graphical environment to work from.

## 3. Basic tools to install right after Debian is set up

Once Debian is installed and you can open a terminal, install these — they're the bare minimum needed before any of the actual project steps will work:

```bash
sudo apt update
sudo apt install -y build-essential wget git vim
```

**What each of these actually is:**
- `build-essential` — a bundle that includes `gcc`, `make`, and other core tools needed to compile software from source. Without this, none of the build steps in this project will work at all.
- `wget` — downloads files from the internet directly via the terminal (used constantly to fetch source code tarballs).
- `git` — version control, used to track changes to this project's own files and commit your progress.
- `vim` — a text editor that works inside the terminal (any terminal text editor works — `nano` is a friendlier beginner alternative if `vim` feels unfamiliar: `sudo apt install -y nano`).

## 4. Setting up folders for this project

Once the tools above are installed, create a clean, dedicated space to work in — keeping this project's files separate from anything else on your system:

```bash
mkdir -pv ~/enroot-project
cd ~/enroot-project

# This is where the actual repo (docs, scripts, configs) will live
git init
```

If you're cloning this repo instead of starting from scratch:
```bash
git clone <your-repo-url> ~/enroot-project
cd ~/enroot-project
```

From here, the rest of the project's setup (the loopback build image, environment variables, source downloads) is covered in `00-environment-setup.md` — that's the next file to read once your Debian system and these basic tools are in place.

## Quick checklist before moving on

- [ ] Laptop/desktop with at least 4GB RAM and 50GB free disk space
- [ ] Debian installed (via netinst) and booting normally
- [ ] Can open a terminal and run commands
- [ ] `build-essential`, `wget`, `git` installed and confirmed working (`gcc --version`, `wget --version`, `git --version` all print something without an error)
- [ ] A dedicated project folder created

Once every box is checked, you're ready for `00-environment-setup.md`.
