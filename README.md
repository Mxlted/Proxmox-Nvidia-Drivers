# NVIDIA Driver Setup on Proxmox

Step-by-step instructions for installing NVIDIA drivers on a Proxmox host and within a container for GPU passthrough.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Setup on Proxmox Host](#setup-on-proxmox-host)
- [Setup in a Container](#setup-in-a-container)
- [NVENC Session Limit Patch (Optional)](#nvenc-session-limit-patch-optional)
- [Updating Drivers](#updating-drivers)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Proxmox VE 9.x instance (tested on 9.1.9, kernel 6.17 default / 7.0 opt-in)
- SSH access to the Proxmox host
- NVIDIA GPU compatible with the target driver version
- Confirm driver/GPU compatibility at https://www.nvidia.com/Download/index.aspx

---

## Setup on Proxmox Host

### 1. Install Dependencies

Install the kernel headers meta-package (no version suffix) so headers automatically pull in on future kernel updates, along with build tools and DKMS:

```bash
apt update
apt install -y pve-headers build-essential dkms
```

### 2. Blacklist Nouveau

Create the blacklist file:

```bash
cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF
```

Update initramfs and reboot:

```bash
update-initramfs -u
reboot
```

After rebooting, confirm nouveau is not loaded:

```bash
lsmod | grep nouveau
```

### 3. Download the Driver

Create a directory for the driver, then grab the latest x64 `.run` file from https://download.nvidia.com/XFree86/Linux-x86_64/

```bash
mkdir -p /opt/nvidia
cd /opt/nvidia
wget https://download.nvidia.com/XFree86/Linux-x86_64/595.71.05/NVIDIA-Linux-x86_64-595.71.05.run
chmod +x NVIDIA-Linux-x86_64-595.71.05.run
```

### 4. Install the Driver

Run the installer with DKMS so the kernel module automatically rebuilds on future kernel updates:

```bash
./NVIDIA-Linux-x86_64-595.71.05.run --dkms --no-questions --ui=none --disable-nouveau
```

### 5. Enable the Persistence Daemon

On headless systems like Proxmox, the NVIDIA kernel module can unload when no GPU clients are running, which causes `/dev/nvidia*` device nodes to disappear. The persistence daemon keeps the driver loaded at all times, ensuring containers can always access the GPU.

Enable and start it:

```bash
systemctl enable nvidia-persistenced
systemctl start nvidia-persistenced
```

Confirm it is running:

```bash
systemctl status nvidia-persistenced
```

You should see `Persistence-M: On` in `nvidia-smi` output after this is active.

### 6. Install the NVIDIA Container Toolkit

Required for GPU access from LXC containers:

```bash
apt install -y nvidia-container-toolkit
```

### 7. Verify

```bash
nvidia-smi
```

Confirm the driver version and GPU are listed, then reboot:

```bash
reboot
```

After reboot, run `nvidia-smi` again to confirm the driver persists and persistence mode is on. Optionally verify DKMS is tracking the module:

```bash
dkms status
```

---

## Setup in a Container

The container installation follows the same download steps as the host but skips loading the kernel module since the host already owns it. The driver version **must match** the host.

### 1. Download the Driver

```bash
mkdir -p /opt/nvidia
cd /opt/nvidia
wget https://download.nvidia.com/XFree86/Linux-x86_64/595.71.05/NVIDIA-Linux-x86_64-595.71.05.run
chmod +x NVIDIA-Linux-x86_64-595.71.05.run
```

### 2. Install Without the Kernel Module

```bash
./NVIDIA-Linux-x86_64-595.71.05.run --no-kernel-module
```

### 3. Reboot and Verify

```bash
reboot
```

After reboot:

```bash
nvidia-smi
```

---

## NVENC Session Limit Patch (Optional)

Consumer-grade NVIDIA GPUs limit the number of concurrent NVENC encoding sessions (currently capped at 5). If you run workloads like Plex, Jellyfin, or Immich that need more simultaneous hardware transcode sessions, you can remove this limit using the [nvidia-patch](https://github.com/keylase/nvidia-patch) project.

Driver 595.71.05 is supported for both the NVENC and NvFBC patches.

### Apply on the Host

```bash
cd /opt/nvidia
git clone https://github.com/keylase/nvidia-patch.git
cd nvidia-patch
bash ./patch.sh
```

If you also need NvFBC (frame buffer capture for streaming tools like Moonlight/Sunshine):

```bash
bash ./patch-fbc.sh
```

### Verify

Run more concurrent NVENC sessions than the default limit allows. The patch project wiki has an [ffmpeg-based verification method](https://github.com/keylase/nvidia-patch/wiki/Verify-NVENC-patch).

### Rollback

If something goes wrong, the patch backs up the original file automatically:

```bash
bash ./patch.sh -r
bash ./patch-fbc.sh -r
```

### Note for Containers

It is possible to use this patch inside nvidia-docker containers even if the host drivers are not patched. See the `Dockerfile` in the nvidia-patch repo for an example of on-the-fly patching via dynamic linker manipulation.

---

## Updating Drivers

> **Important:** Perform the update on the host first, then repeat in each container with the matching driver version.

### 1. Prepare the System

Before touching the old driver, make sure headers and nouveau blacklisting are in place. This prevents nouveau from grabbing the GPU after the old driver is removed.

Confirm the blacklist file exists at `/etc/modprobe.d/blacklist-nouveau.conf` with:

```
blacklist nouveau
options nouveau modeset=0
```

Install headers for the current kernel (if not already covered by the meta-package):

```bash
apt update
apt install -y pve-headers build-essential dkms
```

### 2. Download the New Driver

Get the newest `.run` file from https://download.nvidia.com/XFree86/Linux-x86_64/ and place it in `/opt/nvidia`. Do not run it yet.

```bash
cd /opt/nvidia
wget https://download.nvidia.com/XFree86/Linux-x86_64/NEW_VERSION/NVIDIA-Linux-x86_64-NEW_VERSION.run
chmod +x NVIDIA-Linux-x86_64-NEW_VERSION.run
```

### 3. Uninstall the Current Driver

```bash
cd /opt/nvidia
./CURRENT_DRIVER.run --uninstall
```

### 4. Clean Up and Reboot

```bash
apt purge -y '*nvidia*'
apt autoremove --purge -y
apt clean
update-initramfs -u
reboot
```

### 5. Install the New Driver

```bash
cd /opt/nvidia
./NVIDIA-Linux-x86_64-NEW_VERSION.run --dkms --disable-nouveau
```

Accept the recommended defaults when prompted.

### 6. Re-enable Persistence and Verify

```bash
systemctl enable nvidia-persistenced
systemctl start nvidia-persistenced
nvidia-smi
dkms status
reboot
```

After reboot, confirm with `nvidia-smi` one more time, then repeat the container installation steps with the matching new driver version using `--no-kernel-module`.

### 7. Re-apply NVENC Patch (If Used)

If you previously applied the nvidia-patch, you need to re-apply it after every driver update since the patched library gets replaced:

```bash
cd /opt/nvidia/nvidia-patch
git pull
bash ./patch.sh
bash ./patch-fbc.sh  # only if you use NvFBC
```

---

## Troubleshooting

- **Driver version compatibility** - confirm your driver version supports your GPU at https://www.nvidia.com/Download/index.aspx

- **Kernel module build failures** - verify that `nouveau` is blacklisted and that kernel headers match the running kernel before attempting the install:
  ```bash
  uname -r
  dpkg -l | grep pve-headers
  lsmod | grep nouveau
  ```

- **Driver doesn't survive kernel updates** - make sure you installed with `--dkms` and that the `pve-headers` meta-package (without version suffix) is installed. Verify with `dkms status` that the NVIDIA module is registered.

- **GPU device nodes missing after reboot** - this usually means the persistence daemon is not running. Check with `systemctl status nvidia-persistenced` and enable it if needed. Some setups use a cron-based workaround instead:
  ```
  @reboot root /usr/bin/nvidia-smi
  @reboot root /usr/bin/nvidia-persistenced
  ```

- **Kernel 7.0 on PVE 9.x** - the 7.0 kernel is opt-in as of PVE 9.1 and planned as the default in 9.2. The R595 driver branch is compatible with kernel 7.0, but if you hit build failures on a bleeding-edge kernel, you can pin back to 6.17:
  ```bash
  proxmox-boot-tool kernel list
  proxmox-boot-tool kernel pin <version>
  ```

- **Secure Boot** - if Secure Boot is enabled, DKMS-built kernel modules must be signed with a Machine Owner Key (MOK). You will need to generate a signing key, enroll it via `mokutil`, and configure DKMS to sign modules automatically. See the [Proxmox vGPU wiki](https://pve.proxmox.com/wiki/NVIDIA_vGPU_on_Proxmox_VE) for details.

- **Container can't see the GPU** - ensure the container's driver version exactly matches the host. The container must use `--no-kernel-module` since the host owns the kernel module.

- For additional help, refer to the [Proxmox community forums](https://forum.proxmox.com/) or NVIDIA support.
