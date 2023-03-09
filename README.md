# UEFI ONLY
# arch-linux-full-installation-guide

##### Check if there is an Internet connection (if on wired)
 ```
 ip addr show
 ```
##### For WiFi, you can use iwctl
###### Access the iwd prompt:
 ```
 iwctl
 ```
##### Obtain a list of Wifi devices in your system:
 ```
 device list
 ```
###### Take note of the device name for your WiFi device, we’ll need it later.

 Note: If you don’t see a Wifi device there, and you’re sure you do have WiFi capability, you shouldn’t proceed any further with installing Arch. You’ll want to check hardware compatibility with your Wifi card and Linux, and then resume installation at a later date.

##### Scan for wireless access points:
```
station <device> scan
```
##### View a list of detected networks:
 ```
 station <device> get-networks
 ```
##### Connect to a wireless network:
 ```
 station <device> connect <wireless-network-name>
 ```


##### Preparing the hard disk (UEFI)
###### See partitions/drives on the system (find the name of your hard drive)
 ```
 fdisk -l
 ```
##### == Start the partitioner (fdisk)
```
fdisk /dev/<DEVICE> (substitute <DEVICE> for your device name, example: /dev/sda or /dev/nvme0n1)
```
##### Show current partitions
 ```
 p
 ```
##### Create EFI partition
 ```
 g (to create an empty GPT partition table)
 n
 enter
 enter
 +500M
 t
 1 (for EFI)
 ```
##### Create LVM partition
 ```
 n
 enter
 enter
 enter
 t
 enter
 43 (for Linux LVM)
 ```
##### Show current partitions again
 ```
 p
 ```
##### Finalize partition changes
 ```
 w
 ```
##### Format the EFI partition
 ```
 mkfs.fat -F32 /dev/sda1 (or whatever the device name of the first partition is)
 ```
##### Set up lvm
 ```
 pvcreate --dataalignment 1m /dev/sda2 (or whatever the device name is of the second partition)
 vgcreate volgroup0 /dev/sda2 (or whatever the device name is of the second partition)
 lvcreate -L 30GB volgroup0 -n lv_root
 lvcreate -l 100%FREE volgroup0 -n lv_home (or use something like "-L 250GB" if you want to make the volume size lower)
 modprobe dm_mod
 vgscan
 vgchange -ay
 ```
##### Format the root partition
```
mkfs.ext4 /dev/volgroup0/lv_root
```
##### Mount the root partition
```
mount /dev/volgroup0/lv_root /mnt
```
##### Format the home partition
```
mkfs.ext4 /dev/volgroup0/lv_home
```
##### Create the home partition mount point
```
mkdir /mnt/home
```
##### Mount the home volume
```
mount /dev/volgroup0/lv_home /mnt/home
```
##### Create the /etc dirctory
```
mkdir /mnt/etc
```
##### Create the /etc/fstab file
```
genfstab -U -p /mnt >> /mnt/etc/fstab
```
##### Check the /etc/fstab file
```
cat /mnt/etc/fstab
```

#### Install Arch Linux
##### Install Arch Linux base packages
```
pacstrap -i /mnt base
```
##### Access the in-progress Arch installation
```
arch-chroot /mnt
```
##### Install a kernel and headers
```
pacman -S linux linux-headers
```
##### For LTS:
```
pacman -S linux-lts linux-lts-headers
```
###### Or both:
```
pacman -S linux linux-lts linux-headers linux-lts-headers 
```
##### Install a text editor
```
pacman -S nano
```
##### Install optional packages
```
pacman -S base-devel openssh
```
##### Enable OpenSSH if you’ve installed it
```
systemctl enable sshd
```
##### Install packages for networking
```
pacman -S networkmanager wpa_supplicant wireless_tools netctl
```
##### Install dialog (required for wifi-menu)
```
pacman -S dialog
```
##### Enable networkmanager
```
systemctl enable NetworkManager
```
##### Add LVM support
```
pacman -S lvm2
```
##### Edit /etc/mkinitcpio.conf
```
nano /etc/mkinitcpio.conf
```
###### On the “HOOKS” line, add support for lvm2 and optionally encryption.

##### unencrypted hard disk:
##### Add “lvm2” in between “block” and “filesystems”

##### encrypted hard disk:
##### Add “encrypt lvm2” in between “block” and “filesystems”

##### It should look similar to the following (don’t copy this line in case they change it, but just add the required new items):

##### HOOKS=(base udev autodetect modconf block encrypt lvm2 filesystems keyboard fsck)
##### Create the initial ramdisk for the main kernel
```
mkinitcpio -p linux
```
##### Create the initial ramdisk for the LTS kernel (if you installed it)
```
mkinitcpio -p linux-lts
```
##### Uncomment the line from the /etc/locale.gen file that corresponds to your locale
```
nano /etc/locale.gen (uncomment en_US.UTF-8)
```
##### Generate the locale
```
locale-gen
```
##### Set the root password
```
passwd
```
##### Create a user for yourself
```
useradd -m -g users -G wheel <username>
```
##### Set your password
```
 passwd <username>
```
##### Install sudo (may already be installed)
```
pacman -S sudo
```
##### Allow users in the ‘wheel’ group to use sudo
```
EDITOR=nano visudo
```
#######Uncomment:
```
%wheel ALL=(ALL) ALL
```
##### Setting up GRUB
###### GRUB is the bootloader that was used in the video. Follow ONE of the following sections, depending on whether you are using UEFI, encryption, etc


##### Installing GRUB for UEFI, with no encryption
##### Install the required packages for GRUB:
```
pacman -S grub efibootmgr dosfstools os-prober mtools
```

##### Create the EFI directory:
```
mkdir /boot/EFI
```
##### Mount the EFI partition:
```
mount /dev/<DEVICE PARTITION 1> /boot/EFI
```
##### Install GRUB:
```
grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck
```
##### Create the locale directory for GRUB
```
mkdir /boot/grub/locale
```
###### Copy the locale file to locale directory
```
cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo
```
##### Generate GRUB’s config file
```
grub-mkconfig -o /boot/grub/grub.cfg
```

##### Testing the installation
##### Check the /etc/fstab file to make sure it includes all the right partitions
 ```
 cat /etc/fstab
```
##### You should have a mountpoint for all of the partitions that were created.

##### Moment of truth: Reboot your machine
##### Exit the chroot environment
```
exit
```
##### Unmount everything (some errors are okay here)
```
umount -a
```
##### Reboot the machine
```
reboot
```
##### Post-Install Tweaks/Enhancements
##### Create swap file
```
dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
chmod 600 /swapfile
mkswap /swapfile
```
##### Back up the /etc/fstab file
```
 cp /etc/fstab /etc/fstab.bak
```
##### Add the swap file to the /etc/fstab file
 ```
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab
```
##### Set time zone
###### List time zones:
```
timedatectl list-timezones
```
##### Set your time zone:
```
timedatectl set-timezone America/Detroit
```
##### Enable time synchronization via systemd:
```
systemctl enable systemd-timesyncd
```
##### Set the hostname
###### Consider setting the hostname of your new installation. You can do so with the following command:
```
hostnamectl set-hostname myhostname
```
###### Also, make the same change in /etc/hosts:
```
nano /etc/hosts
```
####### Example lines to add:
```
127.0.0.1 localhost
127.0.1.1 (myhostname)
```
Install CPU Microde files (AMD CPU)
```pacman -S amd-ucode
```
Install CPU Microde files (Intel CPU)
```
pacman -S intel-ucode
```
Install Xorg if you plan on having a GUI
```
pacman -S xorg-server
```
Install 3D support for Intel or AMD graphics
If you have an Intel or AMD GPU, install the mesa package:
```
pacman -S mesa
```
Install Nvidia Driver packages if you have an Nvidia GPU
```
pacman -S nvidia nvidia-utils
```
Note: Install nvidia-lts if you’ve installed the LTS kernel:
```
pacman -S nvidia-lts
```
Install Virtualbox guest packages
If you’re installing Arch inside a Virtualbox virtual machine, install these packages:
```
pacman -S virtualbox-guest-utils xf86-video-vmware
```
Installing a Desktop Environment
GNOME
To install GNOME, install the gnome package:
```
sudo pacman -S gnome
```
##### Also consider installing GNOME Tweaks:
```
 sudo pacman -S gnome-tweaks
 ```
To enable the login screen to appear automatically at boot, run:

``` 
sudo systemctl enable gdm
```
Note: At first login, one or more GNOME apps may fail to start. You might see a spinning circle or equivelant, and then the app never appears. To prevent this situation, you should first open GNOME’s settings, then “Region and Language”, and set your info there
