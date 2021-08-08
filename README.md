# Arch installation with GRUB, GNOME and either ext4 or btrfs

[![](https://github.com/ickyicky/arch_install/blob/main/result.png?raw=true)](https://github.com/ickyicky/arch_install)

## Table of contents

- [Arch installation with GRUB, GNOME and either ext4 or btrfs](#arch-installation-with-grub--gnome-and-either-ext4-or-btrfs)
  * [Table of contents](#table-of-contents)
  * [Making things easy](#making-things-easy)
    + [Connecting to a wireless network](#connecting-to-a-wireless-network)
    + [Getting your ip address](#getting-your-ip-address)
  * [Preparing for installation](#preparing-for-installation)
    + [Load keyboard layout](#load-keyboard-layout)
    + [Verify boot mode](#verify-boot-mode)
    + [Update the system clock](#update-the-system-clock)
  * [Disk partitioning](#disk-partitioning)
    + [Legacy boot mode](#legacy-boot-mode)
    + [UEFI boot mode](#uefi-boot-mode)
    + [Using fdisk](#using-fdisk)
    + [Installing alongside Windows](#installing-alongside-windows)
    + [Wiping disk](#wiping-disk)
    + [Setting partition types and flags - Legacy boot mode](#setting-partition-types-and-flags---legacy-boot-mode)
    + [Setting partition types and flags - UEFI boot mode](#setting-partition-types-and-flags---uefi-boot-mode)
    + [Last step](#last-step)
  * [Formating partitions](#formating-partitions)
    + [EFI Boot partition - if created](#efi-boot-partition---if-created)
    + [SWAP partition - if created](#swap-partition---if-created)
    + [Linux partition - format + mounting for install](#linux-partition---format---mounting-for-install)
      - [EXT4](#ext4)
      - [BTRFS](#btrfs)
  * [Installation!](#installation-)
  * [Setting up installed system](#setting-up-installed-system)
    + [Generate fstab](#generate-fstab)
    + [chroot into install](#chroot-into-install)
    + [Basic configuration](#basic-configuration)
      - [Network configuration](#network-configuration)
      - [Root password](#root-password)
  * [Installing essential packages](#installing-essential-packages)
    + [Adding btrfs module - btrfs only](#adding-btrfs-module---btrfs-only)
    + [Installing GRUB Boot loader - Legacy boot mode](#installing-grub-boot-loader---legacy-boot-mode)
    + [Installing GRUB Boot loader - UEFI boot mode](#installing-grub-boot-loader---uefi-boot-mode)
  * [Additional config](#additional-config)
    + [Creating user and enabling him to sudo](#creating-user-and-enabling-him-to-sudo)
    + [Enabling services](#enabling-services)
    + [Install display server](#install-display-server)
    + [Install and enable GNOME](#install-and-enable-gnome)
  * [Reboot into installed Arch](#reboot-into-installed-arch)
  * [Additional step - configure GNOME](#additional-step---configure-gnome)

## Making things easy

First of all, you want to dual boot and have windows already installed, shrink its partition. Next, burn Arch iso to an USB or CD and boot it. Disable safe boot in BIOS if enabled.

To make things easy, it is good idea to perform the installation through ssh. This enables you to copy and paste commands which is really usefull :) I highly encourage to copy all commands i supplied to some editor, edit their arguments and then paste it to ssh session and execute. To enable this feature, first set up password for **root** account:

```sh
passwd
```

Next step is to connect to the local network. This step is not necessary if you are using ethernet cable, in that case you should be already connected to the internet. If you are using wifi, follow the steps below.

### Connecting to a wireless network

To connect to a wireless network use **iwctl** command. First step is to list avalibe devices:

```sh
iwctl station list
```

There should be device called **wlan0** or something simillar, that is the one we want to use to connect to the wifi. In next examples replace **wlan0** with your wireless network interface name. Next, scan avalibe networks using command:

```sh
iwctl station wlan0 scan
iwctl station wlan0 get-networks
```

You should see your network name on the list. Now, connect to the wifi using command:

```sh
iwctl station wlan0 connect $NETWORK_NAME
```

where $NETWOK_NAME is you wireless network name. In my case network name is "Internet", co my command looks like:

```sh
iwctl station wlan0 connect Intenet
```

You will be asked to enter network passphrase (password), enter password and confirm it with enter.

### Getting your ip address

Now, that the internet connection is established, it is time to get your ip address and connect via ssh. To get ip addresses assigned to all of your interfaces use command:

```sh
ip a
```

which is short for:

```sh
ip address
```

As a result you should get something like this:

```sh
1: lo: (...)
2: enp3s0: (...)
4: wlan0: (...)
    inet 192.168.0.54/24 (...)
```

Where what follows inet is your ipv4 address you are looking for. **lo** interface is your loopback, so ignore it. In my case my ipv4 address is **192.168.0.54**. In order to connect from another machine we need to enter command:

```sh
ssh root@$IP_ADDRESS
```

where $IP_ADDRESS is ipv4 address assigned to our machine running arch install, in my case:

```sh
ssh root@192.168.0.54
```

Enter your password set in the beggining with **passwd** command and you are ready to go.

## Preparing for installation

Now, you need to prepare for installation.

### Load keyboard layout

Default keyboard layout is **us**. In order to list avalibe ones use command:

```sh
ls /usr/share/kbd/keymaps/**/*.map.gz
```

In my case i want to load Polish keyboard layout, so additionaly grep for for "pl":

```sh
ls /usr/share/kbd/keymaps/**/*.map.gz | grep pl
```

There is layout:

```sh
/usr/share/kbd/keymaps/i386/qwerty/pl.map.gz
```

which i want to use, it is simple, default polish layout. In order to load it i use command:

```sh
loadkeys pl
```

As you see, you only specify filename, without extension or path.

### Verify boot mode

Next step is to verify boot mode. It is important, because both patritioning and GRUB isntallation depend on your boot mode. Check for defined efivars:

```sh
ls /sys/firmware/efi/efivars
```

If you get "No such file or directory" error, it means you are using legacy mode. In other case, you are booted in UEFI mode. Remember which mode you have, it is very important!

### Update the system clock

Simply paste this command:

```sh
timedatectl set-ntp true
```

## Disk partitioning

This step is very important. Now is the right time to make mind about partions layout. I will try to cover most common layouts. Before you begin this step decide if you want swap partition and if you want to completely wipe your disk. If you are installing Arch overwriting existing Linux installation you can skip this part, your disk is already partitioned. Just skip to next part.

Now, before next steps identify disk name using command:

```sh
fdisk -l
```

Device name is something like that: */dev/sdb* with partition names like */dev/sdb1*.


### Legacy boot mode

If you don't have efivars, you are in legacy mode. That means you don't have to create additional partition for boot loader (GRUB). The final layout should be like this:

- / partition - new partition / existing Linux one
- SWAP partition if needed, for example 16GB
- other partitions with other operating systems

### UEFI boot mode

In that case, you need one additional partition:

- /bool/efi partition - about 300MB+

### Using fdisk

To create new partition for Linux use command:

```sh
fdisk /dev/sdb
```

Create new partition with **n**, select **p**, primary partition type, use default partition sumber (remember it!), use default first sector, specify size, for example +180G for 180G partition or just use default for using whole avalibe space. If you want to use swap, create swap partition first, for example with +16G for swap and then just use all avalibe space for primary partition.

Example steps:
- 500MB partition for UEFI boot - n -> p -> ENTER -> ENTER -> +500M
- 16GB partition for SWAP - n -> p -> ENTER -> ENTER -> +16G
- all avalibe space for linux partition - n -> p -> ENTER -> ENTER -> ENTER

if you are asked to overwrite signature, say Yes to it :)

### Installing alongside Windows

In that case you should have empty space on your disk. In my case it was */dev/sdb* with only one partition, */dev/sdb1* with Windows on it of size 120G, where whole disk is 240G.

### Wiping disk

In that case, in fdisk create new patrition table. For Legacy boot mode use DOS partition table with command **o**, for UEFI use GPT with command **g**. Next, create required partitions.

### Setting partition types and flags - Legacy boot mode

List partitions with **p**.

In that case, you need to add bootable flag to partition with Linux, just use **a** command and enter partition number.

For SWAP partition you need to change partition type to SWAP. To do that, use **t**, then enter SWAP partition number, seletc L to list partition types and choose one for **swap**, in my case 82.

### Setting partition types and flags - UEFI boot mode

List partitions with **p**.

For SWAP partition you need to change partition type to SWAP. To do that, use **t**, then enter SWAP partition number, seletc L to list partition types and choose one for **swap**, in my case 19.

For EFI Boot partition you need to change partition type to EFI BOOT. To do that, use **t**, then enter SWAP partition number, seletc L to list partition types and choose one for **EFI system**, in my case 1.

### Last step

Write changes with **w** command.

## Formating partitions

Next, format partitions you created + Linux one.

### EFI Boot partition - if created

use **fdisk -l** to get all partitions, select one with EFI type (in my case /dev/sda1) and format it:

```sh
mkfs.fat -F32 /dev/sda1
```

### SWAP partition - if created

use **fdisk -l** to get all partitions, select one with Linux swap type (in my case /dev/sdb1) and activate swap:

```sh
mkswap /dev/sdb1
swapon /dev/sdb1
```

### Linux partition - format + mounting for install

Identify Linux partition with **fdisk -l** and format it. In my case it is /dev/sdb2.

#### EXT4

To format it to etx4 (easy way - recommended) use:

```sh
mkfs.ext4 /dev/sdb2
```

Now just mount it:

```sh
mount /dev/sdb2 /mnt
```

If you are in UEFI boot mode mount /boot partition as well:

```sh
mount /dev/sda1 /mnt/boot
```

#### BTRFS

For btrfs use:

```sh
mkfs.btrfs /dev/sdb2
```

Additionaly, create subvolumes. Good example is:

```sh
PARTITION="/dev/sdb2"
mount $PARTITION /mnt
btrfs su cr /mnt/@
btrfs su cr /mnt/@home
btrfs su cr /mnt/@var
btrfs su cr /mnt/@opt
btrfs su cr /mnt/@tmp
btrfs su cr /mnt/@.snapshots
umount /mnt
mount -o noatime,commit=120,compress=zstd,space_cache,subvol=@ $PARTITION /mnt
mkdir /mnt/{boot,home,var,opt,tmp,.snapshots}
mount -o noatime,commit=120,compress=zstd,space_cache,subvol=@home $PARTITION /mnt/home
mount -o noatime,commit=120,compress=zstd,space_cache,subvol=@opt $PARTITION /mnt/opt
mount -o noatime,commit=120,compress=zstd,space_cache,subvol=@tmp $PARTITION /mnt/tmp
mount -o noatime,commit=120,compress=zstd,space_cache,subvol=@.snapshots $PARTITION /mnt/.snapshots
mount -o subvol=@var $PARTITION /mnt/var
```

Just replace PARTITION= with correct partition name and you should be OK. If you don't trust me, go read BTRFS documentation :)

If you are in UEFI boot mode mount /boot partition as well:

```sh
mount /dev/sda1 /mnt/boot
```

## Installation!

Installation is done with:

```sh
pacstrap /mnt
```

command. You need to specify packages you want to install. Base installation is:

```sh
pacstrap /mnt base linux linux-firmware vim
```

It containst only base linux system, kernel and firmware files for linux + vim of course. Depending on installation options additionally add following ones:
- AMD processor: ```amd-ucode```
- Intel processor: ```intel-ucode```
- BTRFS filesystem: ```btrfs-progs```

For me, intell processor + btrfs filesystem the command is following:

```sh
pacstrap /mnt base linux linux-firmware vim intel-ucode btrfs-progs
```

## Setting up installed system

Now, that the system is installed, we need to take additional steps.


### Generate fstab

In order for installed linux to automatically mount correct partitions you need to generate fstab configuration file. To do so, type the following:

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

### chroot into install

Since the installation is complete, we need to chroot into it in order to install additional packages and configure installed system. Exec the following:

```sh
arch-chroot /mnt
```

### Basic configuration

Set timezone (you can list avalibe ones with ```timedatectl list-timezones```) and sync hardware clock:

```sh
ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
hwclock --systohc
```

Next, decide on what locale (language) you want to use. Edit */etc/locale.gen* in search for correct one. In my case, I want to use Polish language, so i uncomment the line with pl UTF8 locale:

```sh
vim /etc/locale.gen
```

resulting in ```pl_PL.UTF-8 UTF-8``` uncommented. Then, generate locale with:

```sh
locale-gen
```

and set generated locale to /etc/locale.conf. Use the name of locale displayed while generating.

```sh
echo LANG=pl_PL.UTF-8 >> /etc/locale.conf
```

Next step is to set correct keyboard layout. If you didn't use ```loadkeys``` command in the beggining use us keymap, which is the default one.

```sh
echo KEYMAP=pl >> /etc/vconsole.conf
```

#### Network configuration

Select machine name and save it to /etc/hostname and generate /etc/hosts. Edit HOSTNAME_NEW value to desired value and execute:

```sh
HOSTNAME_NEW=ArchThinkpad
echo $HOSTNAME_NEW >> /etc/hostname
echo "127.0.0.1       localhost" >> /etc/hosts
echo "::1       localhost" >> /etc/hosts
echo "127.0.1.1	$HOSTNAME_NEW.localdomain	$HOSTNAME_NEW" >> /etc/hosts
```

#### Root password

Set the password for root account with ```passwd```.

## Installing essential packages

Next, install essential packages with ```pacman -S``` command. The ones you need for sure:
- ```grub``` - grub boot loader
- ```grub-btrfs``` - grub btrfs module, if you used btrfs filesystem
- ```efibootmgr``` - EFI Boot Manager, if you have UEFI boot mode
- ```base-devel linux-headers networkmanager network-manager-applet wpa_supplicant dialog os-prober mtools dosfstools reflector git lsb-release xdg-utils xdg-user-dirs``` - all the essentials
- ```bluez bluez-utils``` - bluetooth support
- ```cups``` - printing support
- ```tlp``` - laptop power management, only for laptops

In my case, i used the following command for BTRFS on Legacy boot mode installation (so everything without efibootmgr):

```sh
pacman -S base-devel linux-headers networkmanager network-manager-applet wpa_supplicant dialog os-prober mtools dosfstools reflector git lsb-release xdg-utils xdg-user-dirs bluez bluez-utils cups grub grub-btrfs tlp
```

Just use defaults and install all the things you want.

### Adding btrfs module - btrfs only

Edit /etc/mkinitcpio.conf (```vim /etc/mkinitcpio.conf```) and add **btrfs** to *MODULES=()* line, so it looks like:

```sh
MODULES=(btrfs)
```

and execute:

```sh
mkinitcpio -p linux
```

### Installing GRUB Boot loader - Legacy boot mode

In that case, simply execute:

```sh
grub-install /dev/sdb
```

Where /dev/sdb is the disk on which you installed Arch. Then generate grub config:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

### Installing GRUB Boot loader - UEFI boot mode

In that case, simply execute:

```sh
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch
```

Where /dev/sdb is the disk on which you installed Arch. Then generate grub config:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

## Additional config

### Creating user and enabling him to sudo

Create an user with specified username in wheel group (for sudo):

```sh
MYUSERNAME=archmaster
useradd -mG wheel $MYUSERNAME
passwd $MYUSERNAME
```

where MYUSERNAME is your desired username of course.

Then execute ```visudo``` and uncomment the following line:

```
%wheel ALL=(ALL) ALL
```

### Enabling services

Enable netowrking, bluetooth, tlp and CUPS services. Just paste the following:

```sh
systemctl enable NetworkManager
systemctl enable bluetooth || echo "no bluetooth"
systemctl enable cups.service || echo "no cups"
systemctl enable tlp.service || echo "no tlp"
```

### Install display server

Next step is to install display server and display drivers. Correct opensource drivers are:
- ```xf86-video-amdgpu``` - for AMD GPU
- ```xf86-video-intel``` - for Intell GPU, integrated one as well
- ```xf86-video-nouveau``` - for NVIDIA GPU
- ```xf86-video-vesa``` - generic driver

Additionally install OpenGL support (```mesa```) and X11 display server (```xorg``` group). For me (Intell CPU with integrated GPU):

```sh
pacman -S xf86-video-intel mesa xorg
```

### Install and enable GNOME

Love it or hate it, GNOME is the most complete Linux DE, together with KDE. For ease of use i suggest installing GNOME:

```sh
pacman -S gnome gnome-extra
systemctl enable gdm.service
```
## Reboot into installed Arch

Exit chroot with ```exit``` and reboot machine (```reboot```). Congratulations, you have installed Arch with GNOME. Now check if it works and if now search through archwiki to solve all the problems :) In my case i forgot the bootable flag on */dev/sdb2*, easily fixable no operating system found error :)

## Additional step - configure GNOME

I provide additional config for GNOME myself. In order to install it:
1. open Extensions App and enable *User Themes* extension
2. open Terminal, go to preferences and add new profile with random name
3. git clone <https://github.com/ickyicky/dotfiles>
4. ```cd dotfiles```
5. ```./install.sh``` and select y for every option
6. log out and log in
7. open terminal, select Base16Ocean256 profile, if needed add transparency in colors tab
8. reopen terminal

Now you are ready to post neofetch screenshot to r/uniporn

You can additionally install pamac for GUI software installation:

```sh
yay -S pamac
```
