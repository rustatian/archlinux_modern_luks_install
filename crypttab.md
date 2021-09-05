1. Create the key file (`/root/.keyfile-1`), actually, you may choose any place  
`dd if=/dev/urandom of=/boot/keyfile bs=1024 count=4`
2. PERMISSIONS: `chmod 0400 /root/.keyfile-1`  
3. Add key: `cryptsetup -v luksAddKey /dev/<DISK> /root/.keyfile-1`  
4. Update `/etc/crypttab`. You can grab disk UUIDs via `blkid`    
Sample:
```
backups UUID=<UUID> /root/.backups-keyfile luks
data UUID=<UUID> /root/.data-keyfile luks,discard
```
5. Update `/etc/fstab`
```
#Data
/dev/mapper/data /run/media/valery/data xfs defaults 0 0

#Backups
/dev/mapper/backups /run/media/valery/backups xfs defaults 0 0
```
