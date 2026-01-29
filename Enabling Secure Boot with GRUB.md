 Just a quick guide on enabling secure boot on your Arch Linux install with a GRUB bootloader. This guide used sbctl to create the keys and is done after Arch has already been installed. You can also do this during the install process, but I wouldn't recommend it as it requires some reboots. This guide also does not deal with the enabling of a TPM, only secure boot.
 
1. Install/reinstall GRUB with these extra flags `grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB --modules="tpm" --disable-shim-lock`. [^1]
	- If reinstalling the bootloader, make sure to use the same `bootloader-id` that you used to do your initial install.
2. Update GRUB config file with `grub-mkconfig -o /boot/grub/grub.cfg`. 
	- Not sure if this is necessary if already done before without changing GRUB config file, but seems like a good idea after reinstalling GRUB.
3. Install `sbctl` with `pacman -S sbctl`[^2]
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


[^1]: https://wiki.archlinux.org/title/GRUB#Secure_Boot_support 
[^2]: https://wiki.archlinux.org/title/Unified_Extensible_Firmware_Interface/Secure_Boot#Assisted_process_with_sbctl