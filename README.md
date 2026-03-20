# NVIDIA Driver Setup on Proxmox

Step-by-step instructions for installing NVIDIA drivers on a Proxmox host and within a container for GPU passthrough.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup on Proxmox Host](#setup-on-proxmox-host)
- [Setup in a Container](#setup-in-a-container)
- [Updating Drivers](#updating-drivers)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Proxmox VE instance
- SSH access to the Proxmox host
- NVIDIA GPU compatible with the target driver version

---

## Setup on Proxmox Host

1. Create a directory for the driver and navigate to it:

```bash
mkdir /opt/nvidia
cd /opt/nvidia
```

2. Download the driver. Grab the latest x64 `.run` file from https://download.nvidia.com/XFree86/Linux-x86_64/

```bash
wget https://download.nvidia.com/XFree86/Linux-x86_64/590.48.01/NVIDIA-Linux-x86_64-590.48.01.run
```

3. Make it executable:

```bash
chmod +x NVIDIA-Linux-x86_64-590.48.01.run
```

4. Run the installer:

```bash
./NVIDIA-Linux-x86_64-590.48.01.run --no-questions --ui=none --disable-nouveau
```

---

## Setup in a Container

The container installation follows the same steps as the host, but skips loading the kernel module since the host already owns it.

1. Create a directory for the driver and navigate to it:

```bash
mkdir /opt/nvidia
cd /opt/nvidia
```

2. Download the same driver version used on the host:

```bash
wget https://download.nvidia.com/XFree86/Linux-x86_64/590.48.01/NVIDIA-Linux-x86_64-590.48.01.run
```

3. Make it executable:

```bash
chmod +x NVIDIA-Linux-x86_64-590.48.01.run
```

4. Run the installer without the kernel module:

```bash
./NVIDIA-Linux-x86_64-590.48.01.run --no-kernel-module
```

5. Reboot the container to finalize:

```bash
reboot
```

---

## Updating Drivers

### 1. Download the Latest Driver

Get the newest `.run` file from https://download.nvidia.com/XFree86/Linux-x86_64/ and place it in `/opt/nvidia`. Do not run it yet.

### 2. Uninstall the Current Driver

```bash
cd /opt/nvidia
./CURRENT_DRIVER.run --uninstall
```

### 3. Reboot and Purge Leftover Packages

```bash
reboot
```

After rebooting:

```bash
apt purge -y '*nvidia*'
apt autoremove --purge -y
apt clean
```

### 4. Install Kernel Headers

Proxmox requires matching headers before the driver can build:

```bash
apt update
apt install -y pve-headers-$(uname -r) build-essential dkms
```

### 5. Blacklist Nouveau

Create or edit the blacklist file:

```bash
nano /etc/modprobe.d/blacklist-nouveau.conf
```

Ensure it contains:

```
blacklist nouveau
options nouveau modeset=0
```

Update initramfs and reboot:

```bash
update-initramfs -u
reboot
```

### 6. Install the NVIDIA Container Toolkit

```bash
apt install -y nvidia-container-toolkit
```

### 7. Install the New Driver

```bash
cd /opt/nvidia
./LATEST_DRIVER.run --disable-nouveau
```

Accept the recommended defaults when prompted.

### 8. Verify and Reboot

```bash
nvidia-smi
reboot
```

---

## Troubleshooting

- **Driver version compatibility** - confirm your driver version supports your GPU at https://www.nvidia.com/Download/index.aspx
- **Kernel module failures** - verify `nouveau` is blacklisted and that `pve-headers-$(uname -r)` matches the running kernel before attempting the install again
- For additional help, refer to the [Proxmox community forums](https://forum.proxmox.com/) or NVIDIA support
