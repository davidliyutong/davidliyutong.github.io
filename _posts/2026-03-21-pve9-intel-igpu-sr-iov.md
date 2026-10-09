---
layout: post
title: Complete Guide to Intel i3-12100 iGPU SR-IOV on Proxmox VE 9
date: 2026-03-21 00:01:00
description: Split the UHD 730 iGPU of an i3-12100 into SR-IOV virtual functions for multiple VMs on Proxmox VE 9 (ZFS, systemd-boot) with i915-sriov-dkms.
tags: proxmox sr-iov gpu virtualization IT
categories: IT
toc:
  sidebar: left
---

This guide walks through enabling SR-IOV on an Alder Lake iGPU under Proxmox VE 9, so that a single integrated GPU can be shared by several VMs.

> - **Hardware**: Intel i3-12100 (Alder Lake, UHD 730 Xe iGPU)
> - **System**: Proxmox VE 9.x, ZFS root with systemd-boot
> - **Goal**: Use SR-IOV to split the iGPU into up to 7 virtual functions (VFs) and assign them to multiple VMs

---

## Disclaimer

- This setup relies on a community DKMS driver (`i915-sriov-dkms`) and is **not supported by Proxmox or Intel**.
- **Do not try this directly on a production host.** Validate it in a test environment first.
- Take a full backup before you start, especially of your VM configurations and important data.

---

## Check the Hardware and Environment

### Identify the CPU and iGPU

```bash
# Show the CPU model
lscpu | grep "Model name"
# Expected: 12th Gen Intel(R) Core(TM) i3-12100

# Show the iGPU PCI device
lspci | grep VGA
# Expected: 00:02.0 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730] (rev 0c)

# Note the iGPU's PCI address (usually 00:02.0); you will need it later
```

> **Why SR-IOV**: The i3-12100 is a 12th-gen Alder Lake part with a UHD 730 (Xe) iGPU. It does **not** support GVT-g, which only covers 5th through 10th gen, so **SR-IOV** is the only option.

### BIOS Settings

Enter the motherboard firmware setup and make sure the following options are set:

| Setting            | Value                    | Notes                                                                                                                                                                                             |
| ------------------ | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Intel VT-d (IOMMU) | **Enabled**              | The foundation for device passthrough                                                                                                                                                             |
| SR-IOV             | **Enabled**              | Some boards have a separate toggle                                                                                                                                                                |
| Internal Graphics  | **Enabled**              | Make sure the iGPU is not disabled                                                                                                                                                                |
| Primary Display    | **CPU Graphics or Auto** | Make sure the system can see the iGPU                                                                                                                                                             |
| Secure Boot        | **Disabled**             | Out-of-tree DKMS modules won't load with Secure Boot on (unless you sign them, see the upstream [Secure Boot guide](https://github.com/strongtz/i915-sriov-dkms/blob/master/docs/secure-boot.md)) |
| Above 4G Decoding  | **Enabled**              | Recommended                                                                                                                                                                                       |

> **Note**: Option names differ between vendors (ASUS, MSI, Gigabyte, and so on). Look under Advanced > System Agent or the chipset configuration.

### Check the PVE Version and Kernel

```bash
# Show the PVE version
pveversion -v

# Show the running kernel
uname -r
# PVE 9.0 shipped kernel 6.14.x-pve; PVE 9.1 made 6.17.x-pve the default
```

The kernel version matters twice in this guide: the kernel headers you install must match the running kernel, and each `i915-sriov-dkms` release only supports a specific kernel range.

### Determine the Bootloader (Important)

This guide assumes a ZFS root installed in UEFI mode with Secure Boot disabled. In that configuration PVE boots with **systemd-boot**, not GRUB. ZFS installs on legacy BIOS, or installed with Secure Boot enabled, use GRUB instead, so confirm which one you have:

```bash
# Method 1: check the EFI directory
ls /sys/firmware/efi
# If the directory exists, the system booted in UEFI mode

# Method 2: check the active EFI boot entry (the most reliable check on a running system)
efibootmgr -v
# systemd-boot: Boot0006* Linux Boot Manager    [...] File(\EFI\systemd\systemd-bootx64.efi)
# GRUB:         Boot0005* proxmox       [...] File(\EFI\proxmox\grubx64.efi)  (shimx64.efi with Secure Boot)

# Method 3: check the boot tool status
proxmox-boot-tool status
# Output that mentions "systemd-boot" (or similar) means systemd-boot

# Method 4: check whether the cmdline file exists
cat /etc/kernel/cmdline
# ZFS + UEFI (systemd-boot) uses this file instead of /etc/default/grub
# Typically: root=ZFS=rpool/ROOT/pve-1 boot=zfs
```

> **With systemd-boot, the kernel command line lives in `/etc/kernel/cmdline`. Don't edit `/etc/default/grub`.**
> Apply changes with `proxmox-boot-tool refresh`, not `update-grub`.
> If your host turns out to boot with GRUB, put the parameters in `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub` and run `update-grub` instead.

### Check Whether IOMMU Is Enabled

```bash
dmesg | grep -i -e DMAR -e IOMMU
# Messages like "DMAR: IOMMU enabled" mean the IOMMU is active
# If not, check VT-d in the BIOS; the kernel parameters added below also cover it
```

> On kernel 6.8 and newer, the Intel IOMMU is enabled by default, so it should already be on with PVE 9 kernels. `intel_iommu=on` is redundant there, but harmless, and the upstream driver docs list it anyway.

---

## Configure Repositories and Install the SR-IOV DKMS Module

### Configure the Proxmox Repositories (Important)

`proxmox-headers` and the other packages you need live in the **Proxmox repositories**, not in Debian's. If no usable Proxmox repository is configured, installing the kernel headers and the DKMS module will fail.

**Check the current sources**:

```bash
ls /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/*.list 2>/dev/null
cat /etc/apt/sources.list.d/*.sources 2>/dev/null
```

If nothing points at `download.proxmox.com`, add the repository yourself.

**Add the free no-subscription repository**. PVE 9 is based on Debian Trixie and uses the deb822 `.sources` format (apt on Trixie warns about the legacy one-line `.list` format):

```bash
cat > /etc/apt/sources.list.d/proxmox.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

**Disable the enterprise repositories (if you have no subscription)**. On PVE 9 they are `.sources` files, and an entry is disabled by adding `Enabled: no` to it:

```bash
# Check whether the enterprise sources exist
cat /etc/apt/sources.list.d/pve-enterprise.sources 2>/dev/null
cat /etc/apt/sources.list.d/ceph.sources 2>/dev/null

# Disable pve-enterprise (single entry) to avoid 401 errors from apt update
grep -q '^Enabled:' /etc/apt/sources.list.d/pve-enterprise.sources || echo 'Enabled: no' >> /etc/apt/sources.list.d/pve-enterprise.sources
```

If `ceph.sources` contains an entry pointing at `enterprise.proxmox.com`, add `Enabled: no` to that entry as well. You can also toggle both repositories in the web UI under the node's **Updates → Repositories** panel.

> **Why this step?**
>
> A fresh PVE install enables the enterprise repository, which requires a paid subscription. Without one, `apt update` fails with `401 Unauthorized`.
> Packages such as `proxmox-headers` and `proxmox-kernel` only exist in the Proxmox repositories, not in Debian's official ones.
> Without a Proxmox repository, installing the headers fails with `Unable to locate package`.

**Verify the repository configuration**:

```bash
apt update
# The output should contain a line like this, with no errors:
# Hit:X http://download.proxmox.com/debian/pve trixie InRelease
```

### Update the System

```bash
apt update && apt full-upgrade -y
reboot
```

After the reboot, check the kernel version:

```bash
uname -r
# You should now be on the newest kernel installed by the upgrade (6.14.x-pve on PVE 9.0, 6.17.x-pve on PVE 9.1)
```

### Install the Dependencies

```bash
# Build tools and kernel headers (the meta package tracks the default kernel series)
apt install dkms build-essential proxmox-default-headers sysfsutils -y
```

> `proxmox-default-headers` is a meta package that pulls in headers for the current default kernel series, and keeps doing so when that kernel is updated.
> DKMS can only build against headers that **exactly match the running kernel**, which is why you reboot into the new kernel first. To install the matching headers explicitly:
>
> ```bash
> # Exactly the running kernel (most reliable)
> apt install proxmox-headers-$(uname -r)
> # Or by kernel series, e.g. proxmox-headers-6.14 on PVE 9.0, proxmox-headers-6.17 on PVE 9.1
> apt install proxmox-headers-6.17
> ```

**Verify that the headers are installed**:

```bash
ls /usr/src/linux-headers-$(uname -r)
# The directory should exist and contain a Makefile and other files
```

> **Common error**: if `proxmox-headers` still can't be found, check that:
>
> 1. The Proxmox repository is configured (see "Configure the Proxmox Repositories" above)
> 2. `apt update` ran without errors
> 3. You rebooted into the newest kernel (the version from `uname -r` matches the headers available in the repository)

### Download and Install the DKMS Module

Get the latest `.deb` from the [i915-sriov-dkms releases](https://github.com/strongtz/i915-sriov-dkms/releases) page. Each release supports a specific kernel range (listed under "Required kernel" in the project README) and older kernels are served by older releases, so pick the release that covers the kernel shown by `uname -r`.

```bash
# Create a working directory
mkdir -p /opt/i915-sriov && cd /opt/i915-sriov

# Download a release (replace with the URL of the release that matches your kernel)
# The URL below is only an example; get the current one from GitHub
wget -O i915-sriov-dkms.deb "https://github.com/strongtz/i915-sriov-dkms/releases/download/2025.11.10/i915-sriov-dkms_2025.11.10_amd64.deb"

# Install the package
dpkg -i i915-sriov-dkms.deb
```

> **If the installation fails**, remove the old version first. Upstream's uninstall command is `dpkg -P i915-sriov-dkms`; if leftovers remain, delete them manually:
>
> ```bash
> rm -rf /var/lib/dkms/i915-sriov-dkms*
> rm -rf /usr/src/i915-sriov-dkms*
> # Then install again
> dpkg -i i915-sriov-dkms.deb
> ```

### Verify the DKMS Build

```bash
dkms status
# Expected output similar to:
# i915-sriov-dkms/2025.11.10, 6.14.x-x-pve, x86_64: installed
```

If the status is not `installed`, check the build log:

```bash
cat /var/lib/dkms/i915-sriov-dkms/*/build/make.log
```

---

## Set the Kernel Parameters (ZFS / systemd-boot)

### Edit the Kernel Command Line

> **On a ZFS root with systemd-boot, edit `/etc/kernel/cmdline`, not `/etc/default/grub`.**

```bash
# Back it up first
cp /etc/kernel/cmdline /etc/kernel/cmdline.bak

# Show the current contents
cat /etc/kernel/cmdline
# Typically: root=ZFS=rpool/ROOT/pve-1 boot=zfs
```

Edit the file:

```bash
nano /etc/kernel/cmdline
```

Append the following parameters to the end of the **same line**. The whole file must be a single line with no blank lines:

```text
root=ZFS=rpool/ROOT/pve-1 boot=zfs intel_iommu=on iommu=pt i915.enable_guc=3 i915.max_vfs=7 module_blacklist=xe
```

**Parameters**:

| Parameter             | Purpose                                                                     |
| --------------------- | --------------------------------------------------------------------------- |
| `intel_iommu=on`      | Enables the Intel IOMMU (VT-d)                                              |
| `iommu=pt`            | IOMMU passthrough mode; improves performance for devices not passed through |
| `i915.enable_guc=3`   | Enables GuC submission and HuC loading (required for SR-IOV)                |
| `i915.max_vfs=7`      | Sets the maximum number of virtual functions (up to 7)                      |
| `module_blacklist=xe` | Blocks the newer `xe` driver so that `i915` is used                         |

> **Remember**:
>
> - Everything must be on **one line**, with no line breaks
> - No blank lines; a single trailing newline at the end of the file is fine
> - Keep your existing `root=ZFS=...` part exactly as it is

### Refresh the Boot Configuration

```bash
# Update the initramfs
update-initramfs -u -k all

# Refresh the systemd-boot configuration (the key step on a ZFS root)
proxmox-boot-tool refresh
```

### Create the VFs at Boot with sysfsutils

First confirm the iGPU's PCI address:

```bash
lspci | grep VGA
# Usually 00:02.0
```

Have `sysfsutils` create the virtual functions at boot. Debian's `sysfsutils` reads `/etc/sysfs.conf` plus any `/etc/sysfs.d/*.conf`, so put the setting in its own file instead of overwriting `/etc/sysfs.conf`:

```bash
echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" > /etc/sysfs.d/i915-sriov.conf
```

> - If your iGPU is not at `00:02.0`, use the actual address.
> - Leave the package's default `/etc/sysfs.conf` in place: the `sysfsutils` systemd unit only runs if that file exists.
> - If you'd rather keep everything in `/etc/sysfs.conf`, append the line idempotently instead of overwriting the file: `grep -qs 'sriov_numvfs' /etc/sysfs.conf || echo "devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7" >> /etc/sysfs.conf`

### Reboot

```bash
reboot
```

---

## Verify That SR-IOV Works

### Check the Kernel Parameters

```bash
cat /proc/cmdline
# Should contain intel_iommu=on i915.enable_guc=3 i915.max_vfs=7 module_blacklist=xe
```

### Check That i915 Loaded

```bash
dmesg | grep i915
# Look for lines like these (illustrative; exact wording varies by kernel and driver version):
# i915 0000:00:02.0: Running on Alder Lake-S
# i915 0000:00:02.0: [drm] GT0: GUC: submission enabled
# i915 0000:00:02.0: [drm] GT0: HuC: authenticated
```

**What to look for**:

- `GUC: submission enabled`: GuC submission is active
- `HuC: authenticated`: HuC is authenticated
- No errors such as `Incompatible option enable_guc`

### Check the Virtual Functions

```bash
lspci | grep VGA
# Expected: 1 PF + 7 VFs (illustrative; device names may differ slightly)
# 00:02.0 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
# 00:02.1 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
# 00:02.2 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
# 00:02.3 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
# 00:02.4 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
# 00:02.5 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
# 00:02.6 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
# 00:02.7 VGA compatible controller: Intel Corporation Alder Lake-S GT1 [UHD Graphics 730]
```

> `00:02.0` is the physical function (PF); `00:02.1` through `00:02.7` are the virtual functions (VFs).

### Check sriov_numvfs

```bash
cat /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs
# Expected: 7
```

### Check the Driver in Use

```bash
lspci -nnk | grep -A3 "VGA"
# Both the PF and the VFs should use the i915 driver
# Kernel driver in use: i915
```

---

## Assign a VF to a VM

### Windows VMs

#### In the Web UI

1. Select the VM → **Hardware** → **Add** → **PCI Device**
2. Pick a VF (for example `0000:00:02.1`). **Never pick 00:02.0, the PF**: passing the PF to a VM crashes all the other VFs.
3. Set the options:
   - **All Functions**: unchecked
   - **Primary GPU**: check it if this should be the VM's primary display adapter
   - **ROM-Bar**: **turning it off is recommended** (a known issue with PVE 9 on kernel 6.14)
   - **PCI-Express**: checked (requires the q35 machine type)

#### On the Command Line

```bash
# Edit the VM config (replace <VMID> with the actual ID)
nano /etc/pve/qemu-server/<VMID>.conf
```

Add this line (`rombar=0` is the CLI equivalent of unchecking ROM-Bar):

```text
hostpci0: 0000:00:02.1,pcie=1,rombar=0
```

> To use it as the primary GPU:
>
> ```text
> hostpci0: 0000:00:02.1,pcie=1,rombar=0,x-vga=1
> ```

#### Other Recommended VM Settings

```text
machine: q35
bios: ovmf
cpu: host
```

#### Install the Driver Inside Windows

1. Boot the VM and install the Intel graphics driver.
2. Recommended driver versions: **32.0.101.6460** or **32.0.101.6259** (tested and known to work).
3. Download it from the [Intel Download Center](https://www.intel.com/content/www/us/en/download-center/home.html).

### Linux VMs

Linux guests need the `i915-sriov-dkms` module as well:

1. Assign a VF to the VM as a PCIe device (same as above).
2. Inside the VM, install the build tools and headers (`apt install build-essential dkms linux-headers-$(uname -r)` on Debian/Ubuntu), then install `i915-sriov-dkms`. Using the same version as the host is the simplest choice, although upstream notes that host and guest don't strictly have to match.
3. Add `i915.enable_guc=3 module_blacklist=xe` to the guest's kernel command line (for GRUB guests: `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub`, then `update-grub` and `update-initramfs -u`), and reboot the VM.
4. Install `vainfo` to verify hardware acceleration:

```bash
# Run inside the VM
apt install vainfo
vainfo
# Should list the codecs supported through VA-API
```

### LXC Containers

If you only need hardware acceleration inside LXC (for example Jellyfin transcoding), you can map the device directly. SR-IOV isn't required for this:

```bash
# Edit the container config
nano /etc/pve/lxc/<CTID>.conf

# Add these lines
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
```

---

## Troubleshooting

### The Host Won't Boot / Black Screen

**Symptom**: the system no longer boots after changing the kernel parameters.

**Fix**: temporarily disable the drivers from the boot menu:

1. Press `e` in the systemd-boot menu to edit the boot entry.
2. Append `module_blacklist=i915,xe` (preceded by a space) to the end of the kernel command line.
3. Press Enter to boot.
4. Once the system is up, fix `/etc/kernel/cmdline` and run `proxmox-boot-tool refresh`.

**If you can't reach the boot menu at all**, rescue the system from the PVE installer ISO. Boot the ISO **in UEFI mode** (`proxmox-boot-tool` checks `/sys/firmware/efi` to decide between systemd-boot and GRUB), choose the debug-mode install under Advanced Options, and work at the second shell prompt, which has the full toolset including ZFS:

```bash
# Import the pool under /mnt without mounting anything, then mount only the root dataset
zpool import -f -N -R /mnt rpool
zfs mount rpool/ROOT/pve-1

# Bind-mount the pseudo filesystems needed inside the chroot
mount --make-private --rbind /dev  /mnt/dev
mount --make-private --rbind /proc /mnt/proc
mount --make-private --rbind /sys  /mnt/sys

chroot /mnt /bin/bash

# Restore the backup, or remove the offending parameters by hand
cp /etc/kernel/cmdline.bak /etc/kernel/cmdline   # or: nano /etc/kernel/cmdline
proxmox-boot-tool refresh
exit

# Unmount everything and export the pool before rebooting
umount -R /mnt/dev /mnt/proc /mnt/sys
zpool export rpool
reboot
```

> - `-N` keeps `zpool import` from auto-mounting datasets, and `-R /mnt` sets a temporary altroot, so nothing gets mounted over the rescue system's own `/`.
> - `proxmox-boot-tool refresh` mounts the ESPs itself and does not need a separate `efivarfs` mount (only `proxmox-boot-tool init` installs the bootloader). The recursive `/sys` bind carries `efivars` along anyway if the rescue system has it mounted.
> - Don't skip `zpool export rpool`. The pool was last imported by the rescue system, so if it isn't exported, the next boot can refuse to import it and drop you into the initramfs shell, where you would have to run `zpool import -f rpool` by hand.

### DKMS Build Fails

**Symptom**: `dkms status` shows `build error`.

**Steps**:

```bash
# Read the full build log
cat /var/lib/dkms/i915-sriov-dkms/*/build/make.log

# Common cause 1: headers don't match the running kernel
apt install proxmox-headers-$(uname -r)
# Note: reboot into the newest kernel before installing headers

# Common cause 2: leftovers from an old version
dpkg -P i915-sriov-dkms
rm -rf /var/lib/dkms/i915-sriov-dkms*
rm -rf /usr/src/i915-sriov-dkms*
# Then reinstall the deb package

# Common cause 3: the kernel version isn't supported by this DKMS release
# Check the supported kernel range in the README / release notes on GitHub
```

### No VFs (lspci Only Shows 00:02.0)

**Steps**:

```bash
# Check whether sriov_numvfs was set
cat /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs

# If it's 0, try setting it manually
echo 7 > /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs

# Look for errors in dmesg
dmesg | grep -i -e sriov -e i915 -e error

# Make sure the sysfsutils service ran
systemctl status sysfsutils

# Make sure the sysfs config is correct
cat /etc/sysfs.d/i915-sriov.conf
# Should print: devices/pci0000:00/0000:00:02.0/sriov_numvfs = 7
```

### GuC Submission Not Enabled

**Symptom**: `dmesg | grep i915` shows `Incompatible option enable_guc`.

**Cause**: this usually happens on older iGPUs without GuC support. The i3-12100's UHD 730 should support it.

```bash
# Make sure the xe driver is blocked
lsmod | grep -w xe
# Should print nothing

# Make sure the i915 module is loaded
lsmod | grep i915
# Should print something
```

### The VM Hangs or Times Out on Start

**Symptom**: after assigning a VF, the VM won't start and the console times out.

**Steps**:

1. **Turn off ROM-Bar**: a known issue with PVE 9 on kernel 6.14.

   - Web UI → VM → Hardware → PCI Device → Edit → **uncheck ROM-Bar**

2. **Use a newer machine version**:

   - VM → **Hardware** → **Machine** → Edit, then pick **9.1** or newer (or Latest) in the **Version** drop-down (shown under Advanced). Windows VMs are pinned to the machine version they were created with, so an older VM may still be on an old one.

3. **Check the VM config**:

   ```bash
   cat /etc/pve/qemu-server/<VMID>.conf
   # Make sure it uses the q35 machine type and OVMF
   # machine: q35
   # bios: ovmf
   ```

### SR-IOV Stops Working After a Kernel Update

The DKMS module has to be rebuilt for every new PVE kernel. With `proxmox-default-headers` installed, the matching headers come with the kernel update and DKMS normally rebuilds the module automatically. If it didn't, fix it after booting into the new kernel:

```bash
# Check whether the automatic build succeeded
dkms status

# If the new kernel isn't listed as installed
apt install proxmox-headers-$(uname -r)
dkms autoinstall

# Refresh the boot configuration
proxmox-boot-tool refresh
reboot
```

If the new kernel is outside the range supported by your `i915-sriov-dkms` release, install a release that supports it.

> **Tip**: pin the known-good kernel before updating:
>
> ```bash
> proxmox-boot-tool kernel pin $(uname -r)
> proxmox-boot-tool refresh
> ```
>
> Once the new kernel and DKMS module are confirmed working, unpin it:
>
> ```bash
> proxmox-boot-tool kernel unpin
> proxmox-boot-tool refresh
> ```

---

## Command Cheat Sheet

```bash
# === Status ===
lspci | grep VGA                                          # List all GPU devices
dmesg | grep i915                                         # i915 driver log
dmesg | grep -i iommu                                     # IOMMU status
cat /proc/cmdline                                         # Current kernel parameters
cat /sys/devices/pci0000:00/0000:00:02.0/sriov_numvfs     # Number of VFs
lspci -nnk | grep -A3 VGA                                 # Driver binding
dkms status                                               # DKMS module status
proxmox-boot-tool status                                  # Boot tool status

# === Maintenance ===
proxmox-boot-tool refresh                # Refresh systemd-boot (required after editing cmdline)
update-initramfs -u -k all               # Update the initramfs
proxmox-boot-tool kernel pin <version>   # Pin a kernel version
proxmox-boot-tool kernel unpin           # Remove the pin

# === Debugging ===
cat /var/lib/dkms/i915-sriov-dkms/*/build/make.log   # DKMS build log
journalctl -b | grep i915                            # i915 log for the current boot
systemctl status sysfsutils                          # sysfsutils service status
```

---

## References

- **DKMS driver project**: <https://github.com/strongtz/i915-sriov-dkms>
- **PVE host bootloader docs**: <https://pve.proxmox.com/wiki/Host_Bootloader>
- **PVE PCIe passthrough docs**: <https://pve.proxmox.com/wiki/PCI(e)_Passthrough>
- **PVE package repositories**: <https://pve.proxmox.com/wiki/Package_Repositories>
- **Community guide**: <https://github.com/Upinel/PVE-Intel-vGPU>

---

_Last updated: 2026-03_
