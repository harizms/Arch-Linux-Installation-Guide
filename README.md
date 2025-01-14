# Arch Linux installation guide

## Introduction

My Notes on Installing [Arch Linux](https://www.archlinux.org/)

## Download the ISO

First, download the ISO here https://www.archlinux.org/download/ and burn to a drive.

## Inital setup

Check if the system is under UEFI:


```sh
ls /sys/firmware/efi/efivars
```


Ensure the system clock is accurate

```sh
timedatectl set-ntp true
```


## Disk management

- https://wiki.archlinux.org/index.php/Partitioning
- https://wiki.archlinux.org/index.php/EFI_system_partition

**$\textcolor{yellow}{\textsf{Assuming that your drive is}}$ /dev/sda .Use `lsblk` to find your drive**

```sh
fdisk /dev/sda
```

**UEFI/GPT:**

| Partition  | Space  | Type             |
|------------|--------|------------------|
| /dev/sda1  | xG     | Linux Filesystem |
| /dev/sda2  | xG     | Home Partition   |
| /dev/sda3  | xG     | Linux swap       |
| /dev/sda4  | 512M   | EFI System       |

**BIOS/MBR:**

| Partition  | Space  | Type             |
|------------|--------|------------------|
| /dev/sda1  | xG     | Linux Filesystem |
| /dev/sda2  | xG     | Home Partition   |
| /dev/sda3  | xG     | Linux swap       |

#### File systems

`/` partition:

```sh
mkfs.ext4 /dev/sda1
mount /dev/sda1 /mnt
```
`/home` partition:

```sh
mkfs.ext4 /dev/sda2
mkdir /mnt/home
mount /dev/sda2 /mnt/home
```
`swap` partition:

```sh
mkswap /dev/sda3
swapon /dev/sda3
```

`/boot` partition: (UEFI/GPT) 

#### $\textcolor{red}{\textsf{Warning : }}$ Only format the EFI system partition if you created it during the partitioning step. If there already was an EFI system partition on disk beforehand, reformatting it can destroy the boot loaders of other installed operating systems.</span>

```sh
mkfs.fat -F32 /dev/sda4
mkdir -p /mnt/boot/EFI
mount /dev/sda4 /mnt/boot/EFI
```

## Install system

Install the base packages:

```sh
pacstrap /mnt base base-devel linux linux-firmware zsh neovim
```

## System setup

Generate fstab:

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```


Enter the chroot:

```sh
arch-chroot /mnt
```

Set timezone:

```sh
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc
```

Set locale :

```sh
sed -i 's/#en_US.UTF-8/en_US.UTF-8/g' /etc/locale.gen
```

Generate locales:

```sh
locale-gen
```

Set default locale:

```sh
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Set hostname:

**Replace $HOSTNAME with a name you prefer**

```sh
echo "$HOSTNAME" > /etc/hostname
```

`/etc/hosts` file:

```sh
127.0.0.1      localhost
::1            localhost
127.0.1.1   $HOSTNAME.localdomain   $HOSTNAME
```

Set root password:

```sh
passwd
```

## Microcode
- https://wiki.archlinux.org/index.php/Microcode

### Intel
```sh
pacman -S intel-ucode
```

### AMD
```sh 
pacman -S amd-ucode
```

## Bootloader: GRUB

- https://wiki.archlinux.org/title/GRUB

### UEFI/GPT
```sh
pacman -S grub efibootmgr 
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
grub-mkconfig -o /boot/grub/grub.cfg
```

### BIOS/MBR
```sh
pacman -S grub dosfstools 
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```
## Networking: NetworkManager

- https://wiki.archlinux.org/index.php/NetworkManager

```sh
pacman -S networkmanager
systemctl enable NetworkManager
```

## Reboot! (Optional)
 
 ```sh
exit
umount -R /mnt
reboot
```

## User accounts

Add user:

```sh
useradd -m -g wheel -c 'Full name' -s /usr/bin/zsh username
passwd username
```

### Admin Privileges

#### Sudo

***sudo already comes with the base-devel package***

Uncomment the line %wheel ALL=(ALL:ALL) ALL :

```sh
sed -i 's/#%wheel ALL=(ALL:ALL) ALL/%wheel ALL=(ALL:ALL) ALL/g' /etc/sudoers
```
#### OpenDoas

```sh
pacman -S opendoas
```
Add this to /etc/doas.conf :
````
permit :wheel 
permit persist :wheel 
permit nopass :wheel as root cmd pacman args -Syu
````
The owner and group for /etc/doas.conf should both be ``0`` ,file permissions should be set to ``0400``:
```sh
chown -c root:root /etc/doas.conf
chmod -c 0400 /etc/doas.conf
```

## Desktop Environment: Plasma KDE

- https://wiki.archlinux.org/index.php/KDE

```sh
pacman -S plasma
```

## Display Manager: SDDM

- https://wiki.archlinux.org/index.php/SDDM

```sh
pacman -S sddm
systemctl enable --now sddm
```

## AUR Helper: paru

- https://wiki.archlinux.org/index.php/AUR_helpers

I prefer [`paru`](https://github.com/morganamilo/paru)

#### $\textcolor{red}{\textsf{Warning :}}$ AUR packages are user-produced content. These PKGBUILDs are completely unofficial and have not been thoroughly vetted. Any use of the provided files is at your own risk.</span>

Note : **Running makepkg as root is not allowed**

```sh
$ git clone https://aur.archlinux.org/paru.git
$ cd paru
$ makepkg -sirc
```



## Installing Thermald  

"thermald is a Linux daemon used to prevent the overheating of Intel CPUs. This  daemon monitors temperature and applies compensation using available cooling methods."

```sh
doas pacman -S thermald 
doas systemctl enable --now thermald.service
```

## Enabling Hardware Acceleration

Enabling hardware acceleration is important, it'll use your laptops GPU for video decoding or encoding instead of your CPU, it will make your laptop run cooler and faster while saving power this can resolve issues like videos stuttering and your laptop being hot while watching videos, I recommend looking into the Arch Wiki guide hardware acceleration for applications below after you've set this up.

&#x200B;
run ``lspci | grep VGA`` to see your GPU

For Intel GPU's 2014 and newer I recommend you install ``doas pacman -S intel-media-driver`` \- From Arch Wiki. > **"HD Graphics series starting from Broadwell (2014) and newer are supported by intel-media-driver."**

For Intel GPU's 2013 and older I recommend you install ``doas pacman -S libva-intel-driver``  \-  From Arch Wiki > **"GMA 4500 (2008) and newer GPUs, including HD Graphics up to Coffee Lake (2017) are supported by libva-intel-driver."**

You'll also want to ``sudo pacman -S libvdpau-va-gl libva-utils vdpauinfo``

run ``export LIBVA\_DRIVER\_NAME=iHD`` \- (If you installed intel-media-driver)

run ``export LIBVA\_DRIVER\_NAME=i965`` \- (If you install libva-intel-driver)

run ``export VDPAU\_DRIVER=va\_gl``

run ``vainfo`` to confirm everything is working.

run ``vdpauinfo`` to confirm everything is working.

If everything is working run ``doas nano /etc/environment`` and add ***LIBVA\_DRIVER\_NAME=iHD*** or ***LIBVA\_DRIVER\_NAME=i965*** (Depending on which driver you installed.) Then add ***VDPAU\_DRIVER=va\_gl*** save and exit.

Here is the [Arch Wiki guide to hardware acceleration for NVIDIA and AMD users](https://wiki.archlinux.org/title/Hardware_video_acceleration).

I recommend looking at [this](https://wiki.archlinux.org/title/Hardware_video_acceleration#Configuring_applications) to enable hardware acceleration in applications.

On the hardware acceleration Arch Wiki page [check if your GPU and hardware acceleration driver can decode VP9](https://wiki.archlinux.org/title/Hardware_video_acceleration#Comparison_tables) if it can't install the h264ify browser extension otherwise you wont be able to watch stuff like YouTube videos with hardware video decoding keep in mind h264ify will disable resolutions above 1080p.

## General recommendations
- https://wiki.archlinux.org/index.php/General_recommendations
