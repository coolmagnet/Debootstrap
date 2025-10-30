# INSTALLING DEBIAN -- THE ARCH LINUX WAY (VIA DEBOOTSTRAP)
Instructions on how to install Debian using debootstrap (LUKS)<br><br>
<b>Disclaimer:  This technical guide is provided without warranty of any kind. The user assumes all responsibility and risk for its use. The provider shall not be liable for any damages arising from the use of this guide.</b><br>
<br>
Installing Debian via debootstrap gives you better control over disk partitoning, package installation, and system configuration.<br>
Original work by:  https://gist.github.com/varqox/42e213b6b2dde2b636ef<br>
Added:  Debian Live CD, LUKS, LVM, Timesync, Additional Steps<br>
<br>
Get the latest Debian Live CD (https://www.debian.org/CD/live/)<br>
<br>
<b>[Optional]</b>
If you want to perform the Debian installation remotely.<br>
Install openssh-server, add new user to sudo group, get the IP address then ssh to Live CD.<br>
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
Install debootstrap from the Live CD<br>
```bash
apt update && apt install debootstrap
```
OR<br><br>
Get the latest debootstrap:  http://ftp.debian.org/debian/pool/main/d/debootstrap/<br>
```bash
curl --remote-name http://ftp.debian.org/debian/pool/main/d/debootstrap/debootstrap_1.0.141_all.deb
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
debootstrap --arch amd64 bullseye /mnt https://deb.debian.org/debian
debootstrap --arch amd64 bookworm /mnt https://deb.debian.org/debian
debootstrap --arch amd64 trixie /mnt https://deb.debian.org/debian
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
<b>[Optional]</b>
Syncing the date and time before chroot ensure 'apt update' registers the correct date and time for your location.<br>
Note: If you perform this step you will still need to install the 'systemd-timesyncd' again after you choose your timezone.<br><br>

Sync Date and Time
```bash
apt install systemd-timesyncd
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

<b>[LUKS]</b> Fill <code>/etc/crypttab</code><br>
Note: The UUID is not the LUKS container but the device that holds the LUKS container.<br>
Example:  The UUID is <code>aae291d8-2556-4f45-bc7a-b74349f7bd27</code>, not <code>cd4bd571-2d92-49f6-b2ae-a67a5a30af85</code>
<pre>sda                                                                                            
├─sda1
│    ext4   1.0   OS                       0cba0188-d0c9-4d49-af7b-bd3ec7fad992    8.9G     7% /
└─sda2
     crypto 2                              aae291d8-2556-4f45-bc7a-b74349f7bd27                
  └─HOME
     ext4   1.0   HOME                     cd4bd571-2d92-49f6-b2ae-a67a5a30af85
</pre>
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




<b>[Optional]</b>
If you are planning to Dual Boot and make this Debian Installation a secondary Operating System:
<ul>
  <li><code>apt install grub2</code></li>
  <li><code>grub-mkconfig -o /boot/grub/grub.cfg</code></li>
  <li><code>update-intramfs -u</code></li>
  <li>Do NOT run <code>grub-install</code></li>
</ul>
<code>update-intramfs -u</code> uses <code>/boot/grub/grub.cfg</code> to build the initial ramdisk.

Here is a sample for how to create a custom grub entry <code>/etc/grub.d/40_custom</code> for a secondary Operating System.  Always best to use the UUID instead of the device name.  You can also use <code>os-prober</code> to detect other Windows or Linux Operating System.  File <code>/etc/grub.d/40_custom</code> or <code>os-prober</code> should be performed on your primary Operating System that has Grub fully installed.
```bash
#### Debian custom grub entry (LUKS) ####

root@debian:~# lsblk -f
NAME           FSTYPE      FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                        
├─sda1         ext2        1.0   boot  55b99e30-8083-41ac-bfa7-7bdb655c3f89  925.9M     8% /boot
└─sda2         crypto_LUKS 2           ca44c280-b454-4715-8d69-6aedf7b01ca6                
  └─sda2_crypt ext4        1.0         adafb379-e6ad-45fe-b337-4e24d3774a3d   34.6G     8% /
sdb                                                                                        
├─sdb1         ext4        1.0         884a0c50-7ff6-4630-a1fc-e8c875a85e2e                
├─sdb2         ext2        1.0   boot  b82a12e7-66bc-4213-baf1-8580250485fb    433M    14% /mnt/boot
└─sdb3         crypto_LUKS 2           dd6fc6c5-18f4-49dd-a8ce-831cd72bc0d1                
  └─sdb3_crypt ext4        1.0         5290f1c7-7da3-44d4-9569-20cd33e05fe6   14.8G    17% /mnt

root@debian:~# cat > /etc/fstab << HEREDOC
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
UUID=b82a12e7-66bc-4213-baf1-8580250485fb /boot           ext4    defaults        0       2
UUID=5290f1c7-7da3-44d4-9569-20cd33e05fe6 /               ext4    errors=remount-ro 0       1
HEREDOC

root@debian:~# echo "debian_crypt UUID=dd6fc6c5-18f4-49dd-a8ce-831cd72bc0d1 none luks,discard" >> /etc/crypttab

root@debian:~# cat /etc/grub.d/40_custom
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'Debian GNU/Linux 13 (trixie) (on /dev/sdb3)' --class debian --class gnu-linux --class gnu --class os $menuentry_id_option 'osprober-gnulinux-simple-6e3bedee-83f5-4f8a-8d39-4216e92c120a' {
	insmod part_msdos
	insmod ext2
	set root='hd1,msdos2'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=hd1,msdos2 --hint-efi=hd1,msdos2 --hint-baremetal=ahci1,msdos2  b82a12e7-66bc-4213-baf1-8580250485fb
	else
	  search --no-floppy --fs-uuid --set=root b82a12e7-66bc-4213-baf1-8580250485fb
	fi
#	linux /vmlinuz-6.12.43+deb13-amd64 root=/dev/mapper/sdb3_crypt ro
	linux /vmlinuz-6.12.43+deb13-amd64 root=UUID=5290f1c7-7da3-44d4-9569-20cd33e05fe6
#	linux /vmlinuz-6.12.43+deb13-amd64 root=UUID=5290f1c7-7da3-44d4-9569-20cd33e05fe6 ro preempt=full mitigations=off nosimplefb=1 net.ifnames=0
	initrd /initrd.img-6.12.43+deb13-amd64
}


#### openSUSE custom grub entry ####

suse:~ # lsblk -f
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                           
├─sda1                                                                        
├─sda2 ext4   1.0         4b7ccc49-d026-47c6-8ff3-7936d4f0c7de   31.9G   15% /
└─sda3 swap   1           0b6aed17-430f-4fff-80c8-c263db929c3d                [SWAP]
sdb                                                                           
├─sdb1 ext4   1.0         3abe6454-cf31-43f3-b8a4-216e52b36896   433M    14% /mnt/boot             
└─sdb2 xfs                1620c02a-b7b2-4e5c-afd6-e725a55d66aa   14.8G   17% /mnt        

suse:~ # cat /etc/grub.d/40_custom
#!/bin/sh
exec tail -n +3 $0
# This file provides an easy way to add custom menu entries.  Simply type the
# menu entries you want to add after this comment.  Be careful not to change
# the 'exec tail' line above.
menuentry 'openSUSE Leap 15.6 (on /dev/sdb2)' --class opensuse --class gnu-linux --class gnu --class os $menuentry_id_option 'osprober-gnulinux-simple-3abe6454-cf31-43f3-b8a4-216e52b36896' {
        insmod part_msdos
        insmod ext2
        set root='hd1,msdos2'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd1,msdos2 --hint-efi=hd1,msdos2 --hint-baremetal=ahci1,msdos2  3abe6454-cf31-43f3-b8a4-216e52b36896
        else
          search --no-floppy --fs-uuid --set=root 3abe6454-cf31-43f3-b8a4-216e52b36896
        fi
        linux /boot/vmlinuz-6.4.0-150600.21-default root=UUID=1620c02a-b7b2-4e5c-afd6-e725a55d66aa splash=silent preempt=full nosimplefb=1
#       linux /boot/vmlinuz-6.4.0-150600.21-default root=UUID=1620c02a-b7b2-4e5c-afd6-e725a55d66aa splash=silent preempt=full nosimplefb=1 mitigations=off
#       linux /boot/vmlinuz-6.4.0-150600.21-default root=UUID=1620c02a-b7b2-4e5c-afd6-e725a55d66aa splash=silent preempt=full quiet security=apparmor mitigations=off
        initrd /boot/initrd-6.4.0-150600.21-default
}
```


<br>


Install Grub boot loader<br>
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

<b>[Optional]</b> If you intend to use `sudo`:
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

