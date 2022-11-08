# Arch Linux install

These are some personal notes on how to install ArchLinux.
Please refer to the official [Installation Guide](https://wiki.archlinux.org/index.php/Installation_Guide) for a more thorough documentation.

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

### Optional: Setup the encryption of the system

```sh
cryptsetup luksFormat /dev/nvme0n1p2
cryptsetup open /dev/nvme0n1p2 luks
```

### Create LVM partitions

#### This creates partitions for root and /home, no /swap.

```sh
pvcreate /dev/mapper/luks
vgcreate vg0 /dev/mapper/luks
lvcreate -L 100G vg0 --name root
lvcreate -l +80%FREE vg0 --name home # 80% leaves some space for snapshots
```

#### Format new partitions
```sh
mkfs.ext4 /dev/mapper/vg0-root
mkfs.ext4 /dev/mapper/vg0-home
```


### Mount the newly created partitions
```sh
mount /dev/mapper/vg0-root /mnt

mkdir /mnt/home
mount /dev/mapper/vg0-home /mnt/home

mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```


### Install the base system plus a few packages

```sh
pacstrap /mnt base linux linux-firmware zsh vim git sudo efibootmgr wpa_supplicant dialog iw
```


#### Generate fstab
```sh
genfstab -L /mnt >> /mnt/etc/fstab
```

#### Verify and adjust /mnt/etc/fstab
#### Change relatime on all non-boot partitions to noatime (reduces wear if using an SSD)
```sh
sudo vim /mnt/etc/fstab
```


## Enter the new system
```sh
arch-chroot /mnt
```

### Setup time
```sh
ln -s /usr/share/zoneinfo/Europe/Zurich /etc/localtime
hwclock --systohc
```

### Generate required locales
```sh
vim /etc/locale.gen # Uncomment desired locales, e.g. "en_US.UTF-8", "de_CH.UTF-8"
locale-gen
```

#### Set desired locale
```sh
echo 'LANG=en_US.UTF-8' > /etc/locale.conf
```

#### Set desired keymap and font
```sh
echo 'KEYMAP=us' > /etc/vconsole.conf
echo 'FONT=latarcyrheb-sun32' >> /etc/vconsole.conf
```


### Set the hostname
```sh
echo 'arch' > /etc/hostname
```

### Add to hosts
```sh
echo '127.0.1.1 arch.localdomain arch' >> /etc/hosts
```

### Set password for root
```sh
passwd
```

### Create user and add it to sudoers
```sh
useradd -m -g users -G wheel -s /bin/zsh <username>
passwd <username>
echo '<username> ALL=(ALL) ALL' > /etc/sudoers.d/<username>
```

### Configure mkinitcpio with modules needed for the initrd image
```sh
vi /etc/mkinitcpio.conf # Add 'ext4 dm_snapshot' to MODULES
                        # Change: HOOKS="base systemd autodetect modconf block keyboard sd-vconsole sd-encrypt sd-lvm2 filesystems"
```


### Regenerate initrd image
```sh
mkinitcpio -p linux
```

### Setup systemd-boot
```sh
bootctl --path=/boot install
```

### Enable Intel microcode updates
```sh
pacman -S intel-ucode
```

### Create bootloader entry

#### Optional Get luks-uuid with: `cryptsetup luksUUID /dev/nvme0n1p2`

```sh
vim /boot/loader/entries/arch.conf
```

```sh
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options root=/dev/mapper/vg0-root rw
# options luks.uuid=<uuid> luks.name=<uuid>=luks root=/dev/mapper/vg0-root rw
```

### Set default bootloader entry
```sh
vim /boot/loader/loader.conf
```

```sh
default arch
```

## Exit and reboot
```sh
exit
reboot
```