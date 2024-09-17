# INSTALLING DEBIAN -- THE ARCH LINUX WAY (VIA DEBOOTSTRAP)
Instructions how to install Debian using debootstrap (LUKS)<br><br>
<b>Disclaimer:  This technical advice is provided without warranty of any kind. The user assumes all responsibility and risk for its use. The provider shall not be liable for any damages arising from the use of this advice.</b><br>
<br>
Installing Debian via debootstrap gives you better control over Disk partitoning, package installation, and system configuration.<br>
Original work by:  https://gist.github.com/varqox/42e213b6b2dde2b636ef<br>
Added:  Debian Live CD, LUKS, LVM, Timesync, Additional Steps<br>
<br>
Get the latest Debian Live CD (https://www.debian.org/CD/live/)<br>
<br>
<b>[Optional]</b>
If you want to perform the Debian installation remotely.<br>
Install openssh-server, add new user to sudo group then ssh to Live CD.<br>
Create user and set password:
```bash
useradd USERNAME -m -s /bin/bash
passwd USERNAME
```
<b>[Optional]</b>
If you intend to use `sudo`:
* Install `sudo`:
    ```bash
    apt install sudo
    ```
* Add the new user to group `sudo`:
    ```bash
    usermod -aG sudo USERNAME
    ```
<br>
<br>

Sample Disk Layout [fdisk -l]:

<pre>
NOTE:  If 'Disklabel type: gpt' then a 'BIOS boot' is required.
=================================================================
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VBOX HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 92C95383-827F-4D44-9EDC-30BBEF0B314A

Device       Start      End  Sectors Size Type
/dev/sda1     2048     4095     2048   1M BIOS boot
/dev/sda2     4096  2101247  2097152   1G Linux filesystem
/dev/sda3  2101248 41940991 39839744  19G Linux filesystem
</pre>

<pre>
NOTE:  If 'EFI/BIOS' then an 'EFI Partition' is required.
=================================================================
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VBOX HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xcd1313cb

Device     Boot   Start      End  Sectors  Size Id Type
/dev/sda1          2048  1026047  1024000  500M ef EFI (FAT-12/16/32)
/dev/sda2       1026048  3123199  2097152    1G 83 Linux
/dev/sda3       3123200 41943039 38819840 18.5G 83 Linux
</pre>

<br>
<br>

Format the /dev/sda2 boot partition with ext2:
```bash
mkfs.ext2 -c -L boot -m 0 /dev/sda2
```
-c Check the device for bad blocks before creating the file system<br>
-m reserved-blocks-percentage<br>
-L new-volume-label<br>

<b>EFI</b>
```bash
mkfs.vfat -F 32 /dev/sda1
```
<br>
<br>

Create new LUKS partition:
```bash
cryptsetup luksFormat --type luks2 <device>
```
* Low powered CPU (Intel Celeron, Intel Core Duo)
```bash
cryptsetup luksFormat --type luks2 \
	--sector-size 4096 \
	--cipher xchacha20,aes-adiantum-plain64 \
	--hash sha256 --key-size 256 <device>
```
<br>
<br>

Open LUKS<br>
Note:  NAME is important to /etc/crypttab
```bash
cryptsetup open <device> NAME
```
```bash
		mkfs.ext4 <device>
```
	Or:
 
```bash
		Create new Logical Volume
		pvcreate /dev/mapper/sda3_crypt
		vgcreate debian-vg /dev/mapper/sda3_crypt

		lvcreate -L 2G debian-vg -n swap
		lvcreate -l +100%FREE debian-vg -n root
		mkswap /dev/mapper/debian--vg-swap
		mkfs.ext4 -m 1 /dev/mapper/debian--vg-root
```
<br>
<br>

Install debootstrap<br>
Get the latest debootstrap:  https://deb.debian.org/debian/pool/main/d/debootstrap/debootstrap<br>
```bash
curl --remote-name https://deb.debian.org/debian/pool/main/d/debootstrap/debootstrap_1.0.137_all.deb
```
Then install it:
```bash
dpkg -i debootstrap_*.*.*_all.deb
```
* If error installing 'debootstrap_*.*.*_all.deb'
```bash
apt install distro-info
If prompted, run 'apt --fix-broken install'
```
<br>
<br>

Mount filesystem (LVM partitioning)
```bash
mount /dev/mapper/debian--vg-root /mnt
mkdir /mnt/boot
mount /dev/sda2 /mnt/boot
```
<b>EFI</b>
```bash
mkdir /mnt/boot/efi
mount /dev/sdaX /mnt/boot/efi
```

<br>
<br>

Install Debain Base System
Usage: `debootstrap --arch ARCH RELEASE DIR MIRROR`
```bash
debootstrap --arch amd64 stable /mnt https://deb.debian.org/debian
```

<br>
<br>

<b>[Optional]</b>
Change the UUID
Useful if you want to revert to the previous partition UUID that /etc/crypttab points to.
But not needed if you intend to run 'update-initramfs -u' at the end of this installation.
```bash
cryptsetup luksUUID /dev/sda5 --uuid "$newuuid"
cryptsetup luksUUID /dev/sda5 --uuid "9a8a3d4f-ee64-4250-b9c4-4371afcb3eac"
```

<br>
<br>

Chroot into installed base system
```bash
mount --make-rslave --rbind /proc /mnt/proc
mount --make-rslave --rbind /sys /mnt/sys
mount --make-rslave --rbind /dev /mnt/dev
mount --make-rslave --rbind /run /mnt/run
chroot /mnt /bin/bash
```

<br>
<br>

Fill `/etc/fstab`:
```bash
cat > /etc/fstab << HEREDOC
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
UUID=$(lsblk --noheadings -o UUID /dev/sda2) /boot           ext2    defaults        0       2
/dev/mapper/debian--vg-root /               ext4    errors=remount-ro 0       1
/dev/mapper/debian--vg-swap none            swap    sw              0       0
HEREDOC
```
<b>EFI</b><br>
Use 'lsblk -f'
```bash
UUID=C087-08D4  /boot/efi       vfat    umask=0077      0       1
```


<br>
<br>

Choose timezone
```bash
dpkg-reconfigure tzdata
```
Sync Date and Time
```bash
apt install systemd-timesyncd
```
To display the current time and date, run:
```bash
timedatectl
```

<br>
<br>


Configure locales
```bash
apt install locales
dpkg-reconfigure locales
```
E.g. select `en_US.UTF-8`

<br>
<br>

Fill `/etc/apt/sources.list`:
```bash
apt install lsb-release
CODENAME=$(lsb_release --codename --short)
cat > /etc/apt/sources.list << HEREDOC
deb https://deb.debian.org/debian/ $CODENAME main contrib non-free non-free-firmware
deb-src https://deb.debian.org/debian/ $CODENAME main contrib non-free non-free-firmware

deb https://security.debian.org/debian-security $CODENAME-security main contrib non-free non-free-firmware
deb-src https://security.debian.org/debian-security $CODENAME-security main contrib non-free non-free-firmware

deb https://deb.debian.org/debian/ $CODENAME-updates main contrib non-free non-free-firmware
deb-src https://deb.debian.org/debian/ $CODENAME-updates main contrib non-free non-free-firmware
HEREDOC
```
Then check if everything is as you like:
```bash
nano /etc/apt/sources.list
```
Finally, run:
```bash
apt update
```


<br>
<br>

Fill `/etc/crypttab`:
```bash
echo "sda3_crypt UUID=$(lsblk --noheadings -o UUID /dev/sda3 |head -n 1) none luks,discard" >> /etc/crypttab
```


<br>
<br>

Install kernel<br>
To boot the system you will need Linux kernel and a boot loader. You can search available kernel images by running:
```bash
apt search linux-image
```
Then install your chosen kernel image, e.g.:
```bash
apt install linux-image-amd64
# LUKS/LVM
apt install linux-image-amd64 cryptsetup lvm2 cryptsetup-initramfs
```


<br>
<br>

Set hostname
```bash
echo "MYHOSTNAME" > /etc/hostname
```
where `MYHOSTNAME` is the hostname you want to set.

Then update `/etc/hosts`:
```bash
cat > /etc/hosts << HEREDOC
127.0.0.1 localhost
127.0.1.1 $(cat /etc/hostname)

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
HEREDOC
```


<br>
<br>

With the Debian Base System already installed (via debootstrap).<br>
You can either install a Networking package to work with your Debian Base System<br>
or install additional packages that you may need.<br>
Examples:  vlc, mpv, firefox-esr, chromium, kde, blender, audacity, lighttpd, build-essential screen samba<br>
<br>
Install a Networking package or Additional packages<br>
<b>Note:  Don't install any boot loader or EFI packages here, it will be done in the next step.</b>
```bash
apt install network-manager
# XFCE
apt install network-manager xfce4 xfce4-goodies lightdm lightdm-gtk-greeter firefox-esr chromium filezilla vlc mpv openssh-server
```

<br>
<br>

Here's you chance to customize your configuration the way you want it.<br>
For example:<br>
* Edit your Samba configuration:  /etc/samba/smb.conf<br>
* Edit your screenrc configuration:  /etc/screenrc<br>
* Create a systemd service.

<br>
<br>

Install boot loader<br>
This will not overwrite the current grub installation on disk, we will do it at the very end of these instructions.
```bash
apt install grub2
```


<br>
<br>

<b>[Optional]</b>
Set root's password
```bash
passwd
```
Remember that an unprivileged user has to be created because, by default ssh'ing onto `root` is forbidden.

## Create an unprivileged user

Create user and set password:
```bash
useradd USERNAME -m -s /bin/bash
passwd USERNAME
```
Replace `USERNAME` with username of an user you want to create.

(Optional) If you intend to use `sudo`:
* Install `sudo`:
    ```bash
    apt install sudo
    ```
* Add the new user to group `sudo`:
    ```bash
    usermod -aG sudo USERNAME
    ```

<br>
<br>


Configure console keyboard layout
```bash
apt install console-setup console-setup-linux
```






<br>
<br>

To set your console font
```bash
dpkg-reconfigure console-setup 
```


<br>
<br>


Enable os_prober in grub<br>
This will make grub search for and add to menu other systems like Windows or other Linux distribution.
```bash
(cat /etc/default/grub; echo GRUB_DISABLE_OS_PROBER=false) | sudo tee /etc/default/grub && sudo update-grub
```



<br>
<br>

Finish installation
<b>EFI</b>
```bash
apt install efibootmgr efivar grub-efi grub-efi-amd64-signed
```
Grub Install
Where `/dev/GRUBDISK` is the disk on which you want grub to be installed e.g. `/dev/sda` (don't confuse it with a partition which is e.g. `/dev/sda1`).
```bash
update-grub && grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB /dev/GRUBDISK
update-grub && grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB /dev/sda
update-grub && grub-install --target=i386-pc /dev/sda
### EFI with Secure Boot ###
update-grub && grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB /dev/sda
efibootmgr -c -d /dev/sda -p 1 -L debian-shim -l \\EFI\\debian\\shimx64.efi
```

<br>
<br>


Update initrd image
```bash
update-initramfs -u
```
<br>
<br>

Exit chroot
```bash
exit
```

Unmount `/mnt`
```bash
umount -R /mnt
```
Reboot into the new system
```bash
reboot

