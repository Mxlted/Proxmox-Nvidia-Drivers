# NVIDIA Driver Setup on Proxmox

This guide provides step-by-step instructions to set up NVIDIA drivers on a Proxmox host and within a container for GPU passthrough.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Setup NVIDIA Drivers on Proxmox Host](#setup-nvidia-drivers-on-proxmox-host)
- [Setup NVIDIA Drivers in a Container](#setup-nvidia-drivers-in-a-container)
- [Troubleshooting](#troubleshooting)

## Prerequisites
- A Proxmox VE instance.
- Access to the Proxmox host via SSH.
- An NVIDIA GPU compatible with the driver version provided.

## Setup NVIDIA Drivers on Proxmox Host

To install the NVIDIA drivers on the Proxmox host, follow these steps:

1. **Create a directory for NVIDIA drivers:**

   `mkdir /opt/nvidia`
   
   `cd /opt/nvidia`

2. **Download the NVIDIA driver:**

   `wget https://download.nvidia.com/XFree86/Linux-x86_64/560.28.03/NVIDIA-Linux-x86_64-560.28.03.run`

3. **Make the driver installer executable:**

   `chmod +x NVIDIA-Linux-x86_64-560.28.03.run`

4. **Run the installer with the necessary options:**

   `./NVIDIA-Linux-x86_64-560.28.03.run --no-questions --ui=none --disable-nouveau`

This will install the NVIDIA drivers on the Proxmox host, allowing it to communicate with the GPU.

## Setup NVIDIA Drivers in a Container

To enable a container to communicate with the GPU and the installed NVIDIA driver, follow these steps within the container:

1. **Create a directory for NVIDIA drivers:**

   `mkdir /opt/nvidia`
   
   `cd /opt/nvidia`

2. **Download the NVIDIA driver:**

   `wget https://download.nvidia.com/XFree86/Linux-x86_64/560.28.03/NVIDIA-Linux-x86_64-560.28.03.run`

3. **Make the driver installer executable:**

   `chmod +x NVIDIA-Linux-x86_64-560.28.03.run`

4. **Run the installer with the following options:**

   `sudo ./NVIDIA-Linux-x86_64-560.28.03.run --no-kernel-module`

This process will install the NVIDIA drivers within the container without attempting to load the kernel module.

## Troubleshooting

- **Driver Version Compatibility:** Ensure the NVIDIA driver version is compatible with your GPU model. Check NVIDIA's [official website](https://www.nvidia.com/Download/index.aspx) for more details.
- **Kernel Module Issues:** If the installation fails due to kernel module issues, ensure the `nouveau` driver is disabled, and that the host kernel is updated.

For further assistance, refer to the Proxmox community forums or NVIDIA support.
