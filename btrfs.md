```
mount /dev/mapper/luks /mnt  
btrfs subvolume create /mnt/@  
btrfs subvolume create /mnt/@home  
umount /mnt  

-----
mount -o subvol=@,ssd,compress=lzo,noatime,nodiratime /dev/mapper/luks /mnt  
mkdir /mnt/{boot,home}  
mount -o subvol=@home,ssd,compress=lzo,noatime,nodiratime /dev/mapper/luks /mnt/home  

---

title Arch Linux [48d90782]
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=UUID=c5936c6f-1db2-43dd-9797-35b75d416ded:luks:allow-discards root=/dev/mapper/luks rootflags=subvol=@ rw nvidia-drm.modeset=1 
```
