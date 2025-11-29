# multipart-efi-boot
setup and manage the existence of multiple EFI boot partitions without the need for mdadm raid or complex GRUB setups

# Introduction

I developed multipart-efi-boot to setup and manage the existence of multiple EFI boot partitions without the need for mdadm raid or complex GRUB setups on Debian or derivative distribution.
This scripts implements it for EFI systems using systemd-boot, but the same idea can be applied to rEFInd or other EFI bootloaders as well.
Also to be fully transparent, script is mimicking a functionality Proxmox also has, but it is not related to their proxmox-boot-tool script, it is written from scratch.

This script is devloped for Debian only so I cannot guarantee it will work the same on your distribution.
It is mostly reliant on debian-specific "installkernel" infrastructure aka the /etc/kernel/postinst.d
and /etc/kernel/postrm.d folders of script hooks, but some distros like Gentoo seem to use them too.
Feel free to send PRs with commits or open issues with instructions to add support for other distros

# Background

The default boot sequence with GRUB on Debian for ZFS or btrfs (or any other non-basic ext4/xfs filesystem) is an absolute clowncar, a EFI partition has the bootloader that goes look for kernel in a dedicated boot partition (either ext4 or ZFS/btrfs with less features enabled) because GRUB does cannot have fully up-to-date ZFS and btrfs drivers because it's has much slower development than linux kernel or ZFS project.

The “simpler” solution is to just replicate the same structure over all disks with a mdadm raid1 that “fakes” a single volume for boot and for EFI for the existing distro setup but it’s “solving compexity with more complexity” and I don’t like that.

Since this is basically used only for booting and updated only when a new kernel is installed, they can be handled as separate EFI partitions each with their own bootloader and kernels. This mimicks Proxmox’s default UEFI installation setup, where they have a script copy the kernel and initramfs in multiple EFI partitions.

I’ve never been a fan of GRUB and 99.99% of the users on a modern system that supports EFI don’t need any of its functionality anyway since we can just boot a kernel in the EFI partition.

I like to use more advanced functionality of zfs to restore snapshots on boot, in a similar way to zfsbootmenu but without adding its own complexity and warts to the equation.

Since we are placing the actual distro kernel in the EFI partition there is no need to keep the zfsbootmenu updated with the distro’s zfs nor handle corner cases like the usb initialization that it does before the kexec, which breaks the Pikvm emulated USB keyboard/mouse function and would require special workarounds in zfsbootmenu to disable.

# Install

**WARNING: This is a somewhat dangerous operation because if something goes wrong your system will not be bootable anymore, so be prepared with a live-cd or other recovery tool to reinstall GRUB or whatever you had before.**

to use this script you must have already installed systemd-boot packages, GRUB or any other bootloader should be already removed.

Any current EFI partition should be unmounted and its entry in /etc/fstab should be commented out or deleted.
When you use this script, the new EFI partitions will live most of their life unmounted, and will be mounted only temporarily when it's time to update the bootloader or kernels, then be unmounted again. This is mostly for compatibility reasons because existing scripts expect there to be only one EFI partition.
You will find a list of the new EFI partition UUIDs in the file **/etc/kernel/efiparts** after adding at least one.


Also the partitions you want to be used as EFI partitions must already exist, but the script will take care of formatting and changing the partition type to EFI  partition

Note: while in the examples I show /dev/sdXX, this scripts does support also nvme drives with /dev/nvmexxxxx


if you don't have it already, create the /efi folder in your root filesystem 
this is where the efi partitions will be mounted temporarily to update the bootloader and kernels
```
sudo mkdir /efi
```
copy current kernel cmdline in this file (create it if it does not exist)
```
nano /etc/kernel/cmdline
```
if you have GRUB, the current cmdline is in /etc/default/grub at the GRUB_CMDLINE_LINUX_DEFAULT= variable

you can also see the current cmdline used to boot by the currently running kernel with 
```
cat /proc/cmdline
```

download the multipart-efi-boot script from this repository and put it in /sbin folder


make the script executable
```
sudo chmod +x /sbin/multipart-efi-boot
```
install the script hooks to the /etc/kernel/* folders
```
sudo multipart-efi-boot install-scripts

```
now you can start adding EFI boot partitions to the script so it prepares them and adds them to the list
```
sudo multipart-efi-boot add /dev/sdXX
```

# Other commands available

remove a boot partition from the list (the partition will be erased)
```
multipart-efi-boot remove
```

refresh all installed kernels (needed only when some very manual change was done to kernels or initramfs, otherwise the script hooks should automatically do the update when necessary)

```
multipart-efi-boot refresh
```

remove the script hooks, this is to help the user uninstall the script. This command does not touch the existing EFI partitions so the system will still be bootable but the kernels won't be updated anymore. It's assumed that after uninstalling this script the user will do whatever other manual steps necessary to install GRUB again or rEFInd or whatever other bootloader of their choosing.
```
multipart-efi-boot remove-scripts
```
