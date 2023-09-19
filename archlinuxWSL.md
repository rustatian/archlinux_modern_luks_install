#### IMPORTANT: Do these steps inside the Linux (you might temporarily install Ubuntu WSL)

#### Download a bootstrap archive:

```
curl https://ftp.icm.edu.pl/pub/Linux/dist/archlinux/iso/2023.06.01/archlinux-bootstrap-x86_64.tar.gz -o archlinux-bootstrap-x86_64.tar.gz
```

#### Extract it:

```
sudo tar -xzf archlinux-bootstrap-x86_64.tar.gz
```

#### Add root FS to the new archive:

```
sudo tar -czvf ../ArchWSL.tar.gz .
```

#### Copy to the needed directory.

#### Import the distro with the following command:

```
wsl --import <desired_distro_name> <path: D:\foo\bar> ArchWSL.tar.gz
```

#### Log in:

```
wsl -d <desired_distro_name>
```

#### Generate mirror list for your country:

- https://archlinux.org/mirrorlist/

#### Get any enrty from the generated list and put it into the WSL ArchLinux mirrorlist:


#### Put the first mirror to the ArchLinux mirrorlist:

```
echo 'Server = http://arch.midov.pl/arch/$repo/os/$arch' >> /etc/pacman.d/mirrorlist
```

#### Populate the ArchLinux keyring:

```
pacman-key --init
```

``` 
pacman-key --populate
```

```
pacman -Sy archlinux-keyring
```

#### Install the base system:

```
pacman -S base base-devel fish vim neovim git sudo ccache linux linux-headers linux-firmware
```

#### Configure the system

- Configure the timezone
```
ln -sf /usr/share/zoneinfo/Europe/<YOUR_TIMEZONE_CITY> /etc/localtime
```

- Sync hardware time in UTC
```
hwclock --systohc --utc
```

- Set up the host name
```
echo rustatian > /etc/hostname
```

- Uncomment needed locales
```
nvim /etc/locale.gen
```

- Generate locales
```
locale-gen
```

- Set default locale
```
localectl set-locale LANG=en_US.UTF-8
```

#### Create the user and password:

- Set the root password
```
passwd
```

- Create a user (note: you may use any shell instead of `/bin/fish`)
```
useradd -mg users -G wheel,storage,power -s /bin/fish valery
```

- Set the password for your user
```
passwd valery
```


```
visudo -> uncomment the following line --> %wheel ALL=(ALL) ALL
```

- OPTIONAL: Add to the Defaults section (visudo)
```
Defaults timestamp_type=global
Defaults timestamp_timeout=15
```

#### Install the needed packages (my example):

```
pacman -S gnome gnome-extra pipewire pipewire-pulse gnome-keyring seahorse docker docker-compose clang llvm lldb gdb lld cmake perf
strace tcpdump lsof iotop xdg-user-dirs xdg-utils ttf-font-awesome
```

#### Generate unique machine ID:

```
systemd-machine-id-setup 
```

#### Set the default user (`/etc/wsl.conf`):

``` 
[boot]
systemd=true
[user]
default=valery
```

#### Shutdown the WSL and enjoy with ArchLinux:

```
wsl --shutdown
```
