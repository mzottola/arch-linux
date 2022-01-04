# Arch linux installation

## Troubleshooting

- issue on virtualbox, efi was not enabled. Check with:
```sh
efivar-tester
```
- keyboard layout on boot: see default layout [xorg keyboard config](https://wiki.archlinux.org/title/Xorg/Keyboard_configuration)
```sh
setxkbmap -print -verbose 10
```

## Beginning
- (optional) modify font terminal: see [console fonts](https://wiki.archlinux.org/title/Linux_console#Preview_and_temporary_changes)
```sh
setfont ter-112n
```
- setup wifi connection: see [iwd utility](https://wiki.archlinux.org/title/Iwd)
- list available layouts
```sh
localectl list-keymaps | grep fr
```
- change keyboard layout
```sh
loadkeys fr-latin9
```
- set correct date through ntp
```sh
timedatectl set-ntp true
```
- use reflector to update pacman mirror list: see [reflector](https://wiki.archlinux.org/title/reflector)
```sh
reflector -c France -a 6 --sort rate --save /etc/pacman.d/mirrorlist
# -c: Country
# -a: servers synchronized since 6 hours
# --sort: sort by some predicate (here rate but could be speed)
# --save: the file to save (already present in arch linux iso)
```
- resynchronize the servers
```sh
pacman -Syy
```

## Create partitions
- list block devices to find the disk name
```sh
lsblk
```
Partitions will be created using [gdisk](https://wiki.archlinux.org/title/GPT_fdisk)
```sh
gdisk /dev/$DISK_NAME

```
### EFI
See [EFI system partition](https://wiki.archlinux.org/title/EFI_system_partition)

```sh
Command: n
Partition number: # <default: press ENTER>
First sector: # <default: press ENTER>
Last sector: +512M # arch linux recommends 300MB or for early/buggy UEFI implementations 512M https://wiki.archlinux.org/title/EFI_system_partition
Hex code: ef00
```
### Swap (optional ?)
```sh
Command: n
Partition number: # <default: press ENTER>
First sector: # <default: press ENTER>
Last sector: +4G # rule of thumb is to make twice of the RAM until 2G, otherwise create 4G
Hex code: 8200
```

### Root
```sh
Command: n
Partition number: # <default: press ENTER>
First sector: # <default: press ENTER>
Last sector: # <default: press ENTER>
Hex code: # <default: press ENTER> (8300)
```

Then we can write it:
```sh
Command: w
```

## Format partitions

```sh
lsblk
```

### EFI
EFI partition must be FAT 32
```sh
mkfs.fat -F32 /dev/$EFI_PARTITION
```

### Swap
```sh
mkswap /dev/$SWAP_PARTITION
swapon /dev/$SWAP_PARTITION
```

### Root

- create the encryption and type passphrase
```sh
cryptsetup -y -v luksFormat /dev/$ROOT_PARTITION
```

- open the partition
```sh
cryptsetup open /dev/$ROOT_PARTITION cryptroot # cryptroot is a mapper to refer later
```

- format it
```sh
mkfs.ext4 /dev/mapper/cryptroot
```

## Mount partitions
- root
```sh
mount /dev/mapper/cryptroot /mnt
```
- efi
```sh
mkdir /mnt/boot
mount /dev/$EFI_PARTITION /mnt/boot
```

## Install basic dependencies
Using [pacstrap](https://man.archlinux.org/man/pacstrap.8)
```sh
pacstrap /mnt base linux linux-firmware vim intel-ucode
```

## Generate file system table
Good documentation [here](https://www.redhat.com/sysadmin/etc-fstab)
```sh
genfstab -U /mnt >> /mnt/etc/fstab # -U for UUID
```

## Installation
see [arch-chroot](https://wiki.archlinux.org/title/chroot)
```sh
arch-chroot /mnt
```

### Set timezone
[timedatectl](https://wiki.archlinux.org/title/System_time#Time_zone)

```sh
timedatectl list-timezones | grep Paris
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc # synchronize clocks
timedatectl status # to check if result is correct
```

### Set locale
- uncomment the desired line in the following file:
```sh
vim /etc/locale.gen
> en_US.UTF8 UTF-8
```
- synchronize the locale:
```sh
locale-gen
```
- write the locale in the conf file:
```sh
vim /etc/locale.conf
> LANG=en_US.UTF-8
```

### Keyboard
Set the default keyboard layout:
```sh
vim /etc/vconsole.conf
KEYMAP=fr-latin9
```

### Host
- hostname called here `archcrypt`:
```sh
vim /etc/hostname
> archcrypt
```
- hosts
```sh
vim /etc/hosts
127.0.0.1	localhost
::1		localhost
127.0.1.1	archcrypt.localdomain	archcrypt
```

### Update password
```sh
passwd
```

### Install dependencies
```sh
pacman -S grub efibootmgr networkmanager network-manager-applet dialog wpa_supplicant mtools dosfstools base-devel linux-headers bluez bluez-utils cups xdg-utils xdg-user-dirs alsa-utils pulseaudio pulseaudio-bluetooth git reflector terminus-font
```

### Modify mkinitcpio.conf
Because we are using encrypted partition, we need to modify the image loaded initially at startup, see [mkinitcpio](https://wiki.archlinux.org/title/mkinitcpio)
- add keyboard, keymap and encrypt in the HOOKS section
```sh
vim /etc/mkinitcpio.conf
HOOKS=(base udev autodetect keyboard keymap modconf block encrypt filesystems fsck)
```
- recreate the image
```sh
mkinitcpio -p linux
```

### GRUB bootloader
see [grub](https://wiki.archlinux.org/title/GRUB)
- install it
```sh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```
- generate config and write it into /boot/grub/grub.cfg
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```
- change configuration file because we use encryption: UUID of our root partition
```sh
blkid # find UUID_ROOT_PARTITION (the 8300 type)
vim /etc/default/grub
>> GRUB_CMDLINE_LINUX="cryptdevice=UUID=$UUID_ROOT_PARTITION:cryptroot root=/dev/mapper/cryptroot"
```
- regenerate config
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

### End of basic install
- enable some services
```sh
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable cups
```
- create user
```sh
useradd -mG wheel foobar
passwd foobar
```
- enable sudo priviledges for group wheel:
```sh
EDITOR=vim visudo
# uncomment line %wheel ALL=(ALL) ALL
```
- exit
```sh
exit
umount -a
reboot
```

### Connect wifi
```sh
nmcli device wifi # display available wifi networks
nmcli device wifi connect $WIFI_SSID --ask
```

### Update [archlinux-keyring](https://wiki.archlinux.fr/pacman-key)
```sh
sudo pacman -S archlinux-keyring
```

### Global
```sh
timedatectl set-ntp true
sudo hwclock --systohc # synchronize clocks
sudo reflector -c France -a 6 --sort rate --save /etc/pacman.d/mirrorlist
sudo pacman -Syy
sudo pacman -S bash-completion
sudo systemctl enable --now reflector.timer
sudo systemctl enable --now fstrim.timer
```
- install graphic
```sh
sudo pacman -S xf86-video-intel xorg i3 dmenu lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings ttf-dejavu ttf-liberation noto-fonts firefox nitrogen picom lxappearance pcmanfm materia-gtk-theme papirus-icon-theme xfce4-terminal archlinux-wallpaper
```
- enable lightdm
```sh
sudo systemctl enable lightdm
```

### Keyboard X11
Either:
- create /etc/X11/xorg.conf.d/00-keyboard.conf and change it according to previous config for `XkbModel`
```sh
Section "InputClass"
        Identifier "system-keyboard"
        MatchIsKeyboard "on"
        Option "XkbLayout" "fr"
        Option "XkbModel" "pc105"
EndSection
```
Or: (to test)
```sh
localectl set-x11-keymap fr
```

### `reboot`
