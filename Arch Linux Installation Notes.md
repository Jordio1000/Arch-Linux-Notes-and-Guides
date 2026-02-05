## Connect to wireless network
- Connect to network using `iwctl` You will enter a interactive prompt.
	- `device list` to find list of network devices
	- `station *name* scan` to find wireless networks
	- `station *name* get-networks`
	- `station *name* connect *SSID*`
	- Enter SSID password
## Format Disks
- `fdisk -l` or `lsblk` to find available disks
- `fdisk *drive name*` to begin formatting process
	-  Can also use `cfdisk *drive name*` to get a GUI for fdisk. Might actually be preferred
- Arch needs minimum 2 partitions to function
	- boot/efi (Min 500Mb, but 1Gb might be preferred)
	- root
	- Swap is optional. Seems to be recommended.
- If using btrfs, just use the basic 2 partitions, maybe a swap. 
- If using ext4, you can to add /home partition if you want.
- **If you are encrypting your drive you need 3 partitions**
	- Linux filesystem
	- EFI
		- recommended to be 1G
	- and **BIOS Boot**
		- Must be 1M and have no filesystem
## SWAP
- Honestly, still don't have a great answer for what is the best. Seems like swapfiles are the way to go for the most part these days
- I'm going with swapfile + zswap. 
- zram is also an option, but seems to be better suited for different tasks
- You must create a swap subvolume, then create a swap directory and then mount the subvolume to the directory before you create your swap file. [reference here](https://bbs.archlinux.org/viewtopic.php?id=289574)
- Created swap file at /mnt/swap/swapfile with command `btrfs filesystem mkswapfile --size 4g --uuid clear /mnt/swap/swapfile`
- activate swap with `swapon /swap/swapfile`
- Modify fstab at /etc/fstab with `/mnt/swap/swapfile none swap defaults 0 0`
## Mounting
- First need to create subvolumes as mounting points. All of this is done in /mnt
	- Create subvolume with `btrfs subvolume create @` for root directory and `btrfs subvolume create @home` for home directory.
		- Naming the subvolumes ***@*** is important as Timeshift uses this as a tag to know what subvolumes to snapshot.
- Created **@swap** subvolume and mounted it at /mnt/swap
- Create directory **/mnt/efi** This is where the bootloader will mount to. 
	- /efi seems to be preferred over /boot at least for btrfs
## Packages
-  Only base, linux, and linux-firmware packagers are required.
- enable `multilib` repository so that you can install 32-bit GPU drivers
## Users and Groups
- `passwd` to set root password
- `useradd -m -G wheel *username*` to create user and add to sudoers group
	- `wheel` is Arch's sudoers group
	- `-m` creates a home directory for the user
	- `-G` is what group you want to add them to
- Open sudoers file with `visudo`
	- You must do this and not edit sudoers file directly. This command protects the file and does some other background sutff
	- uncomment `%wheel ALL=(ALL) ALL`
		- uncomment `%wheel ALL=(ALL) ALL NOPASSWD: ALL` to not have to enter a password after using sudo.
## mkinitcpio
- If the system is encrypted, you need to add `encrypt` into the **HOOKS** section after *block* and before *filesystems*
- Add `btrfs` to the **MODULES** section if using btrfs.
- use `mkinitcpio -p linux` to recreate mkinit
## Grub
-  With and encrypted disk, need to set `GRUB_ENABLE_CRYPTODISK=y` in /etc/default/grub
- You must define your encrypted devices in the `GRUB_CMDLINE_LINUX_DEFAULT` section under /etc/default/grub
	- add `cryptdevice=UUID=*uuid*:main root=/def/mapper/main`
- Add UUID to /etc/default/grub with `blkid -o value -s UUID /dev/nvme0n1pX >> /etc/default/grub`
- `grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB`
- [GRUB install with encryption](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB))




## References 
- [Arch setup with ext4 and lvm(video)](https://www.youtube.com/watch?v=FxeriGuJKTM&t=1746s)
- [Arch setup with btrfs and encryption (video)](https://www.youtube.com/watch?v=Qgg5oNDylG8)
	- [Same person, but in text format](https://github.com/radleylewis/arch_installation_guide)
- [Arch setup with btrfs, no encryption. Includes gaming stuff](https://gist.github.com/mjkstra/96ce7a5689d753e7a6bdd92cdc169bae)
- 