# Framework AMD Arch setup
Framework AMD Arch BTRFS snapshots systemd LUKS KDE Wayland

# Make ISO
https://archlinux.org/download/

# SSH access during set-up

```sh
passwd
systemctl start sshd
ip addr
```
Connect to IP

# Partitioning

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

# LUKS setup

```
cryptsetup --hash sha512 --use-random --verify-passhphrase luksFormat /dev/nvme0n1p2
Are you sure? YES
Enter passhphrase (twice)
```

The `/` partition being encrypted, we will open the LUKS container on `/dev/nvme0n1p2`
disk and name it `cryptlvm`:

```
cryptsetup open /dev/nvme0n1p2 cryptlvm
Enter passhphrase
```
The decrypted container is now available at `/dev/mapper/cryptlvm`.

# LVM setup

#### Create a physical volume on top of the opened LUKS container

```
$ pvcreate /dev/mapper/cryptlvm
```

#### Add the previously created physical volume to a volume group

```
$ vgcreate vg /dev/mapper/cryptlvm
```

#### Create the root logical volume on the volume group

<!-- TODO: See if it is necessary because of swapfile -->
```
$ lvcreate -l 100%FREE vg -n root
```

### Formatting the filesystem

```
$ mkfs.fat -F32 /dev/nvme0n1p1
$ mkfs.btrfs -L btrfs /dev/mapper/vg-root
```

### Btrfs subvolumes

Subvolumes are part of the filesystem with its own and independnet file/directory
hierarchy, where each subvolume can share file extents.

#### Create Btrfs subvolumes

```
$ mount /dev/mapper/vg-root /mnt
$ btrfs subvolume create /mnt/root
$ btrfs subvolume create /mnt/home
$ umount /mnt
```

