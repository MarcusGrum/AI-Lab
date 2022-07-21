# Welcome to the AILab

This repository supports the standardized node configuration, 
so that laboratory components work in harmony.

The lab configuration tool was originally developed by Dr.-Ing. Marcus Grum.

This configuration has been tested on `Ubuntu-20.04` (using `ubuntu-20.04-server-amd64.iso`) 
as well as on `Ubuntu22.04` (using `ubuntu-22.04-server-amd64.iso`).

## Download and install newest Ubuntu

1. Download the recent Ubuntu (server version) from
```https://ubuntu.com/download```

1. Create a bootable USB stick with the help of Etcher. Detailed steps can be found at
```https://ubuntu.com/tutorials/create-a-usb-stick-on-macos#4-install-and-run-etcher```

1. Install Ubuntu and follow Ubuntu installation instructions.
Please consider the following decisions, here:

	* Computer name follows naming convention `AILabNode02`.

	* OS is installed on SSDs on a `RAID1`. This consumes SSD1 and SSD2. 
	This can not be specified via BIOS before Ubuntu installation.
	This can not be specified via Ubuntu-desktop installation.
	This can not be specified after Ubuntu-desktop installation.
	This only can be specified via Ubuntu-server installation.

	* Data tank is on HDDs on a `RAID5`. This consumes HDD1, HDD2 and HDD3.
	This can not be specified via BIOS before Ubuntu installation.
	This can not be specified via Ubuntu-desktop installation.
	This can either be specified after Ubuntu-desktop installation, or
	this only can be specified via Ubuntu-server installation.

	* A `softraid`on `btfs` is based on SSD3 being the `cache` and Data tank being the `data`.
	So, cached data is backed up at runtime when processing time is available.
	This is specified via OS after Ubuntu installation.

	* `RAID1` is based on three partitions. 
	First, 200MB for `EFI`.
	Second, `EXT4` being mounted at `/`.
	Third, `SWAP` partition having the size of RAM.

	* Use `LVM` with the new Ubuntu installation, so that you easily can deal with partitions, later.

1. At Ubuntu server installation, setup partitions according to `https://alexskra.com/blog/ubuntu-20-04-with-software-raid1-and-uefi/`.
Only then, `RAID1` can be setup as required.

   - Reformat both drives if they’re not empty.

   - Mark both drives as a boot device. Doing so will create an ESP(EFI system partition) on both drives.

   - Add an unformatted GPT partition to both drives. They need to have the same size. We’re going to use those partitions for the RAID that contains the OS.

   - Create a software RAID(md) by selecting the two partitions you just created for the OS.

   - Congratulations, you now have a new RAID device. Let’s add at least one GPT partition to it.

   - Optional: If you want the ability to swap, create a swap partition on the RAID device. Set the size to the same as your RAM, or half if you have 64 GB or more RAM.

   - Create a partition for Ubuntu on the RAID device. You can use the remaining space if you want to. Format it as ext4 and mount it at /.

1. After installation, update and upgrade your system

	```
	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get update
	```

1. Install latest ubuntu drivers, which includes `nvidia` driver, for correctly displaying GUI e.g.
For instance, by this, gamma issue (super white filter at standard GUI) vanishes.

	```
	sudo ubuntu-drivers install
	```

1. Install the desktop environment:

	```
	sudo apt install ubuntu-desktop
	``` 

1. Install and set up a display manager to manage users and load up the desktop environment sessions. At installation process, select `GDM3` because it refers to the default display manager of GNOME. Alternatively, you can choose `LightDM`.
	```
	sudo apt install lightdm
	``` 

1. Run this command to start the LightDM service with systemctl:

	```
	sudo systemctl start lightdm.service
	```

1. Run this command to start the LightDM service using the service utility:

	```
	sudo service ligthdm start
	```

1. Reboot your system with the reboot

	```
	sudo systemctl reboot -i
	```

1. Login at desktop GUI and test GUI.


## Set up software RAID

After Ubuntu installation, set up the `softraid` specified before.
Detailled steps can be found at `https://ahelpme.com/linux/lvm/ssd-cache-device-to-a-software-raid5-using-lvm2/`.

1. Use `gparted` to create a partition manually on each `sda`.
Consider `unformated` as file system, here.
Further, set up a partition in the NVME SSD device to occupy only 91% of the space
to have a better SSD endurance and in many cases performance.

1. Install lvm2 and enable the lvm2 service

	```
	sudo su
	parted /dev/sda
	(parted) p
	(parted) q
	parted /dev/sdb
	(parted) p
	(parted) q
	parted /dev/sdc
	(parted) p
	(parted) q
	parted /dev/nvme2n1
	(parted) p
	(parted) q
	```

1. Add partitions to the LVM2 (as physical volumes):

	```
	sudo pvcreate /dev/sda1 /dev/sdb1 /dev/sdc1 /dev/nvme2n1p1
	```

1. Create the LVM Volume Group device. The four physical volumes must be in the same group.

	```
	sudo vgcreate VG_storage /dev/sda1 /dev/sdb1 /dev/sdc1 /dev/nvme2n1p1
	```

1. Create the RAID5 device using the three slow hard drive disks and their partitions 
`/dev/sda1`, `/dev/sdb1` and `/dev/sdc1`. 
We want to use all the available space on our slow disks in one logical storage device we use `100%FREE`. 
The name of the logical device is `lv_slow` hinting it consists of slow disks.

	```
	sudo lvcreate --type raid5 -l 100%FREE -I 512 -n lv_slow VG_storage /dev/sda1 /dev/sdb1 /dev/sdc1
	```

1. Create the cache pool logical volume with the name `lv_cache` (to show it’s a fast SSD device). 
Again, we use 100% available space on the physical volume (100% from the partition we’ve used).

	```
	lvcreate --type cache-pool -l 100%FREE -c 4M --cachemode writethrough -n lv_cache VG_storage /dev/nvme2n1
	```

	Consider `writeback` if you focus on performance. 
	This mode delays writing data blocks from the cache back to the origin LV. 
	But this mode will increase performance, but the loss of a cache device can result in lost data. 
	Hence, this mode should only be used if the fast cache is realized as individual `RAID1`.

	Consider `writethrough` instead if you focus on security.
	This mode ensures that any data written will be stored both in the cache and on the origin LV. 
	The loss of a device associated with the cache in this case would not mean the loss of any data.

1. Convert the cache device – the slow device (logical volume `lv_slow`) 
will have a cache device (logical volume `lv_cache`):

	```
	lvconvert --type cache --cachemode writethrough --cachepool VG_storage/lv_cache VG_storage/lv_slow
	```

1. Format and do not miss to include it in the /etc/fstab to mount it automatically on boot.

	```
	mkfs.ext4 /dev/VG_storage/lv_slow
	```

1. Get to know `UUID` of `lv_slow` by

	```
	blkid |grep lv_slow
	```

1. Add `UUID` of `lv_slow` to the `/etc/fstab`, so that it is mounted automatically on boot.
E.g., the entry looks like this:

	```
	UUID=cbf0e33c-8b89-4b7b-b7dd-1a9429db3987 /mnt/storag ext4 defaults,discard,noatime 1 3
	```

1. You are ready to use this cached `RAID1` by mounting it:

	```
	/mnt/storage
	```

## Set-Up Ubuntu AILab configuration

### Install NVIDIA driver

1. Select most current NVIDIA driver from Ubuntu setting menu.

1. Test nvlink of graphic card `0`:

	```
	nvidia-smi nvlink --status -i 0
	```

1. Test nvlink capabilities of graphic card `0`:

	```
	nvidia-smi nvlink --capabilities -i 0
	```

1. Activate `SLI` modus: 

	TBD?

### Install further tools

1. Install net-tools, so that you e.g. can run `ifconfig` to get to know your current IP address.

	```
	apt install net-tools
	```

1. Install CLI gpu monitor:

	```
	sudo apt install nvtop
	```

1. Test this monitoring tool by running it:

```
	nvtop
```

### Install docker engine

The docker engine suits to deal with programs over-the-air.
Detailed installation steps can be found at https://docs.docker.com/engine/install/ubuntu.

#### a) Set up the repository

Before you install Docker Engine for the first time on a new host machine, you need to set up the Docker repository. Afterward, you can install and update Docker from the repository.

1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:

	```
	sudo apt-get update
	
	sudo apt-get install \
		ca-certificates \
		curl \
		gnupg \
		lsb-release
	```

1. Add Docker’s official GPG key:

	```
	sudo mkdir -p /etc/apt/keyrings
	
 	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
	```

1. Use the following command to set up the repository:

	```
	echo \
		"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
		$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	```

#### b) Install Docker Engine

1. Update the apt package index, and install the latest version of Docker Engine, containerd, and Docker Compose, or go to the next step to install a specific version:

	```
	sudo apt-get update
	
	sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
	```

1. Test docker engine installation:

	```
	sudo docker run hello-world
	```

1. Install docker-compose:

	```
	sudo apt install docker-compose
	```

1. Test docker-compose by showing its current version:

	```
	docker-compose --version
	```

### Setting up NVIDIA Container Toolkit

Setting up NVIDIA Container Toolkit so that it functions as driver for docking docker containers to GPUs.
Detailed steps can be found at https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker.

1. Setup the package repository and the GPG key:

	```
	distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
		&& curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
		&& curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
			sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
			sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
	```

1. Install the nvidia-docker2 package (and dependencies) after updating the package listing:

	```
	sudo apt-get update
	
	sudo apt-get install -y nvidia-docker2
	```

1. Restart the Docker daemon to complete the installation after setting the default runtime:
	
	```
	sudo systemctl restart docker
	```

	At this point, a working setup can be tested by running a base CUDA container:
	
	```
	sudo docker run --rm --gpus all nvidia/cuda:11.0.3-base-ubuntu20.04 nvidia-smi
	```
	
	This should result in a console output shown below:
	
```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.51.06    Driver Version: 450.51.06    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla T4            On   | 00000000:00:1E.0 Off |                    0 |
| N/A   34C    P8     9W /  70W |      0MiB / 15109MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

### Install git tools

Git tools suite for dealing with different kinds of repositories.

1. Install git (detailed steps can be found at https://github.com/git-guides/install-git)

	```
	sudo apt-get update

	sudo apt-get install git-all
	```

1. Test git installation by showing current git version:

	```
	git --version
	```

### Install AILab repositories

1. Prepare joint repository storage:

	```
	mkdir /home/$YourUserAccount$/repositories
	
	cd /home/$YourUserAccount$/repositories
	```

1. Clone relevant repositories:

	```
	git clone https://github.com/MarcusGrum/AI-CPS.git
	```