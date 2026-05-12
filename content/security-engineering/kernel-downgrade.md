---
date: '2026-05-12T18:57:00+08:00'
draft: false
title: 'Kernel Downgrade For Research'
showtoc: true
tags: ["llm", "rag", "ai", "tutorial"]
---

## TL;DR

Reproducing real-world vulnerabilities requires running the exact kernel version where the bug existed, not a patched one. This post walks new security engineers through building an isolated VirtualBox-on-Ubuntu home lab, creating Ubuntu guest VMs, and safely downgrading the guest kernel to a target version using three distinct methods, all without touching the host OS. Every technique is scoped to the VM boundary; your host stays clean.

---

## The "So What?": Why This Matters Even at Day One

The first mistake new security engineers make is trying to learn exploitation on their daily-driver machine. The second mistake is using a cloud VM they can't fully control. A local hypervisor lab gives you:

- **Snapshot checkpoints:** roll back a destroyed environment in seconds
- **Kernel-level isolation:** the host kernel is never touched
- **Network segmentation:** vulnerable VMs never touch the internet
- **Reproducibility:** the exact kernel version, every time

Understanding how to pin and downgrade a kernel version is not just a lab trick. It is the foundational skill behind every CVE reproduction, every Proof-of-Concept (PoC) validation, and every regression test a security team runs. If you can't reproduce the vulnerable state reliably, you cannot understand the vulnerability deeply.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        HOST OS (Ubuntu 24.04)                   │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                   VirtualBox Hypervisor                  │  │
│   │                                                          │  │
│   │  ┌─────────────────────┐   ┌─────────────────────────┐  │  │
│   │  │   VM: Attacker Box  │   │   VM: Vulnerable Target │  │  │
│   │  │   Ubuntu 24.04      │   │   Ubuntu 20.04          │  │  │
│   │  │   Kernel: latest    │   │   Kernel: 5.4.x (pinned)│  │  │
│   │  │                     │   │                         │  │  │
│   │  │  Tools: metasploit, │   │  Deliberately patched   │  │  │
│   │  │  pwndbg, gdb        │   │  DOWN to CVE range      │  │  │
│   │  └──────────┬──────────┘   └────────────┬────────────┘  │  │
│   │             │  Host-Only Network (vboxnet0)│             │  │
│   │             └──────────────┬──────────────┘             │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                │                                 │
│              ┌─────────────────┴──────────────────┐             │
│              │   No internet route for vuln VM     │             │
│              │   (Internal Network adapter only)   │             │
│              └────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

> **Critical design principle:** The vulnerable VM has **no bridged or NAT adapter**. It lives exclusively on a host-only or internal VirtualBox network segment. A VM running a known-vulnerable kernel with internet access is a security incident waiting to happen.

---

## Part 1: Setting Up VirtualBox on Your Ubuntu Host

### Step 1.1: System Prerequisites

Before installing VirtualBox, ensure your host CPU supports hardware virtualization. Intel calls this VT-x; AMD calls it AMD-V. Most machines manufactured after 2010 support it, but BIOS/UEFI may have it disabled.

```bash
# Check for virtualization support
grep -E --color '(vmx|svm)' /proc/cpuinfo | head -5

# If output is empty, enable VT-x/AMD-V in your BIOS settings
# Intel: look for "Intel Virtualization Technology"
# AMD:   look for "SVM Mode" or "AMD-V"
```

Also verify your host kernel supports KVM (VirtualBox uses its own hypervisor, but this is a good sanity check):

```bash
lscpu | grep Virtualization
```

### Step 1.2: Install VirtualBox from the Oracle Repository

The Ubuntu universe repository ships VirtualBox, but it is often several versions behind. Use the Oracle-signed repository to get the latest stable release with full USB 3.0 and NVME support.

```bash
# Install prerequisites
sudo apt update && sudo apt install -y \
    curl \
    wget \
    gnupg2 \
    software-properties-common \
    apt-transport-https

# Import the Oracle VirtualBox signing key
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc \
  | sudo gpg --dearmor --yes \
  --output /usr/share/keyrings/oracle-virtualbox-2016.gpg

# Add the VirtualBox repository (adjust noble/jammy to match your Ubuntu release)
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] \
  https://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" \
  | sudo tee /etc/apt/sources.list.d/virtualbox.list

# Install VirtualBox
sudo apt update && sudo apt install -y virtualbox-7.1

# Add your user to the vboxusers group
sudo usermod -aG vboxusers $USER

# Apply group membership without logout (for current session)
newgrp vboxusers
```

### Step 1.3: Install the VirtualBox Extension Pack

The Extension Pack enables USB 2.0/3.0 passthrough, NVMe, and RDP. It must match the exact VirtualBox version.

```bash
# Get the installed VirtualBox version
VBOX_VERSION=$(vboxmanage --version | cut -d'r' -f1)
echo "VirtualBox version: $VBOX_VERSION"

# Download the matching Extension Pack
wget "https://download.virtualbox.org/virtualbox/${VBOX_VERSION}/Oracle_VirtualBox_Extension_Pack-${VBOX_VERSION}.vbox-extpack"

# Install it (accept the PUEL license when prompted)
sudo vboxmanage extpack install \
  "Oracle_VirtualBox_Extension_Pack-${VBOX_VERSION}.vbox-extpack" \
  --replace
```

### Step 1.4: Configure the Host-Only Network Adapter

This is the network segment your VMs will use to talk to each other and to the host, without any internet access.

```bash
# Create a host-only network (creates vboxnet0)
sudo vboxmanage hostonlyif create

# Assign an IP range to it
sudo vboxmanage hostonlyif ipconfig vboxnet0 \
  --ip 192.168.56.1 \
  --netmask 255.255.255.0

# Disable the DHCP server on this interface so you assign IPs statically
sudo vboxmanage dhcpserver remove --ifname vboxnet0 2>/dev/null || true
```

Verify:

```bash
vboxmanage list hostonlyifs
# Should show vboxnet0 with IP 192.168.56.1
```

---

## Part 2: Creating the Ubuntu Guest VMs

You will create **two VMs**: a "clean baseline" that can be cloned, and a "vulnerable target" derived from that baseline.

### Step 2.1: Download an Ubuntu ISO

Download an Ubuntu Server ISO. For vulnerability research targeting older kernel ranges, Ubuntu 20.04 LTS is particularly useful; its 5.4.x kernel series covers a broad range of historically significant CVEs.

```bash
# Create a directory for ISOs
mkdir -p ~/VirtualBox/ISOs && cd ~/VirtualBox/ISOs

# Ubuntu 20.04.6 LTS (Focal Fossa) Server
wget https://releases.ubuntu.com/20.04/ubuntu-20.04.6-live-server-amd64.iso

# Verify the SHA256 checksum against Ubuntu's published hash
# Check https://releases.ubuntu.com/20.04/SHA256SUMS for the expected hash
sha256sum ubuntu-20.04.6-live-server-amd64.iso
```

> **Why Ubuntu 20.04?** Its default kernel (5.4.x HWE) sits in a range that covers CVEs like DirtyPipe (CVE-2022-0847, fixed in 5.16.11/5.15.25/5.10.102), and the broader OverlayFS privilege escalation family. Ubuntu 24.04 with kernel 6.8.x has fewer reproducible older CVEs.

### Step 2.2: Create the VM via CLI (VBoxManage)

Using `vboxmanage` instead of the GUI builds muscle memory for automation and scripting later.

```bash
# Create the VM
vboxmanage createvm \
  --name "Ubuntu-20.04-Baseline" \
  --ostype Ubuntu_64 \
  --register

# Set system resources
vboxmanage modifyvm "Ubuntu-20.04-Baseline" \
  --memory 4096 \
  --cpus 2 \
  --vram 16 \
  --graphicscontroller vmsvga \
  --boot1 dvd \
  --boot2 disk \
  --boot3 none \
  --boot4 none

# Create and attach a 40GB virtual disk
vboxmanage createhd \
  --filename "$HOME/VirtualBox VMs/Ubuntu-20.04-Baseline/Ubuntu-20.04-Baseline.vdi" \
  --size 40960 \
  --format VDI

# Add a SATA controller
vboxmanage storagectl "Ubuntu-20.04-Baseline" \
  --name "SATA Controller" \
  --add sata \
  --controller IntelAHCI

# Attach the virtual disk
vboxmanage storageattach "Ubuntu-20.04-Baseline" \
  --storagectl "SATA Controller" \
  --port 0 \
  --device 0 \
  --type hdd \
  --medium "$HOME/VirtualBox VMs/Ubuntu-20.04-Baseline/Ubuntu-20.04-Baseline.vdi"

# Attach the ISO as a virtual DVD
vboxmanage storageattach "Ubuntu-20.04-Baseline" \
  --storagectl "SATA Controller" \
  --port 1 \
  --device 0 \
  --type dvddrive \
  --medium "$HOME/VirtualBox/ISOs/ubuntu-20.04.6-live-server-amd64.iso"

# Add a Host-Only network adapter (for internal lab communication)
vboxmanage modifyvm "Ubuntu-20.04-Baseline" \
  --nic1 hostonly \
  --hostonlyadapter1 vboxnet0

# Enable the serial console (useful for headless debugging)
vboxmanage modifyvm "Ubuntu-20.04-Baseline" \
  --uart1 0x3F8 4 \
  --uartmode1 disconnected

# Start the VM for installation
vboxmanage startvm "Ubuntu-20.04-Baseline" --type gui
```

### Step 2.3: Ubuntu Server Installation Notes

During the Ubuntu installer:

- **Storage:** Use the entire disk (no LVM unless you specifically want it)
- **Profile:** Create a non-root user (e.g., `labuser`)
- **SSH:** Enable OpenSSH Server; you will use SSH from the host for all shell access
- **Software:** No snaps, no additional packages at install time; keep it minimal

After installation, eject the ISO and configure a static IP matching the host-only network:

```bash
# Inside the VM, after first boot
# Edit netplan config (file name may differ, check /etc/netplan/)
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
      nameservers:
        addresses: []        # No DNS — this VM has no internet
      routes: []             # No default gateway — intentionally isolated
```

```bash
sudo netplan apply
```

From your **host**, verify connectivity:

```bash
ssh labuser@192.168.56.10
```

### Step 2.4: Take the "Golden Snapshot" Before Any Kernel Work

**This step is non-negotiable.** Every kernel operation you perform later can brick the VM's boot chain. A snapshot taken here costs you nothing and saves you hours.

```bash
# On the host, take a snapshot of the clean baseline
vboxmanage snapshot "Ubuntu-20.04-Baseline" take "clean-install-baseline" \
  --description "Fresh Ubuntu 20.04.6 install, no kernel modifications, static IP 192.168.56.10"

# List snapshots to confirm
vboxmanage snapshot "Ubuntu-20.04-Baseline" list
```

To restore to this snapshot at any time:

```bash
# Shut down the VM first
vboxmanage controlvm "Ubuntu-20.04-Baseline" poweroff

# Restore
vboxmanage snapshot "Ubuntu-20.04-Baseline" restore "clean-install-baseline"

# Restart
vboxmanage startvm "Ubuntu-20.04-Baseline" --type gui
```

---

## Part 3: Kernel Downgrade (Three Methods, One Goal)

All three methods below are executed **inside the VM**. The host kernel is never involved. Before attempting any of them, take a snapshot labeled with the target kernel version so you have a clean restoration point.

### Understanding What You're Working With First

```bash
# Inside the VM — always know your current state before changing it
uname -r
# Example output: 5.4.0-182-generic

# List all installed kernels
dpkg --list | grep linux-image

# See what kernels GRUB knows about
grep -E "submenu|menuentry" /boot/grub/grub.cfg | grep -v "^#" | grep -v echo
```

---

### Method 1: APT Package Manager (Install a Specific Kernel from Ubuntu Repositories)

**Best for:** Kernels within the same Ubuntu release's supported range (e.g., staying within the 5.4.x HWE series for Ubuntu 20.04).

Ubuntu's repositories retain multiple kernel versions. This method uses `apt` to install a specific kernel package by its exact version string.

```bash
# Step 1: Find the exact kernel version strings available in apt
apt-cache showpkg linux-image-generic | grep -A5 "Versions:"

# More targeted — search for specific minor versions
apt-cache search linux-image | grep "5.4.0-" | sort

# Step 2: Install the target kernel and its headers
# Example: downgrading to 5.4.0-150-generic
sudo apt install -y \
  linux-image-5.4.0-150-generic \
  linux-headers-5.4.0-150-generic \
  linux-modules-5.4.0-150-generic \
  linux-modules-extra-5.4.0-150-generic
```

> **Why install headers too?** Kernel exploit PoCs almost always compile kernel modules or use eBPF programs that require kernel headers matching the running kernel. Missing headers = broken exploit compilation.

```bash
# Step 3: Configure GRUB to boot the old kernel by default
sudo nano /etc/default/grub
```

Change these two lines:

```bash
# /etc/default/grub — changes to make
GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.4.0-150-generic"
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
```

```bash
# Step 4: Regenerate GRUB config
sudo update-grub

# Verify the entry exists in the generated config
grep "5.4.0-150" /boot/grub/grub.cfg
```

```bash
# Step 5: Pin the kernel to prevent apt from upgrading it automatically
sudo apt-mark hold \
  linux-image-5.4.0-150-generic \
  linux-headers-5.4.0-150-generic

# Confirm the hold
apt-mark showhold
```

```bash
# Step 6: Reboot and verify
sudo reboot
# After reconnecting via SSH:
uname -r
# Expected: 5.4.0-150-generic
```

**Rollback:** Restore the snapshot you took before this step. Or, if the VM still boots, change `GRUB_DEFAULT` back to `0` (first entry = latest kernel) and `sudo update-grub && sudo reboot`.

---

### Method 2: Ubuntu Mainline Kernel PPA (Any Upstream Kernel Version)

**Best for:** Reproducing CVEs that require a very specific upstream kernel version (e.g., 5.8.0, 5.15.0) that was never packaged in Ubuntu's official repos. Many PoCs specify exact upstream versions like `v5.8.0` or `v5.17.0`.

The Ubuntu Kernel Team maintains a mainline build PPA at `kernel.ubuntu.com/mainline` with prebuilt `.deb` packages for nearly every upstream kernel version since 4.0.

```bash
# Step 1: Identify the exact upstream version you need
# Example: CVE-2022-0847 (DirtyPipe) affects < 5.16.11, < 5.15.25, < 5.10.102
# We'll install 5.15.0 to reproduce it

TARGET_KERNEL="v5.15.0"
ARCH="amd64"
BASE_URL="https://kernel.ubuntu.com/mainline/${TARGET_KERNEL}/${ARCH}"

# Step 2: Download the required .deb packages
mkdir -p /tmp/kernel-${TARGET_KERNEL} && cd /tmp/kernel-${TARGET_KERNEL}

# You need three packages: headers-all, headers-arch-specific, and image
wget "${BASE_URL}/linux-headers-5.15.0-051500_5.15.0-051500.202110312130_all.deb"
wget "${BASE_URL}/linux-headers-5.15.0-051500-generic_5.15.0-051500.202110312130_amd64.deb"
wget "${BASE_URL}/linux-image-unsigned-5.15.0-051500-generic_5.15.0-051500.202110312130_amd64.deb"
wget "${BASE_URL}/linux-modules-5.15.0-051500-generic_5.15.0-051500.202110312130_amd64.deb"
```

> **Finding the exact filenames:** Browse `https://kernel.ubuntu.com/mainline/v5.15.0/amd64/` in a browser to see the exact `.deb` filenames for your target version before downloading.

```bash
# Step 3: Verify package integrity with checksums
# Each mainline directory contains a CHECKSUMS file
wget "${BASE_URL}/CHECKSUMS"
sha256sum --check CHECKSUMS 2>&1 | grep -v "FAILED open"
```

```bash
# Step 4: Install via dpkg (not apt — these are not in a repo)
sudo dpkg -i /tmp/kernel-${TARGET_KERNEL}/*.deb

# If dependency errors occur:
sudo apt --fix-broken install -y
```

```bash
# Step 5: Set GRUB default and reboot (same as Method 1)
sudo nano /etc/default/grub
# GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.15.0-051500-generic"
# GRUB_TIMEOUT_STYLE=menu
# GRUB_TIMEOUT=10

sudo update-grub
sudo reboot
```

```bash
# Step 6: After reboot, verify
uname -r
# Expected: 5.15.0-051500-generic
```

**Uninstalling a mainline kernel** (if you want to clean up):

```bash
sudo dpkg --purge \
  linux-image-unsigned-5.15.0-051500-generic \
  linux-headers-5.15.0-051500-generic \
  linux-headers-5.15.0-051500 \
  linux-modules-5.15.0-051500-generic

sudo update-grub
```

---

### Method 3: GRUB Boot Entry Override (Zero Package Changes, One-Time Boot)

**Best for:** Quick one-off tests where you want to boot an already-installed older kernel without changing the default permanently. Useful when you have multiple kernel versions installed and want to switch between them for different experiments.

This method requires no package installation. It uses GRUB's interactive menu or a one-time boot override.

**Option A: Interactive GRUB Menu (easiest for beginners)**

```bash
# Step 1: Configure GRUB to always show the menu on boot
sudo nano /etc/default/grub

# Set:
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
# Comment out or remove: GRUB_HIDDEN_TIMEOUT

sudo update-grub
sudo reboot
```

On next boot, the GRUB menu appears. Select **Advanced options for Ubuntu**, then choose the specific kernel version you want. This does not change the default; next boot returns to the configured default.

**Option B: `grub-reboot` (One-Time Programmatic Kernel Selection)**

This is the most powerful approach for automation. `grub-reboot` sets the next boot entry to a specific kernel **exactly once**, then reverts to the default.

```bash
# Step 1: List GRUB menu entries with their index numbers
sudo grep -E "submenu|menuentry" /boot/grub/grub.cfg \
  | grep -v "^#" \
  | grep -v "echo" \
  | cat -n
```

Example output:
```
     1  submenu 'Advanced options for Ubuntu' $menuentry_id_option ...
     2      menuentry 'Ubuntu, with Linux 5.4.0-182-generic' ...
     3      menuentry 'Ubuntu, with Linux 5.4.0-182-generic (recovery mode)' ...
     4      menuentry 'Ubuntu, with Linux 5.4.0-150-generic' ...
     5      menuentry 'Ubuntu, with Linux 5.4.0-150-generic (recovery mode)' ...
```

```bash
# Step 2: Set next boot to "Advanced options" submenu (index 1), then 3rd entry (index 2, 0-based)
# Format is "submenu_index>entry_index"
sudo grub-reboot "1>2"

# Reboot — it will boot 5.4.0-150-generic exactly once, then revert
sudo reboot
```

```bash
# After reconnecting — verify it booted into the right kernel
uname -r
```

**Option C: Using the `menuentry` title string directly (more readable)**

```bash
# Use the exact menu entry title from grub.cfg
sudo grub-reboot "Advanced options for Ubuntu>Ubuntu, with Linux 5.4.0-150-generic"
sudo reboot
```

---

## Part 4: Network Isolation (Keeping the Vulnerable VM Contained)

A VM running a deliberately vulnerable kernel **must not have internet access**. This is not optional.

```bash
# On the host — verify the VM has only the host-only adapter
vboxmanage showvminfo "Ubuntu-20.04-Baseline" | grep NIC

# Expected output should show:
# NIC 1: MAC: ..., Attachment: Host-only Interface 'vboxnet0', ...
# NIC 2: disabled
# (all other NICs: disabled)
```

If you need to transfer tools or PoC code into the isolated VM, use `scp` over the host-only network from your host machine:

```bash
# From the host — copy a file into the vulnerable VM
scp /path/to/exploit-poc.c labuser@192.168.56.10:/home/labuser/

# Or use SSH port forwarding for tool installation without internet access
# (proxy through the host's internet connection if truly needed — one-time, then disconnect)
```

**Firewall rules inside the VM** as a secondary control:

```bash
# Inside the vulnerable VM — block all outbound traffic as defense-in-depth
sudo ufw default deny outgoing
sudo ufw default deny incoming
# Allow only SSH from the host-only network
sudo ufw allow from 192.168.56.0/24 to any port 22
sudo ufw enable
```

---

## Part 5: The Snapshot Discipline (Your Most Important Habit)

The entire value of a home lab is destroyed if you don't use snapshots. Develop this pattern from day one:

```bash
# Snapshot naming convention: [vm-name]-[kernel-version]-[state]
vboxmanage snapshot "Ubuntu-20.04-Baseline" take \
  "5.4.0-150-generic-pre-exploit-attempt" \
  --description "Kernel 5.4.0-150, labuser created, network isolated, SSH working"
```

**Recommended snapshot tree for a CVE reproduction session:**

```
clean-install-baseline
    └── kernel-5.4.0-150-installed
            ├── pre-exploit-env-setup
            │       ├── exploit-failed-attempt-1
            │       ├── exploit-working
            │       └── patched-kernel-applied
            └── debugging-symbols-added
```

```bash
# List all snapshots
vboxmanage snapshot "Ubuntu-20.04-Baseline" list --machinereadable

# Delete a snapshot you no longer need (frees disk space)
vboxmanage snapshot "Ubuntu-20.04-Baseline" delete "exploit-failed-attempt-1"
```

---

## Part 6: Cloning the Baseline (Building Your Vulnerable Target VM)

Once the baseline is stable, clone it to create an expendable "target" VM. Never do kernel experiments on your only copy.

```bash
# Create a full clone from the baseline snapshot
vboxmanage clonevm "Ubuntu-20.04-Baseline" \
  --name "Target-CVE-2022-0847" \
  --snapshot "clean-install-baseline" \
  --options keepallmacs \
  --register

# Assign a different static IP to the clone
# (SSH into the clone and update /etc/netplan/)
# Use 192.168.56.20 for the clone
```

---

## Common Failure Modes and How to Recover

| Symptom | Likely Cause | Recovery |
|--------|--------------|----------|
| VM hangs at GRUB, no menu | `GRUB_TIMEOUT_STYLE=hidden` still set | Boot from ISO in rescue mode, chroot, fix grub |
| SSH fails after reboot | New kernel missing network module | Restore snapshot |
| `dpkg -i` fails with dependency error | Missing lib packages | Run `sudo apt --fix-broken install` |
| GRUB doesn't show new kernel | `update-grub` not run | Boot current kernel, run `sudo update-grub` |
| Kernel panics on boot | Incompatible mainline kernel for your virtual hardware | Restore snapshot, try adjacent kernel version |
| `apt-mark hold` not working | Using `linux-image-generic` meta-package | Hold the specific versioned package, not the meta-package |

**Emergency recovery using GRUB rescue mode** (when you can't boot at all):

At the GRUB prompt, manually load a working kernel:

```
grub> set root=(hd0,1)
grub> linux /boot/vmlinuz-5.4.0-182-generic root=/dev/sda1 ro
grub> initrd /boot/initrd.img-5.4.0-182-generic
grub> boot
```

Once booted, restore the GRUB configuration and take stock of what went wrong before trying again.

---

## Strategic Recommendations for Going Deeper

This lab setup is the foundation. Where you take it next depends on what you're trying to learn:

1. **Structured CVE Reproduction Practice**
   Follow [Linux Kernel CVE Tracker](https://www.cve.org/CVERecord?id=CVE-2022-0847) entries for DirtyPipe, OverlayFS, and Dirty COW. Each has published PoC code. Reproduce them in your lab against the exact affected kernel, then apply the patch and confirm the fix works.

2. **Kernel Debugging with KGDB/GDB**
   Enable `KGDB` in your VM kernel boot parameters to attach a GDB debugger from the host over a virtual serial port. Essential for understanding exploit mechanics at the instruction level.

   ```bash
   # Add to GRUB_CMDLINE_LINUX in /etc/default/grub inside the VM
   GRUB_CMDLINE_LINUX="kgdboc=ttyS0,115200 kgdbwait"
   ```

3. **Automated Lab Provisioning**
   Once you've done this manually a few times, automate it with a shell script or Vagrant + VirtualBox provider. The [Vagrant documentation](https://developer.hashicorp.com/vagrant/docs) covers this well.

4. **Expand to Container Escape Research**
   Many kernel CVEs are relevant in container contexts (e.g., namespace escapes). Add a Docker installation inside your vulnerable VM and practice container escape techniques after mastering basic kernel exploitation.

---

## Reference Material

| Resource | Why It's Valuable |
|----------|------------------|
| [Ubuntu Mainline Kernel PPA](https://kernel.ubuntu.com/mainline/) | Prebuilt `.deb` packages for every upstream kernel version |
| [Linux Kernel CVE Tracker](https://www.kernel.org/doc/html/latest/security/index.html) | Official kernel security documentation |
| [VirtualBox CLI Reference (vboxmanage)](https://www.virtualbox.org/manual/ch08.html) | Complete reference for all VBoxManage commands used in this guide |
| [GRUB2 Manual (GNU Project)](https://www.gnu.org/software/grub/manual/grub/grub.html) | Authoritative GRUB configuration reference |
| [NVD: CVE-2022-0847 (DirtyPipe)](https://nvd.nist.gov/vuln/detail/CVE-2022-0847) | Classic example of a kernel write primitive in 5.8-5.16 |
| [NVD: CVE-2016-5195 (Dirty COW)](https://nvd.nist.gov/vuln/detail/CVE-2016-5195) | Race condition in `mmap`, a foundational exploit to reproduce |
| [Linux Kernel Archives](https://www.kernel.org/) | Source tarballs for every kernel version, useful for patch diffing |
| [pwndbg: GDB plugin for exploit dev](https://github.com/pwndbg/pwndbg) | Essential GDB extension for kernel and userland exploit debugging |
| [liveoverflow: YouTube, binary exploitation](https://www.youtube.com/c/LiveOverflow) | High-quality video walkthroughs of kernel exploit techniques |
| [How to Compile a Custom Kernel (Ubuntu Wiki)](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel) | When you need to build with specific `CONFIG_` options enabled for a PoC |
