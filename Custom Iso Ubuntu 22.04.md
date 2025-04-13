# Creating a Custom Ubuntu Forensic Live ISO Based on Ubuntu 22.04 (Jammy)

This guide outlines the process of creating a custom Ubuntu 22.04 (Jammy) live ISO specifically configured for digital forensics work. The resulting ISO is designed to run only in live mode (without installation capability) and comes pre-loaded with essential forensic tools.

## Features

- Based on Ubuntu 22.04 LTS (Jammy Jellyfish)
- Live-only operation (no installation option)
- Lightweight XFCE desktop environment
- Pre-installed forensic tools
- Write-protection mechanisms to prevent accidental modifications
- Desktop shortcuts for common forensic tasks

## Prerequisites

Before starting, ensure you have a working Ubuntu-based system with sudo privileges and at least 20GB of free disk space.

## Step 1: Set up the build environment

First, install the necessary packages:

```shell
sudo apt-get install \
    binutils \
    debootstrap \
    squashfs-tools \
    xorriso \
    grub-pc-bin \
    grub-efi-amd64-bin \
    mtools
```

Create a working directory:

```shell
mkdir $HOME/forensic-ubuntu-live
```

## Step 2: Bootstrap Ubuntu 22.04 (Jammy)

Bootstrap the base system:

```shell
sudo debootstrap \
    --arch=amd64 \
    --variant=minbase \
    jammy \
    $HOME/forensic-ubuntu-live/chroot \
    http://us.archive.ubuntu.com/ubuntu/
```

Configure external mount points:

```shell
sudo mount --bind /dev $HOME/forensic-ubuntu-live/chroot/dev
sudo mount --bind /run $HOME/forensic-ubuntu-live/chroot/run
```

## Step 3: Configure the chroot environment

Enter the chroot environment:

```shell
sudo chroot $HOME/forensic-ubuntu-live/chroot
```

Configure mount points, home, and locale:

```shell
mount none -t proc /proc
mount none -t sysfs /sys
mount none -t devpts /dev/pts
export HOME=/root
export LC_ALL=C
```

Set a custom hostname:

```shell
echo "forensic-ubuntu-live" > /etc/hostname
```

Configure apt sources for Jammy (22.04):

```shell
cat <<EOF > /etc/apt/sources.list
deb http://us.archive.ubuntu.com/ubuntu/ jammy main restricted universe multiverse
deb-src http://us.archive.ubuntu.com/ubuntu/ jammy main restricted universe multiverse

deb http://us.archive.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src http://us.archive.ubuntu.com/ubuntu/ jammy-security main restricted universe multiverse

deb http://us.archive.ubuntu.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src http://us.archive.ubuntu.com/ubuntu/ jammy-updates main restricted universe multiverse
EOF
```

Update package indexes:

```shell
apt-get update
```

Install systemd:

```shell
apt-get install -y libterm-readline-gnu-perl systemd-sysv
```

Configure machine-id and divert:

```shell
dbus-uuidgen > /etc/machine-id
ln -fs /etc/machine-id /var/lib/dbus/machine-id

dpkg-divert --local --rename --add /sbin/initctl
ln -s /bin/true /sbin/initctl
```

Upgrade packages:

```shell
apt-get -y upgrade
```

## Step 4: Install packages needed for Live System

Install base packages:

```shell
apt-get install -y \
    sudo \
    ubuntu-standard \
    casper \
    lupin-casper \
    discover \
    laptop-detect \
    os-prober \
    network-manager \
    resolvconf \
    net-tools \
    wireless-tools \
    wpagui \
    locales \
    grub-common \
    grub-gfxpayload-lists \
    grub-pc \
    grub-pc-bin \
    grub2-common
```

Install the Linux kernel:

```shell
apt-get install -y --no-install-recommends linux-generic
```

## Step 5: Install desktop environment (lightweight XFCE)

For a forensic live system, we'll use a lightweight desktop environment:

```shell
apt-get install -y \
    plymouth-theme-ubuntu-logo \
    xfce4 \
    xfce4-goodies \
    xubuntu-artwork \
    xubuntu-default-settings \
    lightdm
```

## Step 6: Install forensic tools

Install the forensic tools:

```shell
apt-get install -y \
    nmap \
    tcpdump \
    wireshark \
    exiftool \
    binwalk \
    dd \
    dc3dd \
    fdisk \
    mount \
    strings \
    grep \
    xxd \
    bulk-extractor \
    md5sum \
    hashdeep \
    sleuthkit \
    foremost \
    autopsy \
    testdisk \
    ghex \
    gnupg \
    cryptsetup \
    chntpw \
    ophcrack \
    rkhunter \
    clamav \
    scalpel \
    recoverjpeg \
    photorec
```

For Volatility and LiME, which may not be in the standard repositories:

```shell
# First install dependencies
apt-get install -y \
    git \
    python3 \
    python3-pip \
    python3-dev \
    build-essential \
    autoconf \
    automake \
    dkms \
    linux-headers-generic

# Install volatility through pip
pip3 install volatility3

# Clone and install LiME
git clone https://github.com/504ensicsLabs/LiME.git /opt/LiME
cd /opt/LiME/src
make

# Create a script to rebuild the module when needed
cat <<EOF > /usr/local/bin/build-lime-module
#!/bin/bash
cd /opt/LiME/src
make
EOF
chmod +x /usr/local/bin/build-lime-module
```

## Step 7: Configure the system for forensic work

Create a forensic user with sudo privileges:

```shell
useradd -m forensic -s /bin/bash
passwd forensic  # Set a password for the forensic user
usermod -aG sudo forensic
```

Configure automount to be disabled:

```shell
mkdir -p /etc/udev/rules.d/
cat <<EOF > /etc/udev/rules.d/99-no-automount.rules
# Disable automounting
SUBSYSTEM=="block", ENV{UDISKS_AUTO}="0"
EOF
```

Create a script to mount drives as read-only:

```shell
cat <<EOF > /usr/local/bin/mount-ro
#!/bin/bash
# Script to mount a device as read-only
if [ \$# -ne 2 ]; then
    echo "Usage: \$0 <device> <mountpoint>"
    exit 1
fi

DEVICE=\$1
MOUNTPOINT=\$2

# Create mountpoint if it doesn't exist
if [ ! -d "\$MOUNTPOINT" ]; then
    mkdir -p "\$MOUNTPOINT"
fi

# Mount as read-only
mount -o ro "\$DEVICE" "\$MOUNTPOINT"
echo "Mounted \$DEVICE at \$MOUNTPOINT as read-only"
EOF
chmod +x /usr/local/bin/mount-ro
```

Create forensic tools desktop shortcuts:

```shell
mkdir -p /etc/skel/Desktop/forensic-tools

cat <<EOF > /etc/skel/Desktop/forensic-tools/wireshark.desktop
[Desktop Entry]
Name=Wireshark
Comment=Network traffic analyzer
Exec=wireshark
Icon=wireshark
Terminal=false
Type=Application
Categories=Network;
EOF

cat <<EOF > /etc/skel/Desktop/forensic-tools/terminal.desktop
[Desktop Entry]
Name=Terminal
Comment=Use the command line
Exec=xfce4-terminal
Icon=utilities-terminal
Terminal=false
Type=Application
Categories=System;
EOF

chmod +x /etc/skel/Desktop/forensic-tools/*.desktop
```

Create a welcome script:

```shell
cat <<EOF > /usr/local/bin/forensic-welcome
#!/bin/bash
zenity --info --title="Forensic Ubuntu Live" --text="Welcome to Forensic Ubuntu Live.\n\nThis is a specialized Ubuntu distribution for digital forensics.\n\nIMPORTANT: Always mount devices as read-only using the mount-ro script." --width=400
EOF
chmod +x /usr/local/bin/forensic-welcome

# Add it to autostart
mkdir -p /etc/xdg/autostart
cat <<EOF > /etc/xdg/autostart/forensic-welcome.desktop
[Desktop Entry]
Name=Forensic Welcome
Comment=Display welcome message
Exec=/usr/local/bin/forensic-welcome
Terminal=false
Type=Application
Categories=System;
EOF
```

## Step 8: Clean up the chroot environment

```shell
truncate -s 0 /etc/machine-id

rm /sbin/initctl
dpkg-divert --rename --remove /sbin/initctl

apt-get clean
rm -rf /tmp/* ~/.bash_history

umount /proc
umount /sys
umount /dev/pts

export HISTSIZE=0
exit
```

## Step 9: Unbind mount points

```shell
sudo umount $HOME/forensic-ubuntu-live/chroot/dev
sudo umount $HOME/forensic-ubuntu-live/chroot/run
```

## Step 10: Create the CD image directory and populate it

```shell
cd $HOME/forensic-ubuntu-live
mkdir -p image/{casper,isolinux,install}

sudo cp chroot/boot/vmlinuz-**-**-generic image/casper/vmlinuz
sudo cp chroot/boot/initrd.img-**-**-generic image/casper/initrd

sudo cp chroot/boot/memtest86+.bin image/install/memtest86+

wget --progress=dot https://www.memtest86.com/downloads/memtest86-usb.zip -O image/install/memtest86-usb.zip
unzip -p image/install/memtest86-usb.zip memtest86-usb.img > image/install/memtest86
rm -f image/install/memtest86-usb.zip
```

## Step 11: Configure GRUB menu

```shell
cd $HOME/forensic-ubuntu-live
touch image/ubuntu

# Create a GRUB configuration with only live boot options
cat <<EOF > image/isolinux/grub.cfg
search --set=root --file /ubuntu

insmod all_video

set default="0"
set timeout=30

menuentry "Boot Forensic Ubuntu Live" {
   linux /casper/vmlinuz boot=casper nopersistent toram quiet splash ---
   initrd /casper/initrd
}

menuentry "Boot Forensic Ubuntu Live (nomodeset)" {
   linux /casper/vmlinuz boot=casper nopersistent toram quiet splash nomodeset ---
   initrd /casper/initrd
}

menuentry "Boot Forensic Ubuntu Live (verbose mode)" {
   linux /casper/vmlinuz boot=casper nopersistent toram ---
   initrd /casper/initrd
}

menuentry "Check disc for defects" {
   linux /casper/vmlinuz boot=casper integrity-check quiet splash ---
   initrd /casper/initrd
}

menuentry "Test memory Memtest86+ (BIOS)" {
   linux16 /install/memtest86+
}

menuentry "Test memory Memtest86 (UEFI, long load time)" {
   insmod part_gpt
   insmod search_fs_uuid
   insmod chain
   loopback loop /install/memtest86
   chainloader (loop,gpt1)/efi/boot/BOOTX64.efi
}
EOF
```

## Step 12: Create manifest

```shell
cd $HOME/forensic-ubuntu-live
sudo chroot chroot dpkg-query -W --showformat='${Package} ${Version}\n' | sudo tee image/casper/filesystem.manifest
sudo cp -v image/casper/filesystem.manifest image/casper/filesystem.manifest-desktop
```

## Step 13: Compress the chroot

```shell
cd $HOME/forensic-ubuntu-live
sudo mksquashfs chroot image/casper/filesystem.squashfs
printf $(sudo du -sx --block-size=1 chroot | cut -f1) > image/casper/filesystem.size
```

## Step 14: Create diskdefines

```shell
cd $HOME/forensic-ubuntu-live
cat <<EOF > image/README.diskdefines
#define DISKNAME  Forensic Ubuntu Live
#define TYPE  binary
#define TYPEbinary  1
#define ARCH  amd64
#define ARCHamd64  1
#define DISKNUM  1
#define DISKNUM1  1
#define TOTALNUM  0
#define TOTALNUM0  1
EOF
```

## Step 15: Create ISO Image (BIOS + UEFI)

```shell
cd $HOME/forensic-ubuntu-live/image

# Create a grub UEFI image
grub-mkstandalone \
   --format=x86_64-efi \
   --output=isolinux/bootx64.efi \
   --locales="" \
   --fonts="" \
   "boot/grub/grub.cfg=isolinux/grub.cfg"

# Create a FAT16 UEFI boot disk image
(
   cd isolinux && \
   dd if=/dev/zero of=efiboot.img bs=1M count=10 && \
   sudo mkfs.vfat efiboot.img && \
   LC_CTYPE=C mmd -i efiboot.img efi efi/boot && \
   LC_CTYPE=C mcopy -i efiboot.img ./bootx64.efi ::efi/boot/
)

# Create a grub BIOS image
grub-mkstandalone \
   --format=i386-pc \
   --output=isolinux/core.img \
   --install-modules="linux16 linux normal iso9660 biosdisk memdisk search tar ls" \
   --modules="linux16 linux normal iso9660 biosdisk search" \
   --locales="" \
   --fonts="" \
   "boot/grub/grub.cfg=isolinux/grub.cfg"

# Combine a bootable Grub cdboot.img
cat /usr/lib/grub/i386-pc/cdboot.img isolinux/core.img > isolinux/bios.img

# Generate md5sum.txt
sudo /bin/bash -c "(find . -type f -print0 | xargs -0 md5sum | grep -v -e 'md5sum.txt' -e 'bios.img' -e 'efiboot.img' > md5sum.txt)"

# Create the ISO
sudo xorriso \
   -as mkisofs \
   -iso-level 3 \
   -full-iso9660-filenames \
   -volid "Forensic Ubuntu Live" \
   -output "../forensic-ubuntu-live.iso" \
   -eltorito-boot boot/grub/bios.img \
      -no-emul-boot \
      -boot-load-size 4 \
      -boot-info-table \
      --eltorito-catalog boot/grub/boot.cat \
      --grub2-boot-info \
      --grub2-mbr /usr/lib/grub/i386-pc/boot_hybrid.img \
   -eltorito-alt-boot \
      -e EFI/efiboot.img \
      -no-emul-boot \
   -append_partition 2 0xef isolinux/efiboot.img \
   -m "isolinux/efiboot.img" \
   -m "isolinux/bios.img" \
   -graft-points \
      "/EFI/efiboot.img=isolinux/efiboot.img" \
      "/boot/grub/bios.img=isolinux/bios.img" \
      "."
```

## Step 16: Make a bootable USB image

```shell
sudo dd if=forensic-ubuntu-live.iso of=/dev/sdX status=progress oflag=sync
```
(Replace `/dev/sdX` with your actual USB device - BE CAREFUL to select the correct device!)

## Forensic Tools Included

This custom Ubuntu ISO includes the following forensic tools:

- **Network Forensics**: nmap, tcpdump, wireshark
- **Memory Forensics**: volatility3, LiME (Linux Memory Extractor)
- **File Analysis**: exiftool, binwalk, strings, grep, xxd, bulk_extractor
- **Disk Forensics**: dd, dc3dd, fdisk, mount, testdisk, photorec
- **File System Analysis**: sleuthkit, autopsy
- **Hashing Tools**: md5sum, hashdeep
- **Additional Tools**: foremost, ghex (hex editor), cryptsetup, chntpw, rkhunter, clamav, scalpel, recoverjpeg

## Using the Live System

1. Boot from the USB drive or DVD
2. Select the appropriate boot option
3. Use the provided tools for forensic analysis
4. Remember to always mount drives as read-only using the provided mount-ro script

## Best Practices for Forensic Analysis

1. Never mount a suspect drive with write permissions
2. Always create and verify checksums of evidence
3. Document all steps taken during the investigation
4. Maintain chain of custody for all evidence
5. Use the included forensic tools to minimize changes to evidence