![bild](https://github.com/jzenzen/cudaInLXC/assets/5422225/a4c1d922-0d5c-4b4f-9815-1af755685c5c)# cudaInLXC
Nvidia Cuda in Proxmox LXC or any other LXC under Linux and more specifically Debian in this example.

This guid is based on Debian Bookworm and/or Proxmomx 8

Inspired by: 
* https://github.com/gma1n/LXC-JellyFin-GPU
* https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian&target_version=12&target_type=deb_network
* https://docs.nvidia.com/cuda/cuda-installation-guide-linux/#meta-packages

## Check for IOMMU
```
dmesg | grep IOMMU
```
Should result in something like:
```
[    0.554887] pci 0000:00:00.2: AMD-Vi: IOMMU performance counters supported
[    0.560664] pci 0000:00:00.2: AMD-Vi: Found IOMMU cap 0x40
[    0.560961] perf/amd_iommu: Detected AMD IOMMU #0 (2 banks, 4 counters/bank).
```
If you get nothing you better check your bios.

##Debian/Proxmox setuo
```
apt install -y pve-headers build-essential
```
or if you are on Debian and not in Proxmox
```
apt install -y linux-headers build-essential
```

Blacklist nouveau
```
echo "blacklist nouveau" > /etc/modprobe.d/blacklist.conf
echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u
```

##Nvidia CUDA
```
wget https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
echo "deb [signed-by=/usr/share/keyrings/cuda-archive-keyring.gpg] https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/ /" |  tee /etc/apt/sources.list.d/cuda-debian12-x86_64.list
add-apt-repository contrib
apt update
apt-get install -y nvidia-kernel-open-dkms
apt-get install -y cuda-drivers
```

## Now add the output of this to your LXC settings
```
ls -l /dev/nv* |grep -v nvme | grep crw | sed -e 's/.*root root\s*\(.*\),.*\/dev\/\(.*\)/lxc.cgroup2.devices.allow: c \1:* rw\nlxc.mount.entry: \/dev\/\2 \2 none bind,optional,create=file/g'
```
Should look something like this:
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

# Inside the LXC container
