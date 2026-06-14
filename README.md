# Identify your partition names using lsblk. For this guide, we will assume:
• EFI Partition (/dev/sda1)
•	Root Partition (/dev/sda2)

# Format the root partition to ext4
•	sudo su
•	mkfs.ext4 /dev/sda2

# Mount the root partition
•	mount /dev/sda2 /mnt

# Create the EFI mount point and mount it
•	mkdir -p /mnt/boot
•	mount /dev/sda1 /mnt/boot

# Check using lsblk that your partitions are mounted as intended

# Step 2: Bootstrap the Minimal Base
•	apt-get update
•	apt-get install -y debootstrap

# Bootstrap the base system into /mnt
•	debootstrap --variant=minbase resolute /mnt https://mirror.twds.com.tw/ubuntu-ports/

# Mount Virtual Filesystems & Chroot
•	mount --bind /dev /mnt/dev
•	mount --bind /dev/pts /mnt/dev/pts
•	mount --bind /run /mnt/run
•	mount -t proc proc /mnt/proc
•	mount -t sysfs sysfs /mnt/sysfs
•	cp /etc/resolv.conf /mnt/etc/resolv.conf
•	chroot /mnt /bin/bash

# Configure the File System Table (/etc/fstab)
nano /mnt/etc/fstab
•	UUID=your-root-uuid-here / ext4 defaults 0 1
•	UUID=your-efi-uuid-here /boot vfat defaults 0 2
•	UUID=your-swap-uuid-here none swap sw 0 0

# Configure Package Repositories 
•	nano /etc/apt/sources.list
•	deb https://mirror.twds.com.tw/ubuntu-ports/ resolute main restrictred universe multiverse
•	deb https://mirror.twds.com.tw/ubuntu-ports/ resolute-backports main restrictred universe multiverse
•	deb https://mirror.twds.com.tw/ubuntu-ports/ resolute-security main restrictred universe multiverse
•	deb https://mirror.twds.com.tw/ubuntu-ports/ resolute-updates main restrictred universe multiverse
•	apt update

# Set Locales, Timezone, and Hostname
•	apt install --no-install-recommends locales tzdata
•	dpkg-reconfigure locales
•	dpkg-reconfigure tzdata

# Give your system a identity
•	nano /etc/hostname
•	ubuntu
•	nano /etc/hosts
•	127.0.0.1 localhost
•	127.1.1.1 ubuntu
•	::1 localhost ip6-localhost ip6-loopback

# Install Kernel and Core Network Utilities
•	apt install -y linux-image-generic systemd-boot systemd-resolved systemd-networkd efibootmgr
•	bootctl install

# Create Loader &
•	nano /boot/loader/loader.conf
default ubuntu.conf
timeout 4
console-mode max
editor no
•	Check /boot to verify the exact string of the kernel version you just installed (e.g., 6.8.0-31-generic).
•	ls /boot
•	nano /boot/loader/entries/ubuntu.conf
title Ubuntu Minimal (CLI)
linux /vmlinuz-YOUR-KERNEL-VERSION-generic
initrd /initrd.img-YOUR-KERNEL-VERSION-generic
options root=UUID=your-root-uuid-here rw quiet

# User Management & Ethernet Setup
•	passwd
•	useradd -m yourusername
•	passwd yourusername

2. Configure Ethernet
•	nano /etc/systemd/network/dhcp.network
[Match]
Name=en*

[Network]
DHCP=yes
•	systemctl enable systemd-networkd
•	systemctl enable systemd-resolved

# Finalize and Reboot
•	exit
•	sudo umount -R /mnt
•	sudo reboot

# Common Packages for Android Development
apt-get install --no-install-recommends adb bc bison build-essential ca-certificates ccache curl fastboot flex git-lfs gnupg gperf g++-multilib imagemagick libc6-dev-i386 libelf-dev libgl1-mesa-dev libncurses-dev libssl-dev libxml2 libxml2-utils libx11-dev lib32ncurses-dev lib32readline-dev lib32z1-dev lzop lz4 nano pngcrush protobu-compiler python3 python3-protobuf repo rsync schedtool squashfs-tools unzip wget xsltproc x11proto-coro-dev zip zlib1g-dev
