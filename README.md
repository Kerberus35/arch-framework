# Framework AMD Arch setup
Framework AMD Arch BTRFS snapshots systemd LUKS KDE Wayland

(the guide is incomplete but can be used as a general overview)

## Make ISO
https://archlinux.org/download/

## SSH access during set-up

```sh
passwd
systemctl start sshd
ip addr
```
Connect to IP

## Partitioning

GUID Partition Table (GPT) 

| Mount point | Partition name | Partition type | Bootable flags |           Size          |
| :---------: | :------------: | :------------: | :------------: | :---------------------: |
| /boot       | /dev/nvme0n1p1 | EFI System     | Yes						 | 1 Gb 									 |
| /					  | /dev/nvme0n1p2 | Linux LVM      | No						 | Remainder of the device |

Swap will be on BTRFS subvolume according to official guide

```
fdisk -l
cfdisk /dev/nvme0n1
```
And create above partitions

## LUKS setup

Default settings other than doublecheck (verify-passhphrase). We wont be able to beat rubber hose cryptanalysis...

```
cryptsetup --verify-passhphrase luksFormat /dev/nvme0n1p2
Are you sure? YES
Enter passhphrase (twice)
```

The `/` partition being encrypted, open the LUKS container on `/dev/nvme0n1p2`
disk and name it `cryptlvm`:

```
cryptsetup open /dev/nvme0n1p2 luks
Enter passhphrase
```
The decrypted container is now available at `/dev/mapper/luks`.
No need for LVM as we do not expect other disks to be added. 

### Formatting the filesystem

```
$ mkfs.fat -F32 /dev/nvme0n1p1
$ mkfs.btrfs -L btrfs /dev/mapper/luks
```

### Create BTRFS subvolumes

```
$ mount /dev/mapper/luks /mnt
$ btrfs sub create /mnt/@
$ btrfs sub create /mnt/@swap
$ btrfs sub create /mnt/@home
$ btrfs sub create /mnt/@pkg
$ btrfs sub create /mnt/@var_log
$ btrfs sub create /mnt/@snapshots
$ umount /mnt
```

### Mount subvolumes and create root folder structure

```
mount -o noatime,nodiratime,compress=lzo,space_cache=v2,ssd,subvol=@ /dev/mapper/luks /mnt
mkdir -p /mnt/{boot,home,var/cache/pacman/pkg,var/log,.snapshots,btrfs}
mount -o noatime,nodiratime,compress=lzo,space_cache=v2,ssd,subvol=@home /dev/mapper/luks /mnt/home
mount -o noatime,nodiratime,compress=lzo,space_cache=v2,ssd,subvol=@pkg /dev/mapper/luks /mnt/var/cache/pacman/pkg
mount -o noatime,nodiratime,compress=lzo,space_cache=v2,ssd,subvol=@var_log /dev/mapper/luks /mnt/var/log
mount -o noatime,nodiratime,compress=lzo,space_cache=v2,ssd,subvol=@snapshots /dev/mapper/luks /mnt/.snapshots
mount -o noatime,nodiratime,compress=lzo,space_cache=v2,ssd,subvolid=5 /dev/mapper/luks /mnt/btrfs
```

### Mount the EFI partition

```
# mount /dev/nvme0n1p1 /mnt/boot
```

### Create swap
Create swap file:

```
$ cd /mnt/btrfs/@swap
$ btrfs filesystem mkswapfile --size 16g --uuid clear .swapfile
$ swapon ./swapfile
$ cd -
```

## Configuration of the system

### Update the mirrors

```
$ pacman -S reflector
$ reflector --threads 8 --protocol http --protocol https --verbose --sort rate --country Netherlands --save /etc/pacman.d/mirrorlist
```

### Installation of the base packages

```
$ pacstrap /mnt base base-devel linux linux-firmware sudo man-db linux-tools git fwupd sudo btrfs-progs amd-ucode nano
```

### Generate a fstab file

```
$ genfstab -U /mnt >> /mnt/etc/fstab
```

## Setup Arch

Chroot into the new installation

```
$ arch-chroot /mnt/
```

### Setup timezone

```
$ ln -sf /usr/share/zoneinfo/Europe/Amsterdam /etc/localtime
$ hwclock --systohc
```

### Setup Localization
Edit /etc/locale.gen and uncomment en_US.UTF-8 UTF-8 and other needed UTF-8 locales. Generate the locales by running:

```
# locale-gen
```

Create the locale.conf(5) file, and set the LANG variable (LANG=en_US.UTF-8) accordingly:

```
/etc/locale.conf
```

### Network configuration

Install and enable network management daemon:

```
$ pacman -S networkmanager
$ systemctl enable NetworkManager
$ echo Framework > /etc/hostname
```

Edit `/etc/hosts`:

```
127.0.0.1		localhost
::1					localhost
127.0.1.1		ThinkPad.localdomain	ThinkPad
```

### Set the root passsword

```
$ passwd
```

### Create the main user

```
$ useradd -mG storage,wheel -s /bin/bash someone
$ passwd someone
```

Finally, change the `/etc/sudoers` file according to the config that you want
to deal with `sudo` command.

### Create a initial ramdisk environment

```
$ nano /etc/mkinitcpio.conf
```

Edit the `HOOKS` field:

```
...
HOOKS=(base systemd autodetect udev keyboard sd-vconsole modconf block encrypt filesystems btrfs resume)
...
```

Finally, recreate the `initramfs `image:

```
mkinitcpio -P
```

## Setup the boot manager

### Install GRUB into the EFI system partition

```
pacman --needed -Sy grub efibootmgr grub-btrfs
grub-install --target=x86_64-efi --bootloader-id=GRUB --efi-directory=/mnt/boot
grub-mkconfig -o /mnt/boot/grub/grub.cfg

```

### Finalise

```
$ exit
$ umount -R /mnt
$ reboot
```

## Post installation

Install AUR client -> yay
```
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ makepkg -sri
$ cd .. && rm -r yay
```

```
$ sudo pacman -S alacritty tmux firefox pipewire ntfs-3g fish chromium wget curl neofetch libreoffice-still linux-utils kdenlive syncthing remmina thunderbird keepass virtualbox audacity qbittorrent vlc htop calibre python3
$ sudo pacman -S pulseaudio alsa-utils pavucontrol alsamixer
$ sudo pacman -S wayland sddm xorg-xwayland xorg-xlsclients qt5-wayland glfw-wayland plasma kde-applications plasma-wayland-session
$ sudo pacman -S libva-mesa-driver mesa lm_sensors tlp powertop
$ sudo systemctl enable sddm
$ sudo systemctl enable NetworkManager
```

Install from AUR
```
$ yay -S freetube-bin spotify vscodium-bin f5vpn ventoy-bin anki ttf-ms-fonts protonvpn syncthingtray units zenpower3-dkms amdgpu_top dropbox
$ yay -S php81 php81-fpm php81-xml php81-mysql php81-pdo php81-xml 
```

Browser profile not on SSD
```
$ pacman -S profile-sync-daemon
$ systemctl --user enable psd --now
```

TODO:
https://archlinux.org/packages/extra/any/kernel-modules-hook/

