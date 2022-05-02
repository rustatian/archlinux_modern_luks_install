```
mdadm --create --verbose --level=0 --metadata=1.2 --raid-devices=2 /dev/md/md1 /dev/nvme0n1p2 /dev/nvme1n1p2
```

```
mdadm --detail --scan >> /etc/mdadm.conf
```

### mkinitcpio.conf
```
MODULES=(btrfs intel_lpss_pci raid0 nvidia nvidia_modeset nvidia_uvm nvidia_drm)

HOOKS=(base udev autodetect keyboard modconf block **mdadm_udev** filesystems fsck)
```

```
BINARIES=(mdmon)
```

### Block device naming

```
swaplabel -L "new label" /dev/XXX
```

```
btrfs filesystem label /dev/XXX "new label"
```
