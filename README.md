# NVIDIA Driver Setup on Proxmox

This guide provides step-by-step instructions to set up NVIDIA drivers on a Proxmox host and within a container for GPU passthrough.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Setup NVIDIA Drivers on Proxmox Host](#setup-nvidia-drivers-on-proxmox-host)
- [Setup NVIDIA Drivers in a Container](#setup-nvidia-drivers-in-a-container)
- [Update Drivers](#update-drivers)
- [Troubleshooting](#troubleshooting)

## Prerequisites
- A Proxmox VE instance.
- Access to the Proxmox host via SSH.
- An NVIDIA GPU compatible with the driver version provided.

## Setup NVIDIA Drivers on Proxmox Host

To install the NVIDIA drivers on the Proxmox host, follow these steps:

1. **Create a directory for NVIDIA drivers:**
```bash
mkdir /opt/nvidia
cd /opt/nvidia
```

2. **Download the NVIDIA driver**

   **(Scroll to the bottom and grab the latest x64 version from here https://download.nvidia.com/XFree86/Linux-x86_64/) :**
```bash
wget https://download.nvidia.com/XFree86/Linux-x86_64/590.48.01/NVIDIA-Linux-x86_64-590.48.01.run
```

3. **Make the driver installer executable:**
```bash
chmod +x NVIDIA-Linux-x86_64-590.48.01.run
```

4. **Run the installer with the necessary options:**
```bash
./NVIDIA-Linux-x86_64-590.48.01.run --no-questions --ui=none --disable-nouveau
```

This will install the NVIDIA drivers on the Proxmox host, allowing it to communicate with the GPU.

## Setup NVIDIA Drivers in a Container

To enable a container to communicate with the GPU and the installed NVIDIA driver, follow these steps within the container:

1. **Create a directory for NVIDIA drivers:**
```bash
mkdir /opt/nvidia
cd /opt/nvidia
```

2. **Download the NVIDIA driver:**
```bash
wget https://download.nvidia.com/XFree86/Linux-x86_64/590.48.01/NVIDIA-Linux-x86_64-590.48.01.run
```

3. **Make the driver installer executable:**
```bash
chmod +x NVIDIA-Linux-x86_64-590.48.01.run
```

4. **Run the installer with the following options:**
```bash
sudo ./NVIDIA-Linux-x86_64-590.48.01.run --no-kernel-module
```

This process will install the NVIDIA drivers within the container without attempting to load the kernel module.

# **Then REBOOT the container to finalize!**

## Update Drivers

Follow these steps whenever you want to update your NVIDIA drivers on Proxmox.

### 1. Download the Latest Driver (Do Not Install Yet)

Just like the install steps above, grab the newest `.run` file from:

https://download.nvidia.com/XFree86/Linux-x86_64/
Place it inside:
```bash
cd /opt/nvidia
```

### 2. Uninstall the Previous NVIDIA Driver

Run the uninstall command from the same directory:

```bash
./DRIVER_NAME.run --uninstall
```

### 3. Purge Any Leftover NVIDIA Packages

After uninstalling, reboot the host:

```bash
reboot
```

Then remove leftover packages:

```bash
apt purge -y '*nvidia*'
apt autoremove --purge -y
apt clean
```

### 4. Fix Kernel + Headers (Proxmox Requirement)

Make sure Proxmox has matching headers installed:

```bash
apt update
apt install -y pve-headers-$(uname -r) build-essential dkms
```

### 5. Blacklist Nouveau the Proxmox Way

Create or edit your nouveau blacklist file:

```bash
nano /etc/modprobe.d/blacklist-nouveau.conf
```

(Or whatever file you already use inside `/etc/modprobe.d/`)

Make sure it contains:

```bash
blacklist nouveau
options nouveau modeset=0
```

Update initramfs:

```bash
update-initramfs -u
```

Reboot again:

```bash
reboot
```

### 6. Install NVIDIA Container Toolkit (New Versions)

```bash
apt install -y nvidia-container-toolkit
```

### 7. Install the Latest NVIDIA Driver

Back in the driver folder:

```bash
cd /opt/nvidia
```

Run the installer with nouveau disabled:

```bash
./LATEST_DRIVER.run --disable-nouveau
```

During the prompts, choose the recommended defaults (usually "Yes" on most options).

### 8. Verify + Final Reboot

Check driver status:

```bash
nvidia-smi
```

Then reboot the container again to finalize:

```bash
reboot
```

## Troubleshooting

- **Driver Version Compatibility:** Ensure the NVIDIA driver version is compatible with your GPU model. Check NVIDIA's [official website](https://www.nvidia.com/Download/index.aspx) for more details.
- **Kernel Module Issues:** If the installation fails due to kernel module issues, ensure the `nouveau` driver is disabled, and that the host kernel is updated.

For further assistance, refer to the Proxmox community forums or NVIDIA support.
