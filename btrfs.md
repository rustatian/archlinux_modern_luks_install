
mount /dev/mapper/luks /mnt  
btrfs subvolume create /mnt/@  
btrfs subvolume create /mnt/@home  
umount /mnt  

-----
mount -o subvol=@,ssd,compress=lzo,noatime,nodiratime /dev/mapper/luks /mnt  
mkdir /mnt/{boot,home}  
mount -o subvol=@home,ssd,compress=lzo,noatime,nodiratime /dev/mapper/luks /mnt/home  

---

