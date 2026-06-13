Identify your partition names using lsblk. For this guide, we will assume:
•	EFI Partition (/dev/sda1)
•	Root Partition (/dev/sda2)
•	Swap Partition (/dev/sda3)
Let's format your $40\text{ GB}$ EXT4 root partition and mount everything under /mnt.
Bash
# Format the root partition to ext4
sudo mkfs.ext4 /dev/sda2

# Mount the root partition
sudo mount /dev/sda2 /mnt

# Create the EFI mount point and mount it
# Note: systemd-boot prefers EFI mounted at /boot for clean automatic kernel discovery
sudo mkdir -p /mnt/boot
sudo mount /dev/sda1 /mnt/boot

# Activate swap
sudo swapon /dev/sda3
Step 2: Bootstrap the Minimal Base
We'll use the --variant=minbase flag. This pulls down an ultra-lean environment: just apt, bash, and core utilities. No GUI, no bloat.
Bash
# Install debootstrap on your live USB environment if missing
sudo apt update && sudo apt install -y debootstrap

# Bootstrap the base system into /mnt
sudo debootstrap --variant=minbase noble /mnt http://archive.ubuntu.com/ubuntu/
Step 3: Mount Virtual Filesystems & Chroot
To configure your new OS from inside the Live USB, we must bind the kernel's virtual filesystems and enter the environment.
Bash
sudo mount --make-rslave /mnt
sudo mount --rbind /proc /mnt/proc
sudo mount --rbind /sys /mnt/sys
sudo mount --rbind /dev /mnt/dev
sudo mount --rbind /run /mnt/run

# Copy your live environment's DNS configuration so apt works inside
sudo cp /etc/resolv.conf /mnt/etc/resolv.conf

# Chroot into your new system
sudo chroot /mnt /bin/bash
Step 4: System Configuration (Inside Chroot)
Now you are safely inside your isolated Ubuntu minimal environment. Let's initialize the shell paths.
Bash
source /etc/profile
export HISTFILE=/root/.bash_history
1. Configure Package Repositories
Since minbase gives you a barebones package list, update /etc/apt/sources.list to include essential repositories:
Bash
cat <<EOF > /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu/ noble main universe
deb http://archive.ubuntu.com/ubuntu/ noble-updates main universe
deb http://security.ubuntu.com/ubuntu/ noble-security main universe
EOF

apt update
2. Set Locales, Timezone, and Hostname
Bash
# Install critical configuration tools
apt install -y locales tzdata

# Generate your language environment
locale-gen en_US.UTF-8
update-locale LANG=en_US.UTF-8

# Set Timezone (Change UTC to your region, e.g., America/New_York)
ln -sf /usr/share/zoneinfo/UTC /etc/localtime
dpkg-reconfigure -f noninteractive tzdata

# Give your system a identity
echo "ubuntu-minimal" > /etc/hostname
echo "127.0.0.1 localhost" >> /etc/hosts
echo "127.1.1.1 ubuntu-minimal" >> /etc/hosts
3. Configure the File System Table (/etc/fstab)
Run blkid in a separate terminal window to find your partition UUIDs. Populate /etc/fstab using those exact UUIDs to ensure stable mounting across reboots:
Bash
cat <<EOF > /etc/fstab
# /dev/sda2 (Root)
UUID=your-root-uuid-here / ext4 defaults 0 1

# /dev/sda1 (EFI)
UUID=your-efi-uuid-here /boot vfat defaults 0 2

# /dev/sda3 (Swap)
UUID=your-swap-uuid-here none swap sw 0 0
EOF
Step 5: Install Kernel and Core Network Utilities
We need to fetch the generic kernel, the systemd-boot hooks, and standard networking.
Bash
apt install -y linux-image-generic systemd-resolved systemd-boot netplan.io
(If a legacy GRUB prompt pops up during this phase asking you where to target an installation, you can safely cancel or ignore it).
Step 6: Configure systemd-boot (Instead of GRUB)
Initialize systemd-boot directly onto your $1\text{ GB}$ EFI partition.
Bash
bootctl install
1. Create Loader Preferences
Configure the main boot menu behavior:
Bash
cat <<EOF > /boot/loader/loader.conf
default ubuntu.conf
timeout 4
console-mode max
editor no
EOF
2. Create the Entry File
Check /boot to verify the exact string of the kernel version you just installed (e.g., 6.8.0-31-generic).
Bash
ls /boot
Now, create the specific entry file /boot/loader/entries/ubuntu.conf:
Bash
cat <<EOF > /boot/loader/entries/ubuntu.conf
title   Ubuntu Minimal (CLI)
linux   /vmlinuz-YOUR-KERNEL-VERSION-generic
initrd  /initrd.img-YOUR-KERNEL-VERSION-generic
options root=UUID=your-root-uuid-here rw quiet
EOF
Crucial Note: Because your EFI partition is mounted right at /boot, the kernel binaries sit directly at the root of that drive. Ensure the paths to linux and initrd in this file start with a single forward slash / relative to the EFI mount, exactly as shown above.
Step 7: User Management & Ethernet Setup
To fulfill your request for strict administrative separation, we will explicitly set a strong password for root (the system administrator) and create a standard user without adding them to the sudo group.
1. Set Root and Standard User Passwords
Bash
# 1. Set the Administrator (Root) Password
passwd

# 2. Create your everyday non-privileged user
# (Do NOT add this user to 'sudo' or 'admin' groups)
adduser yourusername
2. Configure Ethernet (Netplan)
Because your hardware relies on an ethernet cable, we will write a declarative Netplan rule using a broad wildcard pattern (e*) to catch your ethernet card (whether its name is eth0, enp3s0, etc.) and command it to lease a local IP address dynamically:
Bash
cat <<EOF > /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ethernet-legacy:
      match:
        name: e*
      dhcp4: true
EOF

# Enable the systemd network configuration daemon
systemctl enable systemd-resolved
Step 8: Finalize and Reboot
Your custom minimal platform is isolated and complete. Let's dismount cleanly.
Bash
# Step out of the chroot jail
exit

# Recursively unmount the virtual directory trees
sudo umount -R /mnt

# Reboot into your new OS
sudo reboot


# Common Packages for Android Development
apt-get install --no-install-recommends adb bc bison build-essential ca-certificates ccache curl fastboot flex git-lfs gnupg gperf g++-multilib imagemagick libc6-dev-i386 libelf-dev libgl1-mesa-dev libncurses-dev libssl-dev libxml2 libxml2-utils libx11-dev lib32ncurses-dev lib32readline-dev lib32z1-dev lzop lz4 nano pngcrush protobu-compiler python3 python3-protobuf repo rsync schedtool squashfs-tools unzip wget xsltproc x11proto-coro-dev zip zlib1g-dev
