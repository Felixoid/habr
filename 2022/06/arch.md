---

# Arch linux installation

1. Set up the internet connection
2. Create a disk partition table GPT: `mktable gpt`
3. Create parts:
    - EFI: `mkpart primary fat32 0% 300MiB; toggle 1 esp`
    - LUKS: `mkpart primary 300MiB 100%`
4. Encrypt the LUKS partition:
    - `cryptsetup luksFormat /dev/<DEV_NAME>`
    - `cryptsetup open /dev/<DEV_NAME> crypted`
    - `dd if=/dev/zero of=/dev/mapper/crypted bs=10M status=progress`
5. Create LVM volume on top of LUKS:
    - `pvcreate /dev/mapper/crypted`
    - `vgcreate cLVM /dev/mapper/crypted`
    - `lvcreate -L 50G cLVM -n root`
    - `lvcreate -L ${100% - root - 10%} cLVM -n local` - leave something for the future
6. Format partitions and volumes:
    - boot: `mkfs.fat -F 32 /dev/sdxY`
    - root: `mkfs.xfs /dev/cLVM/root`
    - local: `mkfs.xfs /dev/cLVM/local`
7. Mount and create a directories tree:
    - `mount /dev/cLVM/root /mnt`
    - `mount --mkdir /dev/sdxY /mnt/boot`
    - `mount --mkdir /dev/cLVM/local /mnt/srv/local`
    - `mkdir -p /mnt/srv/local/{home,var}`
    - `mount -o bind --mkdir /mnt/srv/local/home /mnt/home`
    - `mount -o bind --mkdir /mnt/srv/local/var /mnt/var`
8. Install the system: `pacstrap /mnt base base-devel linux linux-firmware xfsprogs gvim lvm2 networkmanager zsh zsh-completions bash-completion sof-firmware`
9. Generate fstab `genfstab -U /mnt >> /mnt/etc/fstab`
10. Follow the [chroot](https://wiki.archlinux.org/title/Installation_guide#Chroot) manual.
    - `arch-chroot /mnt`
    - `ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime`
    - `hwclock --systohc`
    - `systemctl enable systemd-timesyncd.service`
    - for `locale-gen` should be the next `/etc/locale.gen`:

```
de_DE.UTF-8 UTF-8
en_DK.UTF-8 UTF-8
en_IE.UTF-8 UTF-8
en_US.UTF-8 UTF-8
ru_RU.UTF-8 UTF-8
```

    - `/etc/locale.conf`:

```
LANG=en_DK.UTF-8
LC_TIME=en_IE.UTF-8
```

    - `locale-gen`
    - `/etc/hostname`
    - fill `/etc/mkinitcpio.conf`:`HOOKS` and `/etc/kernel/cmdline`

11. Pacman setup:
    - `Color`
    - `ParallelDownloads = 7`
    - add `[community]` and `[multilib]`
12. User creation:
    - `pacman -S docker cups smartmontools libfprint`
    - `useradd -m -G games,network,floppy,power,cups,docker,rfkill,users,video,uucp,storage,lp,wheel -s /bin/zsh felixoid`
    - `passwd felixoid`

14. Install sbctl
    - https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Assisted_process_with_sbctl
    - https://wiki.archlinux.org/title/Unified_kernel_image#sbctl

13. Install yay

15. Post setup
    - `systemctl enable fstrim.timer`
    - `visudo` and uncomment `%wheel`
    - set `-j$(numcpu)}` in `/etc/makepkg.conf`

## Later
- TRIM
