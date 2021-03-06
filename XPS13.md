
# Install Arch Linux - Fully encrypted disk with Btrfs filesystem

**Warning: This setup will completely wipe your system and will only install Arch linux.**

## 1. Set variables
```{r, engine='bash', count_lines}
export INSDRIVE=/dev/nvme0n1
export SWAPPARTITION=/dev/nvme0n1p2
export INSPARTITION=/dev/nvme0n1p3
export BTRFSNAME=btrfsroot
export CRYPTNAME=cryptroot

export MOUNTDIR=/mnt/ARCH
sudo mkdir $MOUNTDIR
```

## 2. Setup disk and partitions

Remove legacy partition information

```{r, engine='bash', count_lines}
sudo sgdisk --zap-all $INSDRIVE
sudo sgdisk -og $INSDRIVE
```

Create the 2 partitions. One for swap and the other for / (the root filesystem). 
    
```{r, engine='bash', count_lines}
sudo sgdisk --clear \
         --new=1:0:+5MiB   --typecode=1:ef02 --change-name=1:bios_boot \
         --new=2:0:+8GiB   --typecode=2:8200 --change-name=2:cryptswap \
         --new=3:0:0       --typecode=3:8300 --change-name=3:cryptsystem \
           $INSDRIVE
           
fdisk -l $INSDRIVE
```


Encrypt the disk

```{r, engine='bash', count_lines}
sudo cryptsetup luksFormat --cipher aes-xts-plain64 --key-size 512 --hash sha256 --use-random $INSPARTITION
sudo cryptsetup luksOpen $INSPARTITION $CRYPTNAME
```


Create (sub)volumes

```{r, engine='bash', count_lines}
sudo mkfs.btrfs -L $BTRFSNAME /dev/mapper/$CRYPTNAME
sudo mount -t btrfs -o defaults,discard,ssd,space_cache,noatime,compress=lzo,autodefrag,subvol=/ /dev/mapper/$CRYPTNAME $MOUNTDIR
btrfs filesystem show
cd $MOUNTDIR
sudo btrfs subvol create $MOUNTDIR/boot
sudo btrfs subvol create $MOUNTDIR/home
cd
sudo umount $MOUNTDIR
```


Mount btrfs (sub)volumes

```{r, engine='bash', count_lines}
sudo mount -o noatime,compress=lzo,discard,ssd,defaults,subvol=/ /dev/mapper/$CRYPTNAME $MOUNTDIR
#sudo mkdir $MOUNTDIR/{home,var}
sudo mount -o noatime,compress=lzo,discard,ssd,defaults,subvol=/boot /dev/mapper/$CRYPTNAME $MOUNTDIR/boot
sudo mount -o noatime,compress=lzo,discard,ssd,defaults,subvol=/home /dev/mapper/$CRYPTNAME $MOUNTDIR/home
sudo sync
```

## 3. Install the base system
```{r, engine='bash', count_lines}
sudo pacstrap $MOUNTDIR base base-devel btrfs-progs openssh net-tools wpa_supplicant networkmanager xf86-video-intel vim zsh
sudo pacman -S broadcom-wl-dkms bluez-firmware linux-headers
```

Optional kernels:
```{r, engine='bash', count_lines}
sudo pacstrap $MOUNTDIR linux-zen linux-lts
```

Generate fstab
```{r, engine='bash', count_lines}
sudo genfstab -p $MOUNTDIR | sudo tee -a $MOUNTDIR/etc/fstab > /dev/null
```

## 4. Chroot to **new system**
```{r, engine='bash', count_lines}
sudo arch-chroot  $MOUNTDIR
```
```{r, engine='bash', count_lines}
export INSDRIVE=/dev/nvme0n1
export SWAPPARTITION=/dev/nvme0n1p2
export INSPARTITION=/dev/nvme0n1p3
export BTRFSNAME=btrfsroot
export CRYPTNAME=cryptroot
```

#### Update package database
```{r, engine='bash', count_lines}
pacman -Syy
```

#### timezone
```{r, engine='bash', count_lines}
ln -s /usr/share/zoneinfo/Europe/Amsterdam /etc/localtime
hwclock --systohc --utc
```


#### Set the hostname
```{r, engine='bash', count_lines}
echo "Joker" > /etc/hostname
```


#### Update locale
```{r, engine='bash', count_lines}
echo 'en_US.UTF-8 UTF-8' >> /etc/locale.gen
echo 'en_US ISO-8859-1' >> /etc/locale.gen
locale-gen
```


### Set virtual console lang and font (FONT NAME NEEDS TO BE FIXED)
```{r, engine='bash', count_lines}
echo keymap=en >> /etc/keymaps
echo consolefont=Lat2-Terminus16 >> /etc/consolefont
echo FONT=ter-p24n > /etc/vconsole.conf
echo FONT_MAP=8859-1 >> /etc/vconsole.conf
```

### Create the keyfile
```{r, engine='bash', count_lines}
dd bs=512 count=4 if=/dev/urandom of=/crypto_keyfile.bin
cryptsetup luksAddKey $INSPARTITION /crypto_keyfile.bin
chmod 000 /crypto_keyfile.bin
```

### Edit kernel modules (HOOKS) in /etc/mkinitcpio.conf 
MODULES=(intel_agp i915 nvme)
BINARIES=""
#FILES="/etc/modprobe.d/modprobe.conf"
FILES="/crypto_keyfile.bin"
HOOKS=(base udev autodetect modconf block consolefont keymap encrypt lvm2 resume filesystems keyboard fsck btrfs)

```{r, engine='bash', count_lines}
touch /etc/modprobe.d/modprobe.conf
mkinitcpio -p linux
```

### set password
```{r, engine='bash', count_lines}
passwd root
```


# Enable services
```{r, engine='bash', count_lines}
systemctl enable NetworkManager # you can use connman instead of this
systemctl enable sshd
```


# grub
```{r, engine='bash', count_lines}
pacman -Sy grub os-prober mtools dosfstools fuse2 
```

```{r, engine='bash', count_lines}
echo 'GRUB_ENABLE_CRYPTODISK=y' >> /etc/default/grub 
echo 'GRUB_CMDLINE_LINUX="cryptdevice='$INSPARTITION':'$CRYPTNAME'"' >> /etc/default/grub 
```

for SSD disk you need to add "allow-discards" enables TRIM support:

```{r, engine='bash', count_lines}
echo 'GRUB_CMDLINE_LINUX="cryptdevice='$INSPARTITION':'$CRYPTNAME':allow-discards"' >> /etc/default/grub 
```

#### Hibernation
#### use SWAPPARTITION=/dev/nvme0n1p2 for hibernation
#GRUB_CMDLINE_LINUX_DEFAULT="resume=/dev/nvme0n1p2"


```{r, engine='bash', count_lines}
grub-install --target=i386-pc $INSDRIVE
grub-mkconfig -o /boot/grub/grub.cfg
```
##### ignore the warning -->   WARNING: Failed to connect to lvmetad. Falling back to device scanning.




## Reboot the system
```{r, engine='bash', count_lines}
reboot
```


# POST installation

## add user
```{r, engine='bash', count_lines}
useradd -m -g users -G wheel,storage,power -s /bin/zsh fdiblen
passwd fdiblen
```

## /etc/crypttab
#SWAP --> /dev/nvme0n1p2
swap /dev/nvme0n1p2 /dev/urandom swap,cipher=aes-cbc-essiv:sha256,size=256

```{r, engine='bash', count_lines}
ls -l /dev/mapper/
```

## /etc/fstab
/dev/mapper/swap swap swap defaults 0 0

```{r, engine='bash', count_lines}
reboot
```


# AUR Helper (yay)

```{r, engine='bash', count_lines}
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

# Wifi driver
yay -S bcm4350-firmware


# Extras

## SSD trim
```{r, engine='bash', count_lines}
sudo systemctl enable fstrim.timer
```

## Fonts
```{r, engine='bash', count_lines}
sudo pacman -S powerline-fonts awesome-terminal-fonts freetype2 terminus-font

echo FONT=Lat2-Terminus16 >> /etc/vconsole.conf
```


## useful packages
```{r, engine='bash', count_lines}
sudo pacman -S zsh htop sudo git wget curl powertop
sudo pacman -S tmux openssl openssh pkgfile unzip unrar p7zip
```
Optional:
```{r, engine='bash', count_lines}
sudo pacman -S linux-zen-headers linux-lts-headers
```{r, engine='bash', count_lines}


## setup sudo and allow wheel group
export EDITOR=vim
visudo

## switch to normal user and continue as this user
su fdiblen && cd


## X-server
```{r, engine='bash', count_lines}
sudo pacman -S xorg xorg-xinit xterm xorg-xeyes xorg-xclock xorg-xrandr xf86-video-intel
```

## i3(-gaps) 
```{r, engine='bash', count_lines}
yay --needed --noconfirm -S i3-gaps polybar-git compton-git dunst rofi-git termite-git
```

# Lightdm - desktop(login) manager
```{r, engine='bash', count_lines}
yay --needed --noconfirm -S lightdm-gtk-greeter
systemctl enable lightdm
```
#### edit /etc/lightdm/lightdm.conf
[Seat:*]
greeter-session=lightdm-gtk-greeter



## Gnome
```{r, engine='bash', count_lines}
sudo pacman -S gnome-shell gdm gnome-terminal gnome-control-center gnome-tweak-tool
sudo systemctl enable gdm
reboot
```


## Printing
yay -S --needed cups gutenprint libpaper foomatic-db-engine ghostscript gsfonts foomatic-db cups-pdf system-config-printer

sudo systemctl enable org.cups.cupsd.service
sudo systemctl enable cups-browsed.service
sudo systemctl start org.cups.cupsd.service
sudo systemctl start cups-browsed.service



## Extra

```{r, engine='bash', count_lines}
yay -S --needed chrome-gnome-shell-git chrome-shutdown-hook pamac-aur \
    numix-circle-icon-theme-git \
    atom-editor-bin \
    tlp gtop \
    wps-office \
    vertex-themes flatplat-theme-git moka-icon-theme-git paper-gtk-theme-git \ 
    opendesktop-fonts ttf-ms-fonts ttf-google-fonts-git nerd-fonts-git \
    vlc \
    inkscape \
    dropbox nautilus-dropbox \
    firefox google-chrome flashplugin \              
    p7zip unrar tar rsync file-roller seahorse-nautilus nautilus-share zlib unzip zip zziplib \ 
    zim \
    spotify \
    texstudio biber texlive-most \
    archlinux-artwork \
    xclip \
    redshift \
    pyenv
```


## Shadowsocks
TODO


## Powertop
TODO


## Grub make-up
```{r, engine='bash', count_lines}
yay -S grub2-theme-arch-leap
```
### /etc/default/grub

GRUB_BACKGROUND="/boot/grub/themes/arch-leap/background.png"
GRUB_THEME="/boot/grub/themes/arch-leap/theme.txt"

```{r, engine='bash', count_lines}
sudo grub-mkconfig -o /boot/grub/grub.cf
```

## Preload
sudo pacman -S preload
sudo systemctl enable preload.service


## Docker
docker docker-compose

#kitematic
#kubernetes


# Issues

## gnome-terminal does not start
set language in 
Settings --> Region & Language --> Language

# Links
https://wiki.archlinux.org/index.php/general_recommendations
