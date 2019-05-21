# Arch Linux install guide

This is personal guide on how to install ArchLinux.
Official reference - [ArchWiki Installation Guide](https://wiki.archlinux.org/index.php/Installation_Guide)

## Before you begin

### Enter BIOS with and configure:
* "System Configuration" > "SATA Operation": "AHCI"
* "Secure Boot" > "Secure Boot Enable": "Disabled"

## Installation

### Connect to Internet
```sh
wifi-menu
```

### Sync clock
```sh
timedatectl set-ntp true
```

### Partitioning
Create three partisions:
* 512MB EFI partition
* 4GB Swap partition
* 100% Linux partiton
```sh
cfdisk /dev/nvme0n1
mkfs.fat -F32 /dev/nvme0n1p1
mkswap /dev/nvme0n1p2
```


# Setup the encryption of the system

cryptsetup luksFormat /dev/nvme0n1p2

cryptsetup open /dev/nvme0n1p2 luks



# Create LVM partitions

# This creates partitions for root and /home, no /swap.

pvcreate /dev/mapper/luks

vgcreate vg0 /dev/mapper/luks

lvcreate -L 100G vg0 --name root

lvcreate -l +80%FREE vg0 --name home # 80% leaves some space for snapshots



mkfs.ext4 /dev/mapper/vg0-root

mkfs.ext4 /dev/mapper/vg0-home



# Mount the new system

# /mnt is the system to be installed

mount /dev/mapper/vg0-root /mnt



mkdir /mnt/home

mount /dev/mapper/vg0-home /mnt/home



mkdir /mnt/boot

mount /dev/nvme0n1p1 /mnt/boot



# Install the base system plus a few packages

pacstrap /mnt base zsh vim git sudo efibootmgr wpa_supplicant dialog iw



# Generate fstab

genfstab -L /mnt >> /mnt/etc/fstab



# Verify and adjust /mnt/etc/fstab

# Change relatime on all non-boot partitions to noatime (reduces wear if using an SSD)



# Enter the new system

arch-chroot /mnt



# Setup time

ln -s /usr/share/zoneinfo/Europe/Zurich /etc/localtime

hwclock --systohc



# Generate required locales

vi /etc/locale.gen # Uncomment desired locales, e.g. "en_US.UTF-8", "de_CH.UTF-8"

locale-gen



# Set desired locale

echo 'LANG=en_US.UTF-8' > /etc/locale.conf



# Set desired keymap and font

echo 'KEYMAP=us' > /etc/vconsole.conf

echo 'FONT=latarcyrheb-sun32' >> /etc/vconsole.conf



# Set the hostname

echo '<hostname>' > /etc/hostname

# Add to hosts

echo '127.0.1.1 <hostname>.localdomain <hostname>' >> /etc/hosts



# Set password for root

passwd



# Add real user

useradd -m -g users -G wheel -s /bin/zsh <username>

passwd <username>

echo '<username> ALL=(ALL) ALL' > /etc/sudoers.d/<username>



# Configure mkinitcpio with modules needed for the initrd image

vi /etc/mkinitcpio.conf

# Add 'ext4 dm_snapshot' to MODULES

# Change: HOOKS="base systemd autodetect modconf block keyboard sd-vconsole sd-encrypt sd-lvm2 filesystems"



# Regenerate initrd image

mkinitcpio -p linux



# Setup systemd-boot

bootctl --path=/boot install



# Enable Intel microcode updates

pacman -S intel-ucode



# Create bootloader entry

# Get luks-uuid with: `cryptsetup luksUUID /dev/nvme0n1p2`

---

/boot/loader/entries/arch.conf

---

title Arch Linux

linux /vmlinuz-linux

initrd /intel-ucode.img

initrd /initramfs-linux.img

options luks.uuid=<uuid> luks.name=<uuid>=luks root=/dev/mapper/vg0-root rw

---



# Set default bootloader entry

---

/boot/loader/loader.conf

---

default arch

---



# Exit and reboot

exit

reboot