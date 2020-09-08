# archlinux modern luks install
Archlinux encrypted (LUKS) install guide

#### Install ARCH Linux with encrypted file-system and UEFI
#### The official installation guide (https://wiki.archlinux.org/index.php/Installation_Guide) contains a more verbose description.
---
#### Download the archiso image from https://www.archlinux.org/
#### Copy to a usb-drive
`dd if=archlinux.img of=/dev/sdX bs=16M && sync # on linux`
#### Or for the GUI install you can use etcher from https://www.balena.io/etcher/  

---
#### This assumes a wifi only system... (wifi-menu removed from the installer image since June 2020)  
```
iwctl 
station list
station <generally wlan0> connect <wifi network name SSID>
enter your password and exit (type exit -> enter)
```

---
#### Create partitions
`cfdisk /dev/nvme0n1`  
1 512MB EFI partition  
2 100% size partiton(to be encrypted)  

---
#### Create EFI partition
`mkfs.fat -F32 /dev/nvme0n1p1`

---
#### Setup the encryption of the system with 256 bit effective size
```
cryptsetup -c aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 3000 -y --use-random luksFormat /dev/nvme0n1p2
cryptsetup luksOpen /dev/nvme0n1p2 luks
```

---
#### Create encrypted partitions
#### This creates one partions for root, modify if /home or other partitions should be on separate partitions  
```
pvcreate /dev/mapper/luks
vgcreate vg0 /dev/mapper/luks
lvcreate --size 16G vg0 --name swap (If you are planning to use deep_sleep or hybernate, you should set size to the Ram * 1.5)
lvcreate -l +100%FREE vg0 --name root
```  

---
#### Create filesystems on encrypted partitions  
```
mkfs.ext4 /dev/mapper/vg0-root (or mkfs.xfs /dev/mapper/vg0-root)  
mkswap /dev/mapper/vg0-swap
```  

---
#### Mount the new system 
```
mount /dev/mapper/vg0-root /mnt # /mnt is the installed system
swapon /dev/mapper/vg0-swap # Not needed but a good thing to test
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```  

---
#### Install the system also includes stuff needed for starting wifi when first booting into the newly installed system
`pacstrap /mnt base base-devel zsh vim git sudo efibootmgr iwd linux linux-headers linux-firmware` 

---
#### Install fstab
`genfstab -pU /mnt >> /mnt/etc/fstab`  

---
#### Make /tmp a ramdisk (add the following line to /mnt/etc/fstab)
`tmpfs	/tmp	tmpfs	defaults,noatime,mode=1777	0	0`  

---
#### Change relatime on all non-boot partitions to noatime (reduces wear if using an SSD)

---
#### Enter the new system
`arch-chroot /mnt`  

---
#### Setup system clock
```
ln -s /usr/share/zoneinfo/Europe/Zurich /etc/localtime
hwclock --systohc --utc
```

---

#### Set the hostname
`echo MYHOSTNAME > /etc/hostname`  

---
### Generate locale
#### Uncomment wanted locales in /etc/locale.gen
```
vim /etc/locale.gen
locale-gen
localectl set-locale LANG=en_US.UTF-8
```

---
#### To avoid problems with gnome-terminal set locale system wide
```
echo LANG=en_US.UTF-8 >> /etc/locale.conf
echo LC_ALL= >> /etc/locale.conf
```

 ---
#### Set password for root
```
passwd
useradd -m -g users -G wheel,storage,power -s /bin/zsh MYUSERNAME
passwd MYUSERNAME
```  

---
#### Configure mkinitcpio with modules needed for the initrd image  
`vim /etc/mkinitcpio.conf`  
Add `ext4` to MODULES  
Add `encrypt` and `lvm2` to HOOKS before filesystems   
Add `resume` after `lvm2` (also has to be after `udev`)  

---
#### Regenerate initrd image
`mkinitcpio -p linux`

---
#### Setup systembootd (grub will not work on nvme at this moment)
`bootctl --path=/boot install`

---
#### Create loader.conf
`echo 'default arch' >> /boot/loader/loader.conf`
`echo 'timeout 5' >> /boot/loader/loader.conf`

---
#### Create arch.conf
`vim /boot/loader/entries/arch.conf`

---

#### Add the following content to arch.conf
#### <UUID> is the the one of the raw encrypted device (/dev/nvme0n1p2). It can be found with the 'blkid' command
```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID>:vg0 root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap rw intel_pstate=no_hwp
```
---
#### Exit new system and go into the cd shell
`exit`

---
#### Unmount all partitions
```
umount -R /mnt
swapoff -a
```

---
#### Reboot into the new system, don't forget to remove the cd/usb
`reboot`
