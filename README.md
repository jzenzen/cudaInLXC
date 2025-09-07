# NvidiaGPU in Proxmox Host then unprivileged LXC and finally available to Docker

Nvidia in Proxmox LXC or any other LXC under Linux, specifically Debian in this example.

This guide is based on Debian Bookworm and/or Proxmox 8.  
I'm not the original author—see fork/references below!

Inspired by:  
* https://github.com/gma1n/LXC-JellyFin-GPU  
* https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian&target_version=12&target_type=deb_network  
* https://docs.nvidia.com/cuda/cuda-installation-guide-linux/#meta-packages  
* https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorflow  
* https://github.com/NVIDIA/libnvidia-container/issues/176  
* https://gist.github.com/MihailCosmin/affa6b1b71b43787e9228c25fe15aeba  
* https://sluijsjes.nl/2024/05/18/coral-and-nvidia-passthrough-for-proxmox-lxc-to-install-frigate-video-surveillance-server/  
* https://stackoverflow.com/questions/8223811/a-top-like-utility-for-monitoring-cuda-activity-on-a-gpu  
* https://forum.proxmox.com/threads/nvidia-drivers-instalation-proxmox-and-ct.156421/  
* https://hostbor.com/gpu-passthrough-in-lxc-containers/  

---

## Check for IOMMU

```bash
dmesg | grep IOMMU
```
Should result in something like:

```
[    0.554887] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported  
[    0.560664] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40  
[    0.560961] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
```
If you get nothing, check your BIOS.

---

## HOST Debian/Proxmox setup

**Install required packages. This will future-proof for DKMS + all kernel upgrades.**

```bash
apt install -y dkms pve-headers build-essential libvulkan1
```
> **Note:** The `pve-headers` meta-package keeps headers for the newest kernel automatically installed after upgrades.  
> Use `pve-headers-$(uname -r)` only if you are running an older kernel, but usually this is not needed.

_For Debian (not Proxmox):_
```bash
sudo apt install -y dkms linux-headers build-essential libvulkan1
```

---

### Blacklist nouveau

```bash
echo "blacklist nouveau" > /etc/modprobe.d/blacklist.conf
echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u
```

---

## Nvidia Driver

### Download and install the latest version—check for your card at https://www.nvidia.com/en-us/drivers/unix/

```bash
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/570.172.08/NVIDIA-Linux-x86_64-570.172.08.run
sh NVIDIA-Linux-x86_64-570.172.08.run --dkms
```
The installer has a few prompts. Skip secondary cards, No 32 bits, No X. Near end will ask about register kernel module sources with dkms - **YES**.

---

### Verify DKMS status

```bash
dkms status
# Should show: nvidia, 570.172.08, <kernel-version>: installed
```
> If not present, repeat the installer after confirming both `pve-headers` and `dkms` are installed.

---

## [Optional, but HIGHLY recommended on HOST] Persistent Mode (Across Reboots)

[NVIDIA Official Reference](https://docs.nvidia.com/deploy/driver-persistence/index.html)  
Ensures your GPU remains initialized and responsive across reboots—**critical for Proxmox, LXC, and Docker GPU workflows**.

1. **Create or overwrite `/etc/systemd/system/nvidia-persistenced.service` with:**

```
[Unit]
Description=NVIDIA Persistence Daemon
After=network.target

[Service]
Type=forking
ExecStart=/usr/bin/nvidia-persistenced --user root
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

2. **Register, enable, and start the service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nvidia-persistenced
sudo systemctl status nvidia-persistenced
```

3. **Result:** Persistence mode will now survive *all* reboots automatically.

> **Note:** No need for manual `nvidia-smi --persistence-mode=1`.  
> Use `systemctl` to manage or check the daemon.

---

## Test the driver is working (you only need to do one of the below tests)

### You can do this same test inside the LXC to confirm the driver install is good there too!

This will loop and call the view at every second.
```bash
nvidia-smi -l 1
```
If you do not want to keep past traces of the looped call in the console history, you can also do:
```bash
# Where 0.1 is the time interval, in seconds.
watch -n0.1 nvidia-smi
```

---

## Add the output of this command to your LXC config file (/etc/pve/nodes/pve/lxc/xxx.conf)

```bash
ls -l /dev/nv* |grep -v nvme | grep crw | sed -e 's/.*root root\s*\(.*\),.*\/dev\/\(.*\)/lxc.cgroup2.devices.allow: c \1:* rwm\nlxc.mount.entry: \/dev\/\2 dev\/\2 none bind,optional,create=file/g'
```
Should look something like this (Do not blindly copy the below):
```
lxc.cgroup2.devices.allow: c 195:* rw
lxc.mount.entry: /dev/nvidia0 nvidia0 none bind,optional,create=file
lxc.cgroup2.devices.allow: c 195:* rw
lxc.mount.entry: /dev/nvidiactl nvidiactl none bind,optional,create=file
lxc.cgroup2.devices.allow: c 195:* rw
lxc.mount.entry: /dev/nvidia-modeset nvidia-modeset none bind,optional,create=file
lxc.cgroup2.devices.allow: c 236:* rw
lxc.mount.entry: /dev/nvidia-uvm nvidia-uvm none bind,optional,create=file
lxc.cgroup2.devices.allow: c 236:* rw
lxc.mount.entry: /dev/nvidia-uvm-tools nvidia-uvm-tools none bind,optional,create=file
lxc.cgroup2.devices.allow: c 10:* rw
lxc.mount.entry: /dev/nvram nvram none bind,optional,create=file
```

> **Note:** For unprivileged LXCs with file/bind mounts, ensure correct UID/GID mapping for storage access.

---

# Inside the LXC container

## Build Nvidia driver & install Nvidia container toolkit

```bash
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/570.172.08/NVIDIA-Linux-x86_64-570.172.08.run
sh NVIDIA-Linux-x86_64-570.172.08.run --no-kernel-module
# The installer has a few prompts. Skip secondary cards, No 32 bits, No X 

#############Install NVIDIA Container Toolkit
apt install curl gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    tee /etc/apt/sources.list.d/nvidia-container-toolkit.list \
  && \
    apt-get update

apt-get install -y nvidia-container-toolkit

nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
sed -i -e 's/.*no-cgroups.*/no-cgroups = true/g' /etc/nvidia-container-runtime/config.toml
```

---

### TEST setup

```bash
# Docker test
docker run --gpus all nvidia/cuda:12.6.1-base-ubuntu24.04 nvidia-smi
```
Now you have everything working in the docker!

All the tests provided should give an output similar to:
```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.144.03             Driver Version: 550.144.03     CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Quadro P4000                   On  |   00000000:86:00.0 Off |                  N/A |
| 46%   30C    P8              5W /  105W |       5MiB /   8192MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

---

## After a Proxmox or Kernel Upgrade

- **Reboot into the new kernel**.
- Run `dkms status` to confirm NVIDIA modules are built for the new kernel.
- Run `nvidia-smi` (host) to verify driver is active.
- Headers are managed by `pve-headers`; if issues arise, manually install with `apt install pve-headers-$(uname -r)`.

---

## Tips

- **Version Matching:** For maximum reliability, keep NVIDIA driver versions the same between host and LXC container.  
- **Troubleshooting:** If Docker inside LXC can't see the GPU, double-check device mappings and driver versions.
- **Cleanup:** Old kernels/headers can accumulate; if disk usage becomes an issue, use `apt autoremove` after rebooting to latest kernel.

---

## Uninstalling/Upgrading

If you need to uninstall a version use the command (not the .run application downloaded for install):
This is also used if you wish to upgrade the driver. Uninstall old on Host AND all LXC's. Then follow directions above to reinstall latest wanted version. 
```bash
nvidia-installer --uninstall 
```
