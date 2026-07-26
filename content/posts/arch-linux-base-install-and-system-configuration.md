---
title: "Arch Linux Install: Base Install & System Configuration"
date: 2026-07-25
draft: false
description: "Install the Arch base system, configure the bootloader, create a user, enable networking, and set up Snapper snapshots."
series: ["Arch Linux Install"]
tags: ["arch-linux-install", "linux", "arch-linux", "systemd-boot", "snapper", "btrfs-snapshots"]
weight: 2
---

This is Part 2 of the [Arch Linux Install]({{< ref "arch-linux-encrypted-desktop.md" >}}) series. With the drive partitioned and encrypted in [Part 1]({{< ref "arch-linux-disk-prep-and-encryption.md" >}}), we'll now install the base system, get it booting, configure the essentials, and set up automatic snapshots with Snapper.

## Why systemd-boot and Snapper?

**systemd-boot over GRUB:** GRUB is the safer default when you need to dual-boot Windows or juggle multiple kernels with automatic OS detection, but this is a single-OS UEFI system, and none of that complexity buys us anything. systemd-boot is a fraction of the size, configures itself from a couple of short text files instead of a generated script, and boots straight from EFI without an intermediate stage. For a dedicated Linux install, it's less to configure and less to break.

**Snapper over Timeshift:** Timeshift can snapshot BTRFS, but it assumes a fairly rigid subvolume layout and has no direct integration with pacman. Snapper was designed specifically for BTRFS, understands the subvolume layout we built in Part 1, and pairs with `snap-pac` to snapshot automatically before and after every package transaction. Rolling back a bad update becomes a single command instead of a full reinstall.

## Assumptions

This post assumes:

* You've completed [Part 1]({{< ref "arch-linux-disk-prep-and-encryption.md" >}}): your drive is partitioned, LUKS-encrypted, formatted with BTRFS, and the subvolumes are mounted at `/mnt`.
* You're still in the Arch Linux installation media environment (not yet chrooted into the new system).
* Your internet connection is still active, since `pacstrap` needs it to download packages.

## 1. Install Arch

With the drive partitioned, encrypted, and mounted, we're ready to install the base Arch system onto it using pacstrap.

1. Run Pacstrap
   * `pacstrap -K /mnt base base-devel linux linux-firmware btrfs-progs cryptsetup networkmanager nano vim git snapper snap-pac intel-ucode`
   * Replace `intel-ucode` with `amd-ucode` if you are using an AMD processor instead of an Intel processor
2. Generate the fstab
   * `genfstab -U /mnt >> /mnt/etc/fstab`
3. Chroot into the new system
   * `arch-chroot /mnt`

## 2. Setting It Up to Boot

The base system is installed, but it can't boot yet. We need to rebuild the initramfs so it knows how to unlock our encrypted drive at boot, then install and configure a bootloader.

1. Configure mkinitcpio
   * `nano /etc/mkinitcpio.conf`
   * Scroll to the hooks line and insert the word `sd-encrypt` after the word `block`
   * Save and exit
2. Generate the initramfs
   * `mkinitcpio -P`
3. Install the Bootloader
   * `bootctl install`
4. Set up loader.conf
   * `nano /boot/loader/loader.conf`
   * Erase everything. It should look like:

     ```
     default arch.conf
     timeout 4
     console-mode max
     editor no
     ```
   * Then save and exit.
5. Create the Arch Boot Entry
   * Our boot entry tells the bootloader exactly which physical partition is encrypted. 
   * `echo "title Arch Linux" > /boot/loader/entries/arch.conf`
   * `echo "linux /vmlinuz-linux" >> /boot/loader/entries/arch.conf`
   * `echo "initrd /initramfs-linux.img" >> /boot/loader/entries/arch.conf`
   * `echo "options rd.luks.name=$(blkid -s UUID -o value /dev/sda2)=cryptroot root=/dev/mapper/cryptroot rootflags=subvol=@ rw" >> /boot/loader/entries/arch.conf`

With that, our Arch install should be bootable.

## 3. Setting Up the PC

From here, it's the usual Linux post-install setup, just done entirely from the CLI.

1. Set Up the Timezone
   * Get the timezone identifier from [here](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)
   * `ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime` # select timezone
   * `hwclock --systohc` # sync time
2. Configure the Locale (System Language)
   * `nano /etc/locale.gen`
   * Scroll down and uncomment the line for the desired language (for US English, that's `en_US.UTF-8 UTF-8`). Save and exit.
   * `locale-gen` # Generate locale
   * `echo "LANG=en_US.UTF-8" > /etc/locale.conf` # Generate locale
3. Set the Hostname
   * `echo "<hostname>" > /etc/hostname`
4. Set the Root Password
   * `passwd`
5. Reboot the PC
   * `exit` # exit chroot
   * `umount -R /mnt` # Unmount cryptroot partition
   * `reboot` # restarts the PC

Let's remove the installation media once the screen turns black. We'll be asked for the decryption password. If it doesn't boot successfully, see the Troubleshooting section below. If it worked, let's move on to setting up our new user!

## 4. Set Up a User

Right now the only account on this system is root. Using root for daily tasks is risky, so let's create a normal user account and grant it sudo access instead.

1. Log in to the root account with username `root` and the password we set
2. Create a User
   * `useradd -m -G wheel -s /bin/bash <your_username>`
     * `/bin/bash` can be replaced with a different shell if preferred
     * `-m` is short for `--create-home` and copies default setup files into the new user's home directory
     * Note that the members of the wheel group have sudo privileges on many Linux distributions
   * `passwd <your_username>`
3. Give users of the Wheel Group Sudo Privileges
   * Arch needs the wheel group to be manually given sudo privileges
   * `EDITOR=nano visudo`
   * Scroll down and delete the `#` in front of `%wheel ALL=(ALL:ALL) ALL`
   * Save and exit nano

## 5. Turn on Networking

We installed `NetworkManager` earlier, but we need to enable it.

```bash
systemctl enable NetworkManager

# Ensure Arch Installation Media doesn't leave its broken DNS symlink
rm /etc/resolv.conf
```

## 6. Initialize Snapper

Now that the whole system has been set up, our user has been created, and the internet is enabled, it's time to set up snapshots with Snapper.

```bash
# Unmount our snapshots folder temporarily
umount /.snapshots

# Remove the empty directory
rm -r /.snapshots

# Let Snapper create its default config (which also creates a nested .snapshots subvolume)
snapper -c root create-config /

# Delete the nested subvolume Snapper just made
btrfs subvolume delete /.snapshots

# Recreate the empty directory
mkdir /.snapshots

# Mount our custom flat subvolume back into place
mount -a

# Verify the subvolume was mounted.
# If the following command returns a line starting with `/dev/mapper/cryptroot on /.snapshots` then the mount was successful
mount | grep .snapshots
```

## Troubleshooting

### Initial Boot Issues

If you try to boot into Arch and the login screen doesn't appear, there may have been a configuration error such as a typo.

To get back in, boot into the Arch installation media.

```bash
# Unlock the LUKS partition
cryptsetup open /dev/sda2 cryptroot

# Mount the root subvolume
mount -o noatime,compress=zstd,space_cache=v2,subvol=@ /dev/mapper/cryptroot /mnt

# Mount the boot subvolume
mount /dev/sda1 /mnt/boot

# Enter the Arch OS
arch-chroot /mnt
```

Now that it's mounted, you can check for any configuration issues.

Some items to check are:

1. `/boot/loader/loader.conf`
2. `/boot/loader/entries/arch.conf`
3. The hooks line in `/etc/mkinitcpio.conf`
   * Run `mkinitcpio -P` after making any changes in this file

### Networking Issues

If you can't connect once you're booted into the new install, revisit [Turn on Networking](#5-turn-on-networking) above and make sure both steps completed without error.

If `systemctl enable NetworkManager` silently failed, check capitalization. `systemctl` is case-sensitive, and the service name is `NetworkManager`, not `networkmanager`.

If NetworkManager is running but DNS still doesn't resolve, you likely have a stale symlink left over from the installation media. Run `sudo rm /etc/resolv.conf && sudo systemctl restart NetworkManager` to clear it.

## Conclusion

The base system is installed and bootable, your user account is set up with sudo access, networking is enabled, and Snapper is ready to track changes automatically. In [Part 3]({{< ref "arch-linux-package-management-and-core-services.md" >}}), we'll expand pacman with multilib and the AUR, and bring up audio, bluetooth, printing, and graphics drivers.
