# Create btrfs swapfile
Archinstall uses zram by default. I have been experiencing constant freezing with zram, so I decided to create a swapfile instead.

## Sources
- <https://wiki.archlinux.org/title/Zram>
- <https://wiki.archlinux.org/title/Swap>
- <https://wiki.archlinux.org/title/Btrfs#Swap_file>
- <https://askubuntu.com/questions/1400756/how-to-disable-zram-at-boot>
- <https://github.com/archlinux/archinstall/issues/1399>

## Fix the fstab file created by Archinstall
A known archinstall bug causes the fstab file to be created with Windows-style line endings (CRLF). This causes the system to sometimes fail to boot. To fix this, run the following commands:
```
# mv /etc/fstab /root/fstab.broken
# tr -d '\015' < /root/fstab.broken > /etc/fstab
```
## Disable zram:
Since we are using real swap, we don't need zram. To disable zram, run the following commands:
```
# swapoff -a

# rmmod zram
```
> Note: This may cause some applications to crash. If this happens, reboot and skip those commands.
To prevent zram from reappearing, add the kernel parameter `systemd.zram=0`:
- Open `/etc/default/grub`

In the file that opens, find the line starting with GRUB_CMDLINE_DEFAULT_LINUX= and add systemd.zram=0 to the list of options inside the quotes, separated by spaces. For example:
```
GRUB_CMDLINE_DEFAULT_LINUX="... systemd.zram=0 ..."
```
Save the file and exit, then update the grub config:
```
# grub-mkconfig -o /boot/grub/grub.cfg
```
## Create Swapfile
These commands create a BTRFS subvolume named "@swap" under the top-level subvolume "@", and mount it to the /swap directory. Then, it creates a 16GB swapfile in /swap (/swap/swapfile). Finally, it activates the swapfile with a priority of 100 using the "swapon" command.

Find out where your root partition is by running lsblk and locating the partition where / is mounted. Note that with BTRFS, there may be multiple mountpoints for a single partition. Replace {your root partition} below with the actual root partition (for example, /dev/nvme0n1p3):
```
# mkdir /swap

# mkdir /mnt/@

# mount -t btrfs /dev/{your root partition} /mnt/@

# btrfs subvolume create /mnt/@/@swap

# mount -t btrfs -o subvol=/@swap /dev/{your root partition} /swap

# btrfs filesystem mkswapfile --size 16g /swap/swapfile

# swapon -p 100 /swap/swapfile
```
## Automount the swapfile with fstab:
- Open `/etc/fstab`

Add the following lines to the end of the file, replacing {root partition uuid} with the UUID of your root partition (which you can find by running lsblk -f and locating the partition where / is mounted):
```
# Mount subvolume @swap
UUID={root partition uuid} /swap btrfs defaults,nodatacow,subvol=/@swap 0 0

# Mount swapfile
/swap/swapfile none swap defaults,pri=100 0 0
```
Save the file and exit.

## Stop & Disable EarlyOOM (Optional, recommended):
If you have EarlyOOM due to OOM freezing issues, you can disable it now.
```
# systemctl stop earlyoom.service
# systemctl disable earlyoom.service
```
## Reboot
Reboot your system for the changes to take effect.
To check if the swapfile is working, run `swapon` and check if the swapfile /swap/swapfile is listed.
