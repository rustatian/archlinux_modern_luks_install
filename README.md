# Archlinux encrupted (with LVM) install
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
1. 1Gb EFI partition --> `/dev/nvme0n1p1`  
2. The rest of the space will be encrypted --> `/dev/nvme0n1p2` (action later)   
3. In case of BTRFS you'll need 3 partitions, boot (EFI), swap and root (with home).

---
#### 4. Create EFI partition
```
mkfs.fat -F32 /dev/nvme0n1p1
```

```
mkswap /dev/nvme0n1p2
```

```
swaplabel -L swappo /dev/nvme0n1p2
```

```
swapon /dev/nvme0n1p2
```
```
btrfs filesystem label /dev/nvme0n1p3 root
```

---
#### 5. Setup the encryption of the system with 512 bit effective size
```
cryptsetup -c aes-xts-plain64 --key-size 512 --hash sha512 --iter-time 3000 -y --use-random luksFormat /dev/nvme0n1p3
```
```
cryptsetup open /dev/nvme0n1p3 luks
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

#### OR for the BTRFS:
```
mkfs.btrfs /dev/mapper/luks
```

```
mount /dev/mapper/luks /mnt
```

```  
btrfs subvolume create /mnt/@
```

```
btrfs subvolume create /mnt/@home
```

```
umount /mnt
``` 

```
mount -o subvol=@,ssd,compress=lzo,discard=async,noatime,nodiratime /dev/mapper/luks /mnt
```
```
mkdir /mnt/{boot,home}
```
```
mount -o subvol=@home,ssd,compress=lzo,discard=async,noatime,nodiratime /dev/mapper/luks /mnt/home
```
```
mount /dev/nvme0n1p1 /mnt/boot
```

---
#### 7. Create filesystems on encrypted partitions (NOT NEEDED FOR THE BTRFS)  
```
mkfs.ext4 /dev/mapper/vg0-root (or mkfs.xfs /dev/mapper/vg0-root, but in case of xfs you also should install xfsprogs) or mkfs.btrfs (btrfs-progs)  
mkswap /dev/mapper/vg0-swap
```  

---

#### My reinstall copy files:
```
cp root/.backups etc/iwd/main.conf etc/environment etc/makepkg.conf etc/pacman.conf etc/locale.gen etc/locale.conf etc/mkinitcpio.conf etc/crypttab etc/vconsole.conf etc/udev/rules.d/81-wifi-powersave.rules boot/loader/entries/arch.conf /mnt/root
```

---
#### 8. Mount the new system (NOT NEEDED FOR THE BTRFS)
```
mount /dev/mapper/vg0-root /mnt # /mnt is the installed system
swapon /dev/mapper/vg0-swap # Not needed but a good thing to test
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```  

---
#### 9. Install the system also includes stuff needed for starting wifi when first booting into the newly installed system

```
pacstrap /mnt base base-devel fish vim neovim git sudo efibootmgr systemd-resolvconf mdadm iw iwd ccache linux linux-headers linux-firmware intel-ucode btrfs-progs
```

---
#### 10. Install fstab
```
genfstab -pU /mnt >> /mnt/etc/fstab
```  

Sample for the BTRFS
```
tmpfs /tmp tmpfs defaults,size=40G,noatime,mode=1777 0 0
```

---
#### 11. Make /tmp a ramdisk (add the following line to /mnt/etc/fstab)
`tmpfs	/tmp	tmpfs	defaults,noatime,mode=1777	0	0` you can also specify a size for the /tmp. To do that, just put after `defaults` `size=xG` where x is the needed size.  

---
#### 12. Enter the new system
```
arch-chroot /mnt
```  

---
#### 13. Setup system clock
```
ln -s /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
```
```
hwclock --systohc --utc
```

- /etc/vconsole.conf
```
FONT=ter-i32n
FONT_MAP=8859-2
```

---

#### 14. Set the hostname

```
echo rustatian > /etc/hostname
```

---
### Generate locale
#### 15. Uncomment wanted locales in /etc/locale.gen
```
nvim /etc/locale.gen
```
```
locale-gen
```
```
localectl set-locale LANG=en_US.UTF-8
```
```
echo LANG=en_US.UTF-8 >> /etc/locale.conf
```
```
echo LANG=pl_PL.UTF-8 >> /etc/locale.conf
```
 ---
#### 16. Set password for root and add user
```
passwd
```
```
useradd -mg users -G wheel,storage,power -s /bin/fish valery
```
```
passwd valery
```
```
visudo -> uncomment the following line --> %wheel ALL=(ALL) ALL
```

- Additional options:
```
Defaults timestamp_type=global
Defaults timestamp_timeout=15
```  

---
#### 17. Configure mkinitcpio with modules needed for the initrd image  (ORDER MATTERS!)
`vim /etc/mkinitcpio.conf`  
Add `ext4` to MODULES (or xfs, btrfs). Also, if you want to see the password screen when laptop lid is closed add `i915` (Intel) to the modules
Add `encrypt` and `lvm2` to HOOKS BEFORE filesystems   (only for the LVM)
Add `resume` AFTER `lvm2` (also has to be after `udev`)  
There is my hooks `HOOKS=(base udev autodetect modconf block encrypt lvm2 resume filesystems keyboard fsck)`  
And modules `MODULES=()` -> in case of nvidia you may add `nvidia, nvidia_modeset, nvidia_uvm and nvidia_drm`
And modules `MODULES=()` -> in case of AMD you may add `amdgpu radeon`

---
#### 18. Regenerate initrd image
```
mkinitcpio -P
```

---
#### 19. Setup systembootd
```
bootctl --path=/boot install
```

---
#### 20. Create loader.conf
```
echo 'default arch' >> /boot/loader/loader.conf
```
```
echo 'timeout 5' >> /boot/loader/loader.conf
```

---
#### 21. Create arch.conf
```
nvim /boot/loader/entries/arch.conf
```

---

#### 22. Add the following content to arch.conf  
UUID is the the one of the raw encrypted device (/dev/nvme0n1p2). It can be found with the `blkid` command
TIP: Use echo to put UUID into /boot/loader/entries/arch.conf.
```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=UUID=<UUID>:vg0 root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap rw nvidia-drm.modeset=1

---
BTRFS
title Arch Linux [rustatian]
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=UUID=c5936c6f-1db2-43dd-9797-35b75d416ded:luks:allow-discards root=/dev/mapper/luks raid0.default_layout=2 rootflags=subvol=@ rw nvidia-drm.modeset=1 
OR for the AMD
options cryptdevice=UUID=c5936c6f-1db2-43dd-9797-35b75d416ded:luks:allow-discards root=/dev/mapper/luks raid0.default_layout=2 rootflags=subvol=@ rw radeon.si_support=0 radeon.cik_support=0 amdgpu.cik_support=1 amdgpu.si_support=1 amdgpu.dpm=1 amdgpu.ppfeaturemask=<EXECUTE printf "0x%08x\n" $(cat /sys/module/amdgpu/parameters/ppfeaturemask)">
```
 
---
#### 23. Install DE

### GNOME
```
pacman -S gnome gnome-extra bluez bluez-utils pipewire pipewire-pulse networkmanager network-manager-applet gvfs gnome-keyring seahorse docker docker-compose llvm lldb gdb lld cmake perf strace tcpdump lsof iotop xdg-user-dirs xdg-utils ttf-font-awesome qemu libvirt
```
 
---
#### 24. Enable services
```
systemctl enable gdm bluetooth NetworkManager systemd-timesyncd libvirtd docker systemd-boot-update.service 
systemctl enable --user pipewire-pulse wireplumber
```

#### Docker
```
usermod -aG docker valery
```

### Nvidia Wayland

```bash
cat >> /etc/modprobe.d/nvidia.conf <<EOF
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia-drm modeset=1
EOF
```
```
mkinitcpio -P
```
```
systemctl enable nvidia-{hibernate,suspend,resume}
```   

### Power management (if have wifi disconnections)

```
pcie_port_pm=off pcie_aspm.policy=performance
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
