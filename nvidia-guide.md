# NVIDIA Driver Recovery Guide for Linux
## The Full Story of How `sudo apt upgrade -y` Broke Everything

---

## Table of Contents
1. [What Happened — The Root Cause](#1-what-happened--the-root-cause)
2. [The Full Timeline — Every Driver We Installed, Removed, and Reinstalled](#2-the-full-timeline)
3. [Why MOK Blocked Us and What It Actually Is](#3-why-mok-blocked-us)
4. [The Fix That Actually Worked](#4-the-fix-that-actually-worked)
5. [Deep Dive: NVIDIA Drivers on Linux](#5-deep-dive-nvidia-drivers-on-linux)
6. [Deep Dive: Secure Boot, MOK, and DKMS](#6-deep-dive-secure-boot-mok-and-dkms)
7. [Deep Dive: Kernel Modules and How They Load](#7-deep-dive-kernel-modules)
8. [Command Reference and Investigation Playbook](#8-command-reference-and-investigation-playbook)
9. [The Lid Close Gotcha](#9-the-lid-close-gotcha)
10. [Future-Proof Checklist](#10-future-proof-checklist)

---

## 1. What Happened — The Root Cause

### The Trigger
```bash
sudo apt upgrade -y
```

This single command did all of the following:
1. **Upgraded the Linux kernel** from `6.14.0-27` to `6.17.0-19`
2. **Removed Xorg packages** (or upgraded them in a way that broke the display server)
3. **Installed/upgraded to Wayland** as the default display server
4. **Left the NVIDIA driver orphaned** — the old driver was built for kernel `6.14.0-27`, but the system now booted into `6.17.0-19`

### Why the Monitor Went Black

```
BEFORE upgrade:
  Kernel 6.14.0-27 ←→ nvidia-driver-575-open (DKMS module built for this kernel) ←→ HDMI output ✅

AFTER upgrade:
  Kernel 6.17.0-19 ←→ ??? (no nvidia module for this kernel) ←→ HDMI output ❌
```

The NVIDIA driver is a **kernel module** (a `.ko` file). Kernel modules are built for a SPECIFIC kernel version. When the kernel upgrades, the module must be rebuilt. DKMS (Dynamic Kernel Module Support) is supposed to do this automatically, but it failed — likely because:
- The upgrade was interrupted or the DKMS build wasn't triggered
- Secure Boot blocked the newly built module because it wasn't signed
- The transition from Xorg to Wayland compounded the issue

### The Wayland Factor

Wayland + NVIDIA (especially older cards like GTX 1650 Ti) = pain. Here's why:

```
Xorg (X11):
  - Been around for 30+ years
  - NVIDIA has excellent proprietary support
  - Mature, stable, boring (good!)
  - Uses nvidia's own driver for rendering

Wayland:
  - Modern replacement for Xorg
  - Requires nvidia-drm kernel module with modeset=1
  - Needs GBM (Generic Buffer Manager) backend
  - GTX 1650 Ti (Turing architecture) has limited Wayland support
  - Many features broken or buggy on older NVIDIA GPUs
```

When the upgrade switched the display server to Wayland AND the NVIDIA kernel module wasn't loaded, there was literally nothing to drive the HDMI output.

---

## 2. The Full Timeline

Here's every driver action we took, in order, and why:

### Attempt 1: Install nvidia-driver-535 (WRONG MOVE)
```bash
sudo apt install xorg xserver-xorg-video-nvidia
# ERROR: xserver-xorg-video-nvidia doesn't exist as a package

sudo apt install nvidia-driver-535
# This DOWNGRADED from 575 to 535
# It REMOVED all nvidia-575 packages
# Downloaded 325MB of 535 driver packages
```

**Why this was wrong:**
- We didn't check what was already installed first (`apt list --installed | grep nvidia`)
- The system had `nvidia-driver-575-open` — going to 535 was a downgrade
- The 535 driver uses proprietary (closed) kernel modules, not the open ones
- Triggered a **Secure Boot / MOK enrollment** prompt

### Attempt 2: MOK Enrollment (COULDN'T COMPLETE)

The 535 driver install triggered this:
```
UEFI Secure Boot requires additional configuration to work with third-party drivers.
A new Machine-Owner Key (MOK) has been generated.
You must choose a password and confirm after reboot.
```

**Why we couldn't complete it:**
- MOK enrollment happens at the **UEFI firmware level** (before Linux boots)
- It displays on the **physical screen**
- The laptop screen is **broken**
- The HDMI monitor **might** work at UEFI level (UEFI uses basic framebuffer, not NVIDIA drivers)
- But we never got to test this properly because the system **powered off** during DKMS build

### Attempt 3: Remove 535, install 575-open (INTERRUPTED)
```bash
sudo apt remove nvidia-driver-535
sudo apt install nvidia-driver-575-open
```

**What went wrong:**
- `dpkg` was locked — another `apt` process (PID 2486) was running (likely `unattended-upgrades`)
- After killing the lock, `dpkg` was in an interrupted state
- Running `sudo dpkg --configure -a` to fix it triggered a DKMS build
- **Mid-build, the system powered off** ("The system will power off now!")
- This was caused by a **lid close event** — Ubuntu's default behavior is to power off when the lid closes

### Attempt 4: dpkg --configure -a again (BUILT BUT BLOCKED)
```bash
sudo dpkg --configure -a
```

This time it completed:
```
Building for 6.17.0-19-generic ... Done.
nvidia.ko.zst: Installing to /lib/modules/6.17.0-19-generic/updates/dkms/
nvidia-modeset.ko.zst: Installing ...
nvidia-drm.ko.zst: Installing ...
nvidia-uvm.ko.zst: Installing ...
```

But `nvidia-smi` still failed! Why?
```
Secure Boot is ON + DKMS-built modules are NOT signed = kernel refuses to load them
```

Even though the modules were built and installed to `/lib/modules/`, the kernel's Secure Boot enforcement said: "I don't recognize the signature on these modules. Rejected."

### Attempt 5: Pre-built, pre-signed kernel modules (THE FIX)
```bash
sudo apt install linux-modules-nvidia-575-open-generic-hwe-24.04
```

This installed `linux-modules-nvidia-580-open-6.17.0-19-generic` — a module that is:
- **Pre-compiled** by Canonical for kernel 6.17.0-19
- **Pre-signed** with Canonical's Secure Boot key
- **Already trusted** by UEFI — no MOK enrollment needed

```bash
sudo modprobe nvidia
nvidia-smi
# SUCCESS! GTX 1650 Ti detected, driver 580.126.09
```

---

## 3. Why MOK Blocked Us

### What is Secure Boot?

```
Power On
  └── UEFI Firmware loads
      └── Checks: "Is the bootloader signed by a trusted key?"
          ├── YES → Boot continues
          └── NO  → Boot blocked

Linux Kernel loads
  └── Tries to load kernel modules (like nvidia.ko)
      └── Checks: "Is this module signed by a trusted key?"
          ├── YES → Module loads
          └── NO  → Module rejected (this is what happened to us!)
```

Secure Boot maintains a **database of trusted keys**. By default, it trusts:
- Microsoft's keys (for Windows)
- Canonical's keys (for Ubuntu, since Ubuntu pays Microsoft to sign their bootloader)
- Your motherboard manufacturer's keys

It does NOT trust:
- Random DKMS-compiled modules (like nvidia built from source on your machine)
- Third-party drivers that aren't signed by a trusted authority

### What is MOK (Machine Owner Key)?

MOK is the workaround. It lets you say: "I, the machine owner, trust this specific key."

```
Normal Secure Boot trust chain:
  UEFI → Microsoft → Canonical → Ubuntu kernel → signed modules ✅

With MOK:
  UEFI → Microsoft → Canonical → Ubuntu kernel → DKMS module signed with YOUR key ✅
                                                    ↑
                                            MOK says "trust this key too"
```

### The MOK Enrollment Process

```
Step 1 (during driver install, via SSH):
  - DKMS generates a new keypair
  - Signs the nvidia module with the private key
  - Asks you to set a MOK password
  - Stores the public key for enrollment

Step 2 (during reboot, on PHYSICAL SCREEN):
  ┌─────────────────────────────────┐
  │     Perform MOK management      │
  │                                 │
  │  → Enroll MOK                   │  ← Select this
  │    Change Secure Boot state     │
  │    Boot normally                │
  └─────────────────────────────────┘

  Then: Enter the password you set in Step 1
  Then: Confirm enrollment
  Then: Reboot

Step 3 (after reboot):
  - UEFI now trusts your MOK key
  - Kernel loads the DKMS nvidia module
  - nvidia-smi works
```

### Why We Couldn't Do MOK

1. **Laptop screen is broken** — can't see the MOK enrollment screen
2. **HDMI might work at UEFI level** — but we never confirmed because:
3. **System kept powering off** — lid close events interrupted everything
4. **Even if HDMI worked at UEFI**, we'd need a keyboard connected to the laptop to navigate the MOK menu (USB keyboard, not SSH)

### The Bypass: Pre-signed Modules

Instead of fighting MOK, we used modules that are **already signed by Canonical**:

```
DKMS modules (what we tried first):
  Source code → compiled on YOUR machine → signed with YOUR key → needs MOK ❌

Pre-built modules (what worked):
  Source code → compiled by CANONICAL → signed with CANONICAL's key → already trusted ✅
```

The package `linux-modules-nvidia-580-open-6.17.0-19-generic` contains `.ko` files that Canonical compiled and signed in their build infrastructure. Since Ubuntu's Secure Boot chain already trusts Canonical's key, these modules load without any MOK enrollment.

---

## 4. The Fix That Actually Worked

### The Golden Commands
```bash
# 1. Install pre-built, pre-signed NVIDIA modules for your kernel
sudo apt install linux-modules-nvidia-575-open-generic-hwe-24.04
# (This pulled in linux-modules-nvidia-580-open-6.17.0-19-generic)

# 2. Remove DKMS versions (they conflict and don't work with Secure Boot anyway)
sudo apt remove nvidia-dkms-575-open nvidia-dkms-580-open nvidia-dkms-535

# 3. Load the module
sudo modprobe nvidia

# 4. Verify
nvidia-smi
```

### Why This Specific Package?

```
Package naming convention:
linux-modules-nvidia-{VERSION}-{TYPE}-{KERNEL}

linux-modules-nvidia-580-open-6.17.0-19-generic
│                    │   │    │
│                    │   │    └── For kernel 6.17.0-19-generic
│                    │   └── Open source kernel modules (not proprietary)
│                    └── NVIDIA driver version 580
└── Pre-built kernel modules (not DKMS)

vs.

nvidia-dkms-580-open
│         │   │
│         │   └── Open source
│         └── Driver version 580
└── DKMS = compiled on YOUR machine from source
```

The `linux-modules-nvidia-*` packages are the way to go when Secure Boot is enabled.

---

## 5. Deep Dive: NVIDIA Drivers on Linux

### Driver Types

There are THREE types of NVIDIA drivers on Ubuntu:

```
1. PROPRIETARY (closed-source)
   Package: nvidia-driver-580
   Kernel module: nvidia-dkms-580
   - NVIDIA's official closed-source driver
   - Best compatibility with older GPUs
   - Requires MOK enrollment with Secure Boot

2. OPEN (open-source kernel modules, closed userspace)
   Package: nvidia-driver-580-open
   Kernel module: nvidia-dkms-580-open
   - Open source kernel modules (nvidia-open-gpu-kernel-modules on GitHub)
   - Closed source userspace libraries (libGL, etc.)
   - Better for newer GPUs (Turing+, i.e., GTX 1650 Ti and newer)
   - Still requires MOK with Secure Boot IF using DKMS

3. PRE-BUILT (Canonical's signed packages)
   Package: linux-modules-nvidia-580-open-6.17.0-19-generic
   - Same open source modules, but compiled and signed by Canonical
   - Works with Secure Boot WITHOUT MOK
   - Only available for Ubuntu's official kernel versions
   - THIS IS WHAT YOU SHOULD USE
```

### Driver Version History (relevant to your system)

```
535  - Older production branch (what we accidentally installed)
550  - Available but we skipped it
570  - Available as pre-built modules
575  - What you originally had (nvidia-driver-575-open)
580  - What's running now (via pre-built modules)
590  - Available (newest at time of writing)
```

### How to Check Your Current Driver

```bash
# Method 1: nvidia-smi (most reliable)
nvidia-smi
# Shows: Driver Version: 580.126.09, CUDA Version: 13.0

# Method 2: Check loaded kernel module
cat /proc/driver/nvidia/version
# Shows: NVRM version: NVIDIA UNIX Open Kernel Module 580.126.09

# Method 3: Check what packages are installed
dpkg -l | grep nvidia-driver
apt list --installed 2>/dev/null | grep nvidia

# Method 4: Check DKMS status
dkms status
# Shows which DKMS modules are built for which kernels

# Method 5: Check loaded modules
lsmod | grep nvidia
# Should show: nvidia, nvidia_modeset, nvidia_drm, nvidia_uvm
```

### How NVIDIA Drivers Load

```
Boot sequence with NVIDIA:

1. UEFI/BIOS
2. GRUB
3. Linux kernel loads
4. initramfs loads early modules
5. systemd starts
6. nvidia kernel modules load:
   nvidia.ko         → Core GPU driver
   nvidia-modeset.ko → Kernel Mode Setting (display output)
   nvidia-drm.ko     → Direct Rendering Manager integration
   nvidia-uvm.ko     → Unified Virtual Memory (CUDA)
   nvidia-peermem.ko → GPU peer memory (multi-GPU)
7. GDM starts
8. Xorg or Wayland starts
9. Xorg loads nvidia_drv.so (userspace driver)
10. Display appears on monitor
```

---

## 6. Deep Dive: Secure Boot, MOK, and DKMS

### DKMS Explained

DKMS = Dynamic Kernel Module Support. It's a framework that automatically rebuilds kernel modules when you install a new kernel.

```bash
# How DKMS works:
sudo apt install nvidia-dkms-580-open
# 1. Stores source code in /usr/src/nvidia-580.126.09/
# 2. Compiles it for your current kernel
# 3. Installs .ko files to /lib/modules/$(uname -r)/updates/dkms/
# 4. Registers with DKMS so it auto-rebuilds on kernel upgrade

# Check DKMS status
dkms status
# Example output:
# nvidia/580.126.09, 6.17.0-19-generic, x86_64: installed

# Manually trigger a rebuild
sudo dkms install nvidia/580.126.09 -k 6.17.0-19-generic

# Remove a DKMS module
sudo dkms remove nvidia/580.126.09 --all
```

### Why DKMS + Secure Boot = Pain

```
WITHOUT Secure Boot:
  DKMS compiles module → installs it → kernel loads it → done ✅

WITH Secure Boot:
  DKMS compiles module → signs with auto-generated key → installs it
  → kernel checks signature → "who is this key?" → REJECTED ❌
  → need MOK enrollment → reboot → physical screen interaction → THEN it works
```

The auto-generated key is stored in:
```
/var/lib/shim-signed/mok/MOK.priv  (private key)
/var/lib/shim-signed/mok/MOK.der   (public key/certificate)
```

### Checking Secure Boot Status

```bash
# Is Secure Boot enabled?
mokutil --sb-state
# Output: SecureBoot enabled  (or disabled)

# List enrolled MOK keys
mokutil --list-enrolled

# Check if there are pending MOK enrollments
mokutil --list-new
# If this shows keys, you need to reboot and complete enrollment

# Disable Secure Boot from Linux (still needs reboot confirmation on screen)
sudo mokutil --disable-secure-boot
# Sets a password, then on reboot you confirm on the MOK screen

# Test if a module would be allowed to load
mokutil --test-key /var/lib/shim-signed/mok/MOK.der
# "is not enrolled" = Secure Boot will block DKMS modules
# "is already enrolled" = DKMS modules will load fine
```

### The Three Ways to Handle Secure Boot + NVIDIA

```
Option 1: Use pre-built modules (RECOMMENDED for your setup)
  sudo apt install linux-modules-nvidia-580-open-$(uname -r)
  - No MOK needed
  - Signed by Canonical
  - Only works for Ubuntu's official kernels

Option 2: Enroll MOK (if you have a working screen)
  - During driver install, set MOK password
  - Reboot, complete enrollment on physical screen
  - DKMS modules will load

Option 3: Disable Secure Boot in BIOS
  - F2 during boot → Security → Secure Boot → Disable
  - All modules load regardless of signing
  - Slightly less secure but eliminates all signing issues
```

---

## 7. Deep Dive: Kernel Modules

### What is a Kernel Module?

A kernel module is a piece of code that can be loaded into the running kernel to extend its functionality — like a plugin.

```bash
# List all loaded modules
lsmod

# Search for nvidia modules
lsmod | grep nvidia
# nvidia_uvm          1503232  0
# nvidia_drm            94208  0
# nvidia_modeset      1355776  1 nvidia_drm
# nvidia              6283264  2 nvidia_uvm,nvidia_modeset

# Load a module
sudo modprobe nvidia

# Unload a module
sudo modprobe -r nvidia_drm

# Get info about a module
modinfo nvidia
# Shows: version, description, author, filename, depends, etc.

# Check where the module file lives
modinfo -F filename nvidia
# /lib/modules/6.17.0-19-generic/updates/dkms/nvidia.ko.zst
#                                │              │          │
#                                │              │          └── Compressed with zstd
#                                │              └── The actual module file
#                                └── "updates/dkms" = installed by DKMS (overrides built-in)
```

### Module Dependencies

Modules depend on each other:

```
nvidia.ko (core)
  └── nvidia_modeset.ko (display/KMS)
      └── nvidia_drm.ko (DRM integration)
  └── nvidia_uvm.ko (CUDA unified memory)
  └── nvidia_peermem.ko (multi-GPU)
```

`modprobe` handles dependencies automatically:
```bash
sudo modprobe nvidia_drm
# Automatically loads nvidia and nvidia_modeset first
```

### Module Parameters

```bash
# nvidia-drm has a critical parameter:
modinfo nvidia-drm | grep parm
# modeset: Register as a DRM driver (bool)

# To set it permanently:
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf

# Then rebuild initramfs so it takes effect on boot:
sudo update-initramfs -u
```

### initramfs — Why It Matters

initramfs is a mini filesystem loaded into RAM during boot, before the real root filesystem is mounted. It contains essential kernel modules and scripts.

```bash
# Rebuild initramfs (do this after any module changes)
sudo update-initramfs -u
# -u = update the initramfs for the current kernel

# Rebuild for a specific kernel
sudo update-initramfs -u -k 6.17.0-19-generic

# Rebuild for ALL kernels
sudo update-initramfs -u -k all
```

---

## 8. Command Reference and Investigation Playbook

### Step-by-Step: "My Monitor Stopped Working After an Upgrade"

```bash
# ============================================
# PHASE 1: INVESTIGATE
# ============================================

# What kernel are we running?
uname -r
# Expected: something like 6.17.0-19-generic

# Is the NVIDIA driver loaded?
nvidia-smi
# If this fails, the driver isn't loaded

# Is the nvidia module loaded?
lsmod | grep nvidia
# If empty, module isn't loaded

# Why isn't it loaded? Try loading it:
sudo modprobe nvidia
# If this fails, check dmesg for why:
sudo dmesg | grep -i nvidia
sudo dmesg | grep -i "module verification"
# Look for: "module verification failed: signature and/or required key missing"
# This means Secure Boot is blocking it

# Is Secure Boot on?
mokutil --sb-state

# What nvidia packages are installed?
dpkg -l | grep nvidia | grep -E "^ii"

# What kernel modules exist for nvidia?
find /lib/modules/$(uname -r) -name "nvidia*.ko*"

# Is DKMS involved?
dkms status

# What display server is running?
loginctl show-session $(loginctl | grep seat | awk '{print $1}') -p Type 2>/dev/null
# x11 = Xorg, wayland = Wayland

# What GPUs does the system see?
lspci | grep -i nvidia
# 01:00.0 3D controller: NVIDIA Corporation TU117M [GeForce GTX 1650 Ti Mobile]

# What display outputs exist?
ls /sys/class/drm/
# Look for card*-HDMI-A-1, card*-DP-1, card*-eDP-1

# Are any displays connected?
for card in /sys/class/drm/card*-*/status; do echo "$card: $(cat $card)"; done

# ============================================
# PHASE 2: FIX
# ============================================

# OPTION A: Secure Boot ON, use pre-signed modules (RECOMMENDED)
# Find available pre-built modules for your kernel:
apt search linux-modules-nvidia | grep "$(uname -r)"

# Install the best match (prefer -open variant, highest version):
sudo apt install linux-modules-nvidia-580-open-$(uname -r)

# Remove DKMS versions:
sudo apt remove $(dpkg -l | grep nvidia-dkms | awk '{print $2}')

# Load the module:
sudo modprobe nvidia

# Verify:
nvidia-smi


# OPTION B: Secure Boot OFF, use DKMS (simpler but less secure)
# First disable secure boot in BIOS (F2 → Security → Secure Boot → Disable)
# Then:
sudo apt install nvidia-driver-580-open
sudo reboot
# nvidia-smi should work after reboot


# OPTION C: Stay on an older kernel that already has working modules
# List available kernels:
dpkg -l | grep linux-image

# Set GRUB to boot an older kernel:
sudo nano /etc/default/grub
# Change: GRUB_DEFAULT=0
# To:     GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.14.0-27-generic"
sudo update-grub
sudo reboot

# ============================================
# PHASE 3: CONFIGURE DISPLAY
# ============================================

# Disable Wayland (use Xorg instead)
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf

# Enable nvidia-drm modeset (needed for display output)
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf
sudo update-initramfs -u

# Restart display manager
sudo systemctl restart gdm3

# Or reboot:
sudo reboot
```

### Quick Reference: Package Commands

```bash
# ============================================
# APT — THE PACKAGE MANAGER
# ============================================

sudo apt update                    # Refresh package lists (always do this first!)
sudo apt upgrade -y                # Upgrade ALL packages (THIS IS WHAT BROKE US)
sudo apt install <package>         # Install a package
sudo apt remove <package>          # Remove a package (keep config files)
sudo apt purge <package>           # Remove package AND config files
sudo apt autoremove                # Remove orphaned dependencies
sudo apt list --installed          # List installed packages
sudo apt search <keyword>          # Search for packages
sudo apt show <package>            # Show package details
apt policy <package>               # Show available versions
sudo apt-mark hold <package>       # PREVENT a package from being upgraded
sudo apt-mark unhold <package>     # Allow upgrades again

# ============================================
# DPKG — LOWER LEVEL PACKAGE TOOL
# ============================================

dpkg -l | grep <keyword>           # Search installed packages
dpkg -L <package>                  # List files installed by a package
dpkg -S /path/to/file              # Which package owns this file?
sudo dpkg --configure -a           # Fix interrupted installs
sudo dpkg -i package.deb           # Install a .deb file manually

# ============================================
# PREVENTING FUTURE BREAKAGE
# ============================================

# Hold nvidia packages so apt upgrade doesn't touch them:
sudo apt-mark hold nvidia-driver-580-open
sudo apt-mark hold linux-modules-nvidia-580-open-$(uname -r)

# List held packages:
apt-mark showhold

# When you're ready to upgrade nvidia manually:
sudo apt-mark unhold nvidia-driver-580-open
sudo apt update
sudo apt install nvidia-driver-580-open
```

### Quick Reference: NVIDIA Commands

```bash
# ============================================
# NVIDIA DIAGNOSTIC COMMANDS
# ============================================

nvidia-smi                         # GPU status, driver version, memory usage, processes
nvidia-smi -q                      # Detailed GPU info
nvidia-smi -L                      # List GPUs
nvidia-smi dmon                    # Monitor GPU in real-time
nvidia-smi --query-gpu=gpu_name,driver_version,memory.total --format=csv
                                   # Custom query

cat /proc/driver/nvidia/version    # Driver version from kernel module

# ============================================
# DISPLAY/MONITOR COMMANDS (via SSH)
# ============================================

# Need to set DISPLAY when running over SSH:
export DISPLAY=:0
export XAUTHORITY=/run/user/$(id -u)/gdm/Xauthority

xrandr                             # List displays and resolutions
xrandr --output HDMI-1-0 --auto    # Enable HDMI output
xrandr --output eDP-1 --off        # Disable internal display

# Without Xorg (using kernel-level display info):
for f in /sys/class/drm/card*-*/status; do echo "$f: $(cat $f)"; done
cat /sys/class/drm/card2-HDMI-A-1/modes    # Supported resolutions
```

### Quick Reference: Flags and Their Quirks

```bash
# ============================================
# APT FLAGS YOU SHOULD KNOW
# ============================================

apt install <pkg>
  -y                    # Auto-yes to prompts (use carefully!)
  --reinstall           # Force reinstall even if already installed
  --no-install-recommends  # Skip recommended (but not required) packages
  --fix-broken          # Try to fix broken dependencies (-f also works)

apt upgrade
  -y                    # THE FLAG THAT BROKE US — auto-confirms everything
  --dry-run             # Show what WOULD happen without doing it (ALWAYS DO THIS FIRST)

apt remove
  --purge               # Also remove config files (same as apt purge)
  --autoremove          # Also remove orphaned dependencies

# SAFER UPGRADE (do this instead of apt upgrade -y):
sudo apt update
sudo apt upgrade --dry-run         # Review changes first!
sudo apt upgrade                   # Without -y, so you can review the prompt

# ============================================
# MODPROBE FLAGS
# ============================================

modprobe <module>                  # Load a module (with dependencies)
modprobe -r <module>               # Remove/unload a module
modprobe -v <module>               # Verbose — show what it's doing
modprobe --dry-run <module>        # Test without actually loading
modprobe -n -v <module>            # Dry run + verbose (see what WOULD happen)

# ============================================
# DKMS FLAGS
# ============================================

dkms status                        # Show all DKMS modules and their state
dkms install <module>/<version>    # Build and install for current kernel
dkms install <module>/<version> -k <kernel-version>  # For specific kernel
dkms build <module>/<version>      # Build without installing
dkms remove <module>/<version> --all  # Remove from ALL kernels

# ============================================
# MOKUTIL FLAGS
# ============================================

mokutil --sb-state                 # Is Secure Boot on or off?
mokutil --list-enrolled            # Show enrolled MOK keys
mokutil --list-new                 # Show pending enrollments
mokutil --test-key <key.der>       # Test if a key is enrolled
mokutil --disable-secure-boot      # Queue Secure Boot disable (needs reboot + screen)
mokutil --enable-secure-boot       # Queue Secure Boot enable

# ============================================
# DPKG LOCK ISSUES
# ============================================

# "Could not get lock /var/lib/dpkg/lock-frontend"
# First, check what's holding it:
ps aux | grep apt
ps aux | grep dpkg

# If it's unattended-upgrades, wait or kill it:
sudo kill <PID>

# If the process is gone but lock remains:
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo rm /var/cache/apt/archives/lock
sudo dpkg --configure -a

# NOTE: Only remove locks if you're SURE no apt/dpkg process is running!
```

---

## 9. The Lid Close Gotcha

During our troubleshooting, the laptop **powered off twice** during DKMS builds because the lid was closed. This is Ubuntu's default behavior.

### The Problem
```
Default lid close behavior:
  Lid closes → systemd-logind detects it → powers off the system

This is TERRIBLE for a headless server with a broken screen because:
  - The lid is always closed
  - Any reboot = system comes up → lid is closed → powers off again
  - You lose SSH access
```

### The Fix
```bash
# Edit logind.conf
sudo nano /etc/systemd/logind.conf

# Find and change these lines (uncomment them):
HandleLidSwitch=ignore           # When on AC power: do nothing
HandleLidSwitchExternalPower=ignore  # When on external power: do nothing
HandleLidSwitchDocked=ignore     # When docked (external monitor): do nothing

# Apply without reboot:
sudo systemctl restart systemd-logind

# Verify:
loginctl show | grep HandleLid
```

### Understanding All Lid Options
```
suspend    = Sleep (default for HandleLidSwitch)
hibernate  = Write RAM to disk, power off
poweroff   = Shut down (THIS IS WHAT KEPT KILLING US)
ignore     = Do absolutely nothing (WHAT WE WANT)
lock       = Lock the screen but stay running
```

---

## 10. Future-Proof Checklist

### Before Running `apt upgrade`

```bash
# 1. Check what will be upgraded
sudo apt update
sudo apt upgrade --dry-run | grep nvidia
sudo apt upgrade --dry-run | grep linux-image
sudo apt upgrade --dry-run | grep linux-modules

# 2. If nvidia or kernel packages will be upgraded, prepare:
#    - Make sure you can access BIOS (for MOK enrollment or Secure Boot changes)
#    - Or use pre-built modules (see below)
#    - Hold packages if you're not ready: sudo apt-mark hold nvidia-driver-580-open

# 3. Run the upgrade
sudo apt upgrade    # WITHOUT -y so you can review

# 4. After upgrade, verify:
nvidia-smi
# If it fails, follow the investigation playbook above
```

### The Ideal NVIDIA Setup for Your Laptop

```bash
# Use pre-built modules (avoids all Secure Boot/MOK issues):
sudo apt install nvidia-driver-580-open
sudo apt install linux-modules-nvidia-580-open-generic-hwe-24.04

# Remove DKMS versions:
sudo apt remove nvidia-dkms-580-open

# Disable Wayland:
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf

# Set nvidia-drm modeset:
echo "options nvidia-drm modeset=1" | sudo tee /etc/modprobe.d/nvidia-drm.conf

# Disable lid switch (headless laptop):
sudo sed -i 's/#HandleLidSwitch=suspend/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/#HandleLidSwitchExternalPower=suspend/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf

# Hold nvidia packages to prevent surprise upgrades:
sudo apt-mark hold nvidia-driver-580-open
sudo apt-mark hold linux-modules-nvidia-580-open-$(uname -r)

# Rebuild initramfs:
sudo update-initramfs -u

# Reboot:
sudo reboot
```

### Emergency Recovery Commands (Save These!)

```bash
# If nvidia-smi fails after a kernel upgrade:
sudo apt install linux-modules-nvidia-580-open-$(uname -r)
sudo modprobe nvidia
nvidia-smi

# If display is black but SSH works:
export DISPLAY=:0
export XAUTHORITY=/run/user/$(id -u)/gdm/Xauthority
xrandr --output HDMI-1-0 --auto

# If system keeps powering off:
sudo sed -i 's/HandleLidSwitch=.*/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind

# If dpkg is broken:
sudo dpkg --configure -a

# If apt is locked:
ps aux | grep -E "apt|dpkg"
sudo kill <PID>
sudo dpkg --configure -a

# If everything is broken, boot into an older kernel:
# (Need physical access or GRUB configured for serial/network console)
sudo grub-reboot "Advanced options for Ubuntu>Ubuntu, with Linux 6.14.0-27-generic"
sudo reboot
```

---

## Summary: What We Learned

| Lesson | Detail |
|--------|--------|
| `apt upgrade -y` is dangerous | It auto-confirms kernel upgrades that can break NVIDIA |
| Always `--dry-run` first | See what will change before committing |
| DKMS + Secure Boot = MOK pain | DKMS modules need signing; MOK enrollment needs a screen |
| Pre-built modules bypass MOK | `linux-modules-nvidia-*` packages are signed by Canonical |
| Lid close = power off | Set `HandleLidSwitch=ignore` for headless laptops |
| Hold critical packages | `apt-mark hold` prevents accidental upgrades |
| NVIDIA + Wayland = unstable | Use Xorg (`WaylandEnable=false`) for GTX 1650 Ti |
| Keep an older kernel around | `dpkg -l \| grep linux-image` — don't remove working kernels |

---

*Written after the great NVIDIA driver saga of March 20, 2026*
*Total time debugging: ~2 hours across SSH*
*Root cause: `sudo apt upgrade -y` + Secure Boot + broken laptop screen + lid close events*
