This document will step by step walk through how I installed Arch Linux manually. This install uses btrfs, grub, an encrypted root partition, and uses an efi system partition as a boot partition. It also uses a mounted btrfs subvolume to handle swap.
## Pre-installation
I am going to assume a few things before we start the guide.
1. You have already downloaded and created your installation media and know how to boot into it.
	- I personally used [Ventoy](https://wiki.archlinux.org/title/USB_flash_installation_medium#Using_Ventoy) as I had, a Ventoy drive already on hand, but the Arch Wiki guide for creating a boot drive is [here](https://wiki.archlinux.org/title/USB_flash_installation_medium#Using_manual_formatting).
	- You can also just use [Rufus](https://rufus.ie/).
2. You have experience and are comfortable working in a Linux terminal.
3. You are using a 64-bit x64 UEFI system.
4. You have disabled Secure Boot in your UEFI firmware setup utility (your motherboard BIOS).
	- Arch Linux does not support Secure Boot directly out of the box, so you will either have to configure it during the setup process, or after installation. This guide just takes the easy route and disables it for the time being.
5. You have done your research and understand why you're choosing to use btrfs, grub, encryption, and a subvolume for swap as these may not be the options for your situations or needs.

**Note: Any time I put a you see a words between two stars \*like this\* that is a variable name and you can substitute it with whatever is applicable for you.**

With that out of the way, please plug in your boot device, boot into the installer, and lets start.
## Connect to Internet 
- If using Ethernet connection, it should automatically connect
- To connect to wifi:
1. Use `iwctl` to enter into an interactive prompt.
2. Use  `device list` to find list of network devices.
3. `station *name* scan` to find wireless networks.
4. `station *name* get-networks`.
5. `station *name* connect *SSID*`.
6. Enter SSID password.
7. `exit` to exit interactive prompt.
8. `ping ping.archlinux.org` to confirm connection.
## Set date-time/timezone
1. `timedatectl list-timezones` to find timezones.
2. `timedatectl set-timezone America/Chicago.`
	- Set your timezone to whatever is applicable for you.
3. `timedatectl set-ntp true`.
## Partition Disk [^1]
1. Find disks you want to partition with `fdisk -l` or `lsblk.`
	-  It is recommended to remove any unnecessary disks from device before installation and partitioning.
	- The disk you are looking for should be something like `/dev/sdX` if you are using a sata ssd, or `/nvmeXn1` if you are using an NVME drive. I am using an NVME, so that is what the rest of the guide will refer to.
2.  Run `cfdisk /dev/nvmeXn1` to enter into fdisk gui.
	- **Triple check that you re using the correct drive. This process will wipe your drive and any data on that drive will become unrecoverable.**
3.  Select gpt format.
4.  Format 1M of disk to BIOS boot.
	-  **This is a necessary partition when you are using full drive encryption**.[^7]
5. Format 1G of disk to EFI system. [^11]
6. Format rest of drive to Linux filesystem
	-  A swap partition may be created before dedicating the rest of the disk space to the Linux filesystem if wanted. But I will not be making one here and instead using a swap subvolume.
7.  Write the changes to disk.
## Encrypt Root Partition
1. Run `cryptsetup luksFormat /dev/nvmeXn1pY` and set a strong password.
2. Run `cryptsetup luksOpen /dev/nvmeXn1pY *cryptroot*` to unlock encrypted partition.
	-  The name `cryptroot` at the end of this command can be anything. This the the name of your root directory of your unencrypted disk and will be referenced later.
## Formating/Mounting Partitions and btrfs Subvols
1. Run `mkfs.btrfs /dev/mapper/cryptroot` to format your Linux filesystem partition to btrfs.
2. Run `mkfs.fat -F32 /dev/nvmeXn1pY` on your EFI system partition.
	- Do **NOT** create a filesystem on the BIOS boot partition.
3. mount your main partition for installation: `mount /dev/mapper/cryptroot /mnt`.
4. `cd` into the `/mnt` directory with `cd /mnt`.
5. create our subvolumes:  
	**root**: `btrfs subvolume create @`  
	**home**: `btrfs subvolume create @home`
	**swap** `btrfs subvolume create @swap`
	- The subvolume naming scheme with ****@**** is purposeful. Naming them this way is needed for Timeshift to recognize the subvolumes to be able to create snapshots.
6. Go back to root directory with `cd /`.
7. unmount our mnt partition: `umount /mnt`.
8. `cd` into `/mnt`.
9. Create a directory for efi, home, and swap subvols with `mkdir efi home swap`.
	- This may need to be done again after mounting root directory in step 10 as it seems to clear /mnt directory.
10. mount root subvolumes: `mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@ /dev/mapper/cryptroot /mnt`
	- The options in this mount command are used to optimize the speed of btrfs.
11. mount home subvolumes: `mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@home /dev/mapper/cryptroot /mnt/home`
12. mount swap subvolumes: `mount -o noatime,ssd,compress=zstd,space_cache=v2,discard=async,subvol=@swap /dev/mapper/cryptroot /mnt/swap`
13. mount our efi partition with `mount /dev/nvmeXn1pY /mnt/efi`.[^6]
## Create Swapfile [^2] 
1. `cd` into `/mnt/swap`. 
2. Run `btrfs filesystem mkswapfile --size 4g --uuid clear swapfile` to create the base 4G swapfile.
	- The size of the file can be altered if needed. 4G is just a good baseline.
	- If you want to be able to use hibernation you need to have a swap file equal in size to your amount of RAM.
3. Run `swapon swapfile` to activate swapfile.
4. Make sure `/etc/fstab` is modified to include `/swap/swapfile none swap defaults 0 0`.
## Install Base Packages
1. Run `pacstrap -K /mnt base base-devel linux linux-firmware linux-headers vim` to install the basic packages needed to initialize linux and allow the ability to chroot into it.
## Generate fstab and chroot into system
1. Run `genfstab -U -p /mnt >> /mnt/etc/fstab` to generate your fstab and pipe it into your fstab file.
2. Run `arch-chroot /mnt` to connect to the newly created linux environment.
## Local Date-time, Locale, Keymap, and Hostname
1. `ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime` (this is in your system, not on the iso).
	- Set it to your local timezone
2. `hwclock --systohc` to synchronize clock.
3. locale `vim /etc/locale.gen` find and uncomment your locale, write and exit and then run `locale-gen`.
	- In Vim, you can type `/*your locale*` so you don't have to scroll through the entire file to find your locale.
4. `echo "LANG=en_US.UTF-8" >> /etc/locale.conf` for locale.
5. `echo "KEYMAP=us" >> /etc/vconsole.conf` for keyboard (select yours as appropriate).
6. change the hostname `echo "*hostname*" >> /etc/hostname`.
	- \*hostname* can be anything you want it to be. 
7. `vim /etc/hosts` to edit hostfile with ```
   ```
   127.0.0.1    localhost
   ::1          localhost
   127.0.1.1    *hostname*
   ```
## Change Root Password and Create New User
1. Run `passwd` to change the root password.
2. Run `useradd -m -G wheel *username*` to create a user and add them to the wheel group.
	- The wheel group is what Arch Linux uses to give root permissions. [^12]
3. Run `visudo` and uncomment the line with `%wheel ALL=(ALL:ALL) ALL` to give root permissions to all users in wheel group.
	- ***NOTE***: You must use `visudo` and not try to directly modify the sudoers config file. 
## Updating Mirrors and Installing More Packages
1. Install `reflector` with `pacman -Syu reflector`. This will also update the system if needed.
2. Run `reflector --verbose -n 20 --country US -p https --sort rate --save /etc/pacman.d/mirrorlist --latest 200` to update the mirror list with the 20 fastest mirrors within the United States. [^3]
	- Change the country to your home country and modify this command as needed.
3. `vim /etc/pacman.conf` to make a few modifications.
	- Uncomment `[multilib]` and `Include = /etc/pacman.d/mirrorlist` below it to enable `multilib` repo. This is needed to install 32bit graphics drivers. [^4] 
	- Uncomment `Color` and `VerbosePkgLists`, and add `ILoveCandy` in `#Misc options` to make pacman installations look better.
		- All completely unnecessary, but I like it.
4. Install `mesa lib32-mesa vulkan-radeon lib32-vulkan-radeon` for both 64 and 32 bit AMD GPU drivers. [^5]
5. Install additional packages. There are a lot of packages so I will just list a number of them say and what they do. While not all of these are required, some are like `amd-ucode/intel-ucode` so I recommend installing all of these packages.
```
# "git" to install the git vcs
# "btrfs-progs" are user-space utilities for file system management ( needed to harness the potential of btrfs )
# "grub" the bootloader
# "efibootmgr" needed to install grub
# "grub-btrfs" adds btrfs support for the grub bootloader and enables the user to directly boot from snapshots
# "inotify-tools" used by grub btrfsd deamon to automatically spot new snapshots and update grub entries
# "timeshift" a GUI app to easily create,plan and restore snapshots using BTRFS capabilities
# "amd-ucode" microcode updates for the cpu. If you have an intel one use "intel-ucode"
# "vim" my goto editor, if unfamiliar use nano
# "networkmanager" to manage Internet connections both wired and wireless ( it also has an applet package network-manager-applet )
# "pipewire pipewire-alsa pipewire-pulse pipewire-jack" for the new audio framework replacing pulse and jack. 
# "wireplumber" the pipewire session manager.
# "reflector" to manage mirrors for pacman
# "openssh" to use ssh and manage keys
# "man man-pages" for manual pages
# "sudo" to run commands as other users
# "iptables-nft firewalld ipset" for managing network configurations
# "acpid" for managing power and energy settings
```
## Configure mkinitcpio and GRUB
1. `vim /etc/mkinitcpio.conf` to edit the mkinitcpio file. [^8]
	1. Add `btrfs` to the `MODULES` section. 
	2. Add `sd-encrypt` to the `HOOKS` section after `block` and before `filesystems`. [^9]
2. Regenerate initramfs with `mkinitcpio -p linux`
3. You must first modify `/etc/default/grub` before installing GRUB. [^10]
	1. `vim /etc/default/grub`
	2. uncomment `GRUB_ENABLE_CRYPTODISK=y` to allow booting from /boot on a LUKS encrypted partition.
		- ***NOTE***: If your root partition is encrypted, you cannot install grub unless this flag is checked.
	3. If using the `sd-encrypt` hook in `mkinitcpio.conf`, add `GRUB_CMDLINE_LINUX="... rd.luks.name=*device-UUID*=cryptroot root=/dev/mapper/cryptroot"`. 
		- If using `encrypt` hook in `mkinitcpio.conf` add `GRUB_CMDLINE_LINUX="... cryptdevice=UUID=_device-UUID_:cryptroot root=/dev/mapper/cryptroot"`.
		- The `device-UUID` is the UUID of your encrypted partition eg. `/dev/nvmeXn1pY`.
		- You can add the UUID directly to `/etc/default/grub` with `blkid -o value -s UUID /dev/nvmeXn1pY >> /etc/default/grub` if you don't want to manually type it out.
4. Install GRUB with `grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --recheck`.
5. Generate/update GRUB configuration file with `grub-mkconfig -o /boot/grub/grub.cfg`.
## Enable Base Services
1. Enable network manager with `systemctl enable NetworkManager`.
2. Enable ssh with `systemctl enable sshd`.
3. Enable firewall with `systemctl enable firewalld`.
4. ETC.
## Exit and Reboot into Arch Linux
At this point you can exit `chroot` and reboot your system. If everything has gone well, you should be prompted to input your disk encryption password to get into GRUB. Then you should be able to choose Arch Linux and then be prompted once more to enter your disk encryption password. Then you should be able to log in with your user.


## Reference Guides
- [Arch Wiki Installation Guide](https://wiki.archlinux.org/title/Installation_guide#)
- [Uses btrfs and encryption, but it uses /boot as its boot partition instead of /efi. Still very good for most of the setup](https://github.com/radleylewis/arch_installation_guide)
- [Uses btrfs but no encryption. Has a lot of extra info for KDE, gaming, and GPU drivers.](https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae)


[^1]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Preparing_the_disk_6
[^2]: https://wiki.archlinux.org/title/Btrfs#Swap_file 
[^3]: https://man.archlinux.org/man/reflector.1#EXAMPLES
[^4]: https://wiki.archlinux.org/title/Official_repositories#multilib
[^5]: https://wiki.archlinux.org/title/AMDGPU
[^6]: https://wiki.archlinux.org/title/EFI_system_partition#Typical_mount_points
[^7]: https://wiki.archlinux.org/title/GRUB#GUID_Partition_Table_(GPT)_specific_instructions
[^8]: https://wiki.archlinux.org/title/Mkinitcpio
[^9]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio_6
[^10]: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Configuring_GRUB_2
[^11]: https://wiki.archlinux.org/title/EFI_system_partition#Create_the_partition
[^12]: https://wiki.archlinux.org/title/Users_and_groups#Group_list