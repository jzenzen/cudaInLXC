# cudaInLXC
Nvidia Cuda in Proxmox LXC or any other LXC under Linux and more specifically Debian in this example.

This guid is based on Debian Bookworm and/or Proxmomx 8

Inspired by: 
* https://github.com/gma1n/LXC-JellyFin-GPU
* https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=Debian&target_version=12&target_type=deb_network
* https://docs.nvidia.com/cuda/cuda-installation-guide-linux/#meta-packages
* https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorflow
* https://github.com/NVIDIA/libnvidia-container/issues/176
* https://gist.github.com/MihailCosmin/affa6b1b71b43787e9228c25fe15aeba

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

## Debian/Proxmox setuo
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

## Nvidia 
Choose one!
### Driver
```
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/535.129.03/NVIDIA-Linux-x86_64-535.129.03.run
sh NVIDIA-Linux-x86_64-535.129.03.run
```

### CUDA
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
ls -l /dev/nv* |grep -v nvme | grep crw | sed -e 's/.*root root\s*\(.*\),.*\/dev\/\(.*\)/lxc.cgroup2.devices.allow: c \1:* rwm\nlxc.mount.entry: \/dev\/\2 dev\/\2 none bind,optional,create=file/g'
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
Choose one

### Build Nvidia driver & Cuda & CudNN
```
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/535.129.03/NVIDIA-Linux-x86_64-535.129.03.run
sh NVIDIA-Linux-x8a_64-535.129.03.run --no-kernel-module

#############Use NVIDIA Container
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

#TEST setup
docker run --gpus all -it --rm nvcr.io/nvidia/tensorflow:23.10-tf2-py3

python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"

```
Now you have everything working in the docker!

You can also try to build everything as done below:
```
############Build everything
apt install perl-modules g++
wget https://developer.download.nvidia.com/compute/cuda/12.3.0/local_installers/cuda_12.3.0_545.23.06_linux.run
sh cuda_12.3.0_545.23.06_linux.run --toolkit --override
echo "/usr/local/cuda-12.3/lib64" > /etc/ld.so.conf.d/cuda-12-3.conf

wget https://developer.download.nvidia.com/compute/cuda/12.2.2/local_installers/cuda_12.2.2_535.104.05_linux.run
sh cuda_12.2.2_535.104.05_linux.run --toolkit --silent --override

wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda_11.8.0_520.61.05_linux.run
sh cuda_11.8.0_520.61.05_linux.run --toolkit --silent --override

apt-get install zlib1g
tar -xvf cudnn-linux-*.tar.xz
cp cudnn-*-archive/include/cudnn*.h /usr/local/cuda/include
cp -P cudnn-*-archive/lib/libcudnn* /usr/local/cuda/lib64
chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*


```

Output:
```
===========
= Summary =
===========

Driver:   Not Selected
Toolkit:  Installed in /usr/local/cuda-12.3/

Please make sure that
 -   PATH includes /usr/local/cuda-12.3/bin
 -   LD_LIBRARY_PATH includes /usr/local/cuda-12.3/lib64, or, add /usr/local/cuda-12.3/lib64 to /etc/ld.so.conf and run ldconfig as root

To uninstall the CUDA Toolkit, run cuda-uninstaller in /usr/local/cuda-12.3/bin
***WARNING: Incomplete installation! This installation did not install the CUDA Driver. A driver of version at least 545.00 is required for CUDA 12.3 functionality to work.
To install the driver using this installer, run the following command, replacing <CudaInstaller> with the name of this run file:
    sudo <CudaInstaller>.run --silent --driver

Logfile is /var/log/cuda-installer.log
```


### CUDA - probably not working...
```
wget https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
echo "deb [signed-by=/usr/share/keyrings/cuda-archive-keyring.gpg] https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/ /" |  tee /etc/apt/sources.list.d/cuda-debian12-x86_64.list
add-apt-repository contrib
apt update
apt-get install -y cuda-drivers

```
