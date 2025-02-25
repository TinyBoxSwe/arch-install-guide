# Installing Arch Linux for Newbies

## 1. Set the Font for Better Visibility
```sh
setfont ter-132b
```

## 2. Identify the Drives

Run one of the following commands to list the drives:
```sh
lsblk
```
or
```sh
fdisk -l
```

You should see something like nvme0nX or sdX. Ensure that the drive you partition is not your installation USB.
## 3. Determine if You're Running BIOS or UEFI
```sh
cat /sys/firmware/efi/fw_platform_size
```

- 64: UEFI mode with 64-bit x64 UEFI.

- 32: UEFI mode with 32-bit IA32 UEFI (limited boot loader choices).

- No such file or directory: BIOS (or CSM) mode.

## 4. Partition the Drive

Assuming you are using `/dev/nvme0n1`:

```sh
fdisk /dev/nvme0n1
```
### Delete Existing Partitions (if any)
```sh
Command (m for help): d
```

Repeat the d command until all existing partitions are deleted.
### Create Partition 1: EFI (for UEFI) or BIOS Boot Partition (for BIOS)
```sh
Command (m for help): n
Partition number (1-128, default 1): Enter ↵
First sector (..., default 2048): Enter ↵
Last sector ...: +256M
```
### Create Partition 2: Main
```sh
Command (m for help): n
Partition number (2-128, default 2): Enter ↵
First sector (..., default ...): Enter ↵
Last sector ...: -4G
```
### Create Partition 3: Swap
```sh
Command (m for help): n
Partition number (3-128, default 3): Enter ↵
First sector (..., default ...): Enter ↵
Last sector ...: Enter ↵
```

### Change Partition Types
```sh
Command (m for help): t
Partition number (1-3, default 1): 1
Partition type or alias (type L to list all): uefi
Command (m for help): t
Partition number (1-3, default 2): 2
Partition type or alias (type L to list all): linux
Command (m for help): t
Partition number (1-3, default 3): 3
Partition type or alias (type L to list all): swap
```
### Write Partitioning to Disk
```sh
Command (m for help): w
```

### 5. Using gdisk for BIOS Systems

If you are on a BIOS system, you should use gdisk to change the partition type of the first partition to "ef02" (BIOS boot partition).
```sh
gdisk /dev/nvme0n1
```
#### Use t to change the type of the first partition to ef02.

#### Write changes with w.

### 6. Format the Partitions
#### For UEFI Systems
```sh
mkfs.fat -F32 /dev/nvme0n1p1  # EFI partition
mkfs.ext4 /dev/nvme0n1p2      # Main partition
mkswap /dev/nvme0n1p3         # Swap partition
swapon /dev/nvme0n1p3         # Enable swap
```
#### For BIOS Systems
```sh
mkfs.ext4 /dev/nvme0n1p1      # Main partition
mkswap /dev/nvme0n1p2         # Swap partition
swapon /dev/nvme0n1p2         # Enable swap
```

#### Note: For BIOS systems, you do not need to format the BIOS boot partition (ef02) with mkfs. It remains unformatted.

### 7. Mount the Partitions
#### For UEFI Systems
```sh
mount /dev/nvme0n1p2 /mnt
mkdir -p /mnt/boot/efi
mount /dev/nvme0n1p1 /mnt/boot/efi
```
#### For BIOS Systems
```sh
mount /dev/nvme0n1p1 /mnt
```
### 8. Install essential packages into new filesystem and generate fstab

```sh
pacstrap -i /mnt base linux linux-firmware sudo vim
genfstab -U -p /mnt > /mnt/etc/fstab
```

#### Good-To-Know:
`genfstab`: This command generates the fstab (file system table) file for your new system.

-U: Uses UUIDs to identify partitions (ensures consistency even if device names change).

-p: Preserves the mount options specified during the installation process.

The fstab file tells your system how to mount partitions (e.g., root, boot, home) at boot time. Without it, your system won't know where to find its filesystems.

### 9. Configure the new system
```sh
arch-chroot /mnt

echo yourhostname > /etc/hostname # replace your host name with your hostname

vim /etc/hosts
    127.0.0.1 localhost
    ::1       localhost
    127.0.1.1 yourhostname

useradd -m -G wheel,storage,power,audio,video -s /bin/bash yourusername
# -G, --groups GROUPS           list of supplementary groups of the new account
# -m, --create-home             create the user's home directory
# -s, --shell SHELL             login shell of the new account
```
#### If you wish to use another shell you can download it using `pacman`
```sh
pacman -S zsh                   # replace /bin/bash with /bin/zsh

passwd root

passwd yourusername

pacman -S grub efibootmgr

visudo
# uncomment following line in file (almost at the very bottom)
# %wheel ALL=(ALL) ALL

grub-install

grub-mkconfig -o /boot/grub/grub.cfg

pacman -S dhcpcd networkmanager resolvconf
# dhcpcd automatically configures IP addresses via DHCP for basic network setup.
# networkmanager manages wired, wireless, and VPN connections with user-friendly tools.
# resolveconf Dynamically updates DNS settings for proper domain name resolution.

systemctl enable dhcpcd
systemctl enable NetworkManager
systemctl enable systemd-resolved

exit
umount /mnt/boot/efi
umount /mnt
reboot
```

### 10. More advanced config

I recommend installing `git` and basic arch and aur tools.
```sh
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

I personally use [JaKooLit's config](https://github.com/JaKooLit/Arch-Hyprland) for Hyprland.

For gnome:
```sh
sudo pacman -S gnome
sudo systemctl enable gdm
reboot
```
For kde-plasma:
```sh
sudo pacman -S plasma-meta
sudo systemctl enable sddm
reboot
```

After rebooting you can add windows to your grub boot menu by installing and running:
```sh
sudo pacman -Sc
sudo pacman -Syu
sudo pacman -S os-prober ntfs-3g
mkdir /mnt/windows
sudo mount -t ntfs-3g /dev/nvme0n1pX /mnt/windows
update-grub
```

If windows is hibernated reboot into windows using the boot menu and shut it down in the cmd using: 
```sh
shutdown /s /f /t 0
```

If for some reason you don't see windows in your boot options you can boot into Arch and run the following:
```sh
sudo efibootmgr
sudo efibootmgr --bootnext XXXX # only the four numbers of the windows partition are needed
```
