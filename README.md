# archlinux modern luks install
Archlinux encrypted (LUKS) install guide

#### Install ARCH Linux with encrypted file-system and UEFI
#### The official installation guide (https://wiki.archlinux.org/index.php/Installation_Guide) contains a more verbose description.
---
#### Download the archiso image from https://www.archlinux.org/
#### 1 .Copy to a usb-drive
`dd if=archlinux.img of=/dev/sdX bs=16M && sync # on linux`
#### Or for the GUI install you can use etcher from https://www.balena.io/etcher/  

---
#### 2. This assumes a wifi only system... (wifi-menu removed from the installer image since June 2020)  
```
1. iwctl 
2. station list
3. station <generally wlan0> connect <wifi network name SSID> -> station wlan0 connect 0xdev
enter your password and exit (type exit -> enter)
```

---
#### 3. Create partitions (nvme in case of NVME disk or sda in case of HDD)
`cfdisk /dev/nvme0n1`  
1. 512MB EFI partition --> `/dev/nvme0n1p1`  
2. The rest of the space will be encrypted --> `/dev/nvme0n1p2` (action later)   

---
#### 4. Create EFI partition
`mkfs.fat -F32 /dev/nvme0n1p1`

---
#### 5. Setup the encryption of the system with 512 bit effective size
```
cryptsetup -c aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 3000 -y --use-random luksFormat /dev/nvme0n1p2
cryptsetup luksOpen /dev/nvme0n1p2 luks
```

---
#### Create encrypted partitions
#### 6. This creates one partions for root, modify if /home or other partitions should be on separate partitions  
```
pvcreate /dev/mapper/luks
vgcreate vg0 /dev/mapper/luks
lvcreate --size 16G vg0 --name swap (If you are planning to use deep_sleep or hybernate, you should set size to the Ram * 1.5)
lvcreate -l +100%FREE vg0 --name root
```  

---
#### 7. Create filesystems on encrypted partitions  
```
mkfs.ext4 /dev/mapper/vg0-root (or mkfs.xfs /dev/mapper/vg0-root, but in case of xfs you also should install xfsprogs)  
mkswap /dev/mapper/vg0-swap
```  

---
#### 8. Mount the new system 
```
mount /dev/mapper/vg0-root /mnt # /mnt is the installed system
swapon /dev/mapper/vg0-swap # Not needed but a good thing to test
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```  

---
#### 9. Install the system also includes stuff needed for starting wifi when first booting into the newly installed system
`pacstrap /mnt base base-devel zsh vim neovim git sudo efibootmgr iwd dhcpcd lvm2 linux linux-headers linux-firmware` + `xfsprogs` in case of XFS filesystem

---
#### 10. Install fstab
`genfstab -pU /mnt >> /mnt/etc/fstab`  

---
#### 11. Make /tmp a ramdisk (add the following line to /mnt/etc/fstab)
`tmpfs	/tmp	tmpfs	defaults,noatime,mode=1777	0	0` you can also specify a size for the /tmp. To do that, just put after `defaults` `size=xG` where x is the needed size.  

---
#### 12. Enter the new system
`arch-chroot /mnt`  

---
#### 13. Setup system clock
```
ln -s /usr/share/zoneinfo/Europe/Minsk /etc/localtime <USE YOUR CITY!>
hwclock --systohc --utc
```

---

#### 14. Set the hostname
`echo <YOU HOST NAME> > /etc/hostname`  

---
### Generate locale
#### 15. Uncomment wanted locales in /etc/locale.gen
```
vim /etc/locale.gen
locale-gen
localectl set-locale LANG=en_US.UTF-8
echo LANG=en_US.UTF-8 >> /etc/locale.conf
```
 ---
#### 16. Set password for root and add user
```
passwd
useradd -mg users -G wheel,storage,power -s /bin/zsh <MYUSERNAME> (or /bin/bash)
passwd <MYUSERNAME>

visudo -> uncomment the following line --> %wheel ALL=(ALL) ALL
```  

---
#### 17. Configure mkinitcpio with modules needed for the initrd image  (ORDER MATTERS!)
`vim /etc/mkinitcpio.conf`  
Add `ext4` to MODULES (or xfs). Also, if you want to see the password screen when laptop lid is closed add `i915` (Intel) to the modules
Add `encrypt` and `lvm2` to HOOKS BEFORE filesystems   
Add `resume` AFTER `lvm2` (also has to be after `udev`)  
There is my hooks `HOOKS=(base udev autodetect modconf block encrypt lvm2 resume filesystems keyboard fsck)`  
And modules `MODULES=(i915 xfs)`  

---
#### 18. Regenerate initrd image
`mkinitcpio -P`

---
#### 19. Setup systembootd
`bootctl --path=/boot install`

---
#### 20. Create loader.conf
```
echo 'default arch' >> /boot/loader/loader.conf
echo 'timeout 5' >> /boot/loader/loader.conf
```

---
#### 21. Create arch.conf
`vim /boot/loader/entries/arch.conf`

---

#### 22. Add the following content to arch.conf  
UUID is the the one of the raw encrypted device (/dev/nvme0n1p2). It can be found with the `blkid` command
TIP: Use echo to put UUID into /boot/loader/entries/arch.conf.
```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID>:vg0 root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap rw zswap.enabled=1
```
 
---
#### 23. Install favorite DE (I use xfce4)
```
pacman -S xorg xfce4 xfce4-goodies nvidia mesa mesa-demos lightdm lightdm-gtk-greeter pulseaudio pulseaudio-bluetooth blueman bluez bluez-utils networkmanager network-manager-applet

pulseaudio-bluetooth blueman bluez bluez-utils used to setup bluetooth
```

#### Or GNOME
```
pacman -S gnome gnome-extra pulseaudio-bluetooth blueman bluez bluez-utils
```
 
---
#### 24. Enable services
```
systemctl enable lightdm bluetooth iwd dhcpcd NetworkManager
```

#### Or in case of GNOME:
```
systemctl enable gdm bluetooth iwd dhcpcd NetworkManager
```

---
#### 25. Exit new system and go into the cd shell
`exit`

---
#### 26. Unmount all partitions
```
umount -R /mnt
swapoff -a
```

---
#### 27. Reboot into the new system, don't forget to remove the cd/usb
`reboot`
