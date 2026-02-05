This is mostly gonna be a brain dump of all the little things that I did or ran into after installing Arch. Includes some guides on how to do various things like installing enabling secure boot.

## Connect to Network
- Make sure NetworkManager is installed and probably install iwd before rebooting into Arch. 
- You can connect with NetowrkManager with these steps
	1. Find network interface with `nmcli device`
	2. Look for networks with `nmcli device wifi list`
	3. Connect to network with `nmcli device wifi connect *SSID* password *password*`  

## Installing KDE Plasma
- While you can install the `plasma-desktop` package along with the `sddm` display manager, it will be very barebones.
- A easier option is to install the `plasma-meta` meta package or to install the `plasma` package group.
- 

## Computer Waking Sleep Immediately
- There is a bug with the Gigabyte X670 GAMING X AX V2 motherboard I use that prevents sleep.[^1] 
- To fix this add `acpi_osi=\"!Windows 2015\"` to `GRUB_CMDLINE_LINUX_DEFAULT`, rerun `grub-mkconfig -o /boot/grub/grub.cfg`, and reboot.

## Pacman
- important `pacman` variables include
	- `-S` for sync. Used in all installations
	- `-R` for remove
	- `-y` for refreshing package database
	- `-u` for updating
	- `-s` for searching
	- `-Q` for querying
	- `--needed` only doesn't reinstall package if it doesn't need to be updated
- So these are the most used combos of variables
	- `pacman -Syu` to update full system
	- `pacman -Syy` to refresh local package database
	- `pacman -Qdt` shows orphaned packages
	- `pacman -Ss *package name*` to search for a package if you don't know it's proper name
	- `pacman -Rs` to remove both the package and any dependencies which are not required by any other installed package
- You need to clear the pacman cache every so often as stated [here](https://wiki.archlinux.org/title/Pacman#Cleaning_the_package_cache)

## Arch User Repository (AUR)
- If a package is not available in the official Arch repositories, then you will have to use the AUR
- There are 2 ways to install from the AUR, manually downloading and building the package or using the `paru` command
	- You must install the `paru` package from the AUR manually before you can use it.
	- There are other AUR package managers like `yay` but paru seems to be the fastest because it's written in rust
- To install an AUR package manually
	1. Find package at `aur.archlinux.org`
	2. Copy the `git clone URL`
	3. Download in terminal with `git clone *package URL*`
	4. `cd` into extracted file
	5. run `makepkg -si` to make the package
	6. You  should now have a `.zst` file
	7. Install package with `pacman -U *.zst file*`
- Download `paru` with `git clone https://aur.archlinux.org/yay.git` and install it with steps above
	- You can probably download most packages with git to skip going to the website
- To use `paru` just use the command `paru * AUR package name` and it will search for that package.
	- You may need to select between a number of possible packages it finds
	- You can also abort the installation with `n` 
	- Not a bad idea to allow it to automatically remove any `makepkg` dependencies after it's done installing
- If you use the `paru -Syu` (or just `paru` as it's an alias for `paru -Syu`) not only will it update your AUR packages, but it will also update your `pacman` packages. So this command is preferred if you have AUR packages installed
- Paru uses the same variable flags as `pacman`
	- `paru -Sua` -- Upgrade AUR packages.
	- `paru -Qua` -- Print available AUR updates.
## Random 
- `neofetch` has been deprecated and `fastfetch` is now the package to install.
## Avoiding having to enter the encryption passphrase twice [^2]
 1. Generate key with `dd bs=512 count=4 if=/dev/random iflag=fullblock | install -m 0600 /dev/stdin /etc/cryptsetup-keys.d/root.key`
 2. Add key to luks with `cryptsetup luksAddKey /dev/nvmeXn1pY /etc/cryptsetup-keys.d/root.key`
 3. Add `FILES=(/etc/cryptsetup-keys.d/root.key)` to `/etc/mkinitcpio.cont/`and regenerate initramfs with `mkinicpio -p linux`
 4. If using `sd-encrypt` hook, add kernel parameter `GRUB_CMDLINE_LINUX="... rd.luks.key=/etc/cryptsetup-keys.d/root.key"`
	- If using `encrypt` hook, add kernel parameter `GRUB_CMDLINE_LINUX="... cryptkey=rootfs:/etc/cryptsetup-keys.d/root.key"`
5. Reboot

## Enabling Secure Boot with GRUB
1. Install/reinstall GRUB with these extra flags `grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --modules="tpm" --disable-shim-lock`. [^3]
2. Update GRUB config file with `grub-mkconfig -o /boot/grub/grub.cfg`. 
	- Not sure if this is necessary if already done before without changing GRUB config file, but seems like a good idea after reinstalling GRUB.
3. Install `sbctl` with `pacman -S sbctl`[^4]
4. Use `sbctl status` and check and see if secure boot is disabled, that setup mode is enabled, and that sbctl is not installed.
	- If this is not the case then you will need to enter your motherboards UEFI and disable secure boot and enable setup mode. Also make sure to clear any currently stored secure boot keys and disable factory key provisioning.
	- This process varies by motherboard manufacture.
5. Create your keys with `sbctl create-keys`.
6. Enroll keys with `sbctl enroll-keys -m`.
	- The `-m` enrolls keys with Microsoft keys. Not doing so can cause issues with some firmware so it is recommended that you do this.
7. Check status again with `sbctl status`. `sbctl` should be installed now, but secure boot will not work until the boot files have been signed with the keys you just created.
8. Check what files need to be signed with `sbctl verify`
9. Sign all unsigned files with `sbctl sign -s /path/to/file`
	- Usually the kernel and the boot loader need to be signed. For example:
		- **Kernel:** `sbctl sign -s /boot/vmlinuz-linux`
		- **Boot loader:** `/efi/EFI/GRUB/grubx64.efi`
10. Reboot system, enter UEFI and re-enable secure boot. If the boot loader and OS load, you should be good. But you can check by running `sbctl status`.
## Get GRUB to Recognize Windows [^5]
1. Edit `/etc/default/grub` and uncomment `GRUB_DISABLE_OS_PROBER=false`. [^6]
2. Find your Windows efi partition by running `fdisk -l`.
3. mount your Windows partition with `mount /dev/nvmeXn1pY /mnt`.
4. regenerate GRUB config with `grub-mkconfig -o /boot/grub/grub.cfg` and your Windows partition should be recognized.
5. Reboot.

[^1]: https://wiki.archlinux.org/title/Power_management/Wakeup_triggers#Instantaneous_wakeup_after_suspending
[^2]: https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#With_a_keyfile_embedded_in_the_initramfs 
[^3]: https://wiki.archlinux.org/title/GRUB#Secure_Boot_support 
[^4]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Assisted_process_with_sbctl
[^5]: https://wiki.archlinux.org/title/GRUB#Configuration
[^6]: https://wiki.archlinux.org/title/GRUB#Detecting_other_operating_systems