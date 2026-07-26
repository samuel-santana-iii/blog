---
title: "Arch Linux Install: Disk Prep & Encryption"
date: 2026-07-25
draft: false
description: "Partition your drive, set up LUKS2 full-disk encryption, and lay out BTRFS subvolumes for a new Arch Linux install."
series: ["Arch Linux Install"]
tags: ["arch-linux-install", "linux", "arch-linux", "encryption", "luks2", "btrfs", "disk-partitioning"]
weight: 1
---

This is Part 1 of the [Arch Linux Install]({{< ref "arch-linux-encrypted-desktop.md" >}}) series. In this post, we'll partition and encrypt the drive with LUKS2, lay out BTRFS subvolumes, and get everything mounted and ready for the base install in [Part 2]({{< ref "arch-linux-base-install-and-system-configuration.md" >}}).

## Why LUKS2 and BTRFS?

Arch's default install guide doesn't set up encryption or BTRFS. I'm using both here, and it's worth explaining why before diving in.

**Full-disk encryption (LUKS2):** If this machine is ever lost or stolen, an unencrypted drive means anyone with physical access can read every file on it just by booting from a USB stick. LUKS2 makes the drive unreadable without the passphrase, even if it's pulled out and connected to another machine directly.

**BTRFS over ext4:** ext4 is the safer, simpler default, and it's a completely reasonable choice. I'm using BTRFS for two features ext4 doesn't have: subvolumes and snapshots. Subvolumes let me split the filesystem into independent, separately-mountable pieces without needing separate partitions for each one. Snapshots, set up with Snapper later in this series, let me roll the entire system back to a known-good state in seconds if an update breaks something, which matters more on a rolling-release distro like Arch than it would on something with a slower release cycle.

**The subvolume layout:** I'm not using one big BTRFS volume. `@home`, `@snapshots`, `@var_log`, and `@pkg` are split out from the root subvolume (`@`) specifically so Snapper's snapshots stay small and meaningful. Logs and the package cache change constantly and aren't worth snapshotting; splitting them out keeps every root snapshot focused on actual system state instead of noise.

## Assumptions

This post assumes:

* You've booted into the Arch Linux installation media on a UEFI system.
* You have a wired internet connection, or you'll connect to Wi-Fi using the optional step below.
* You have a target drive you're okay wiping completely. Every command here destroys existing data on it.

## Connect to Wi-Fi (Optional, Skip if Using Ethernet)

If you're plugged in via Ethernet, skip this step. It works out of the box. Otherwise, connect using `iwctl`, the interactive Wi-Fi tool on the installation media:

```bash
# Enter the interactive iwd prompt
iwctl

# List available Wi-Fi devices
device list

# Scan for networks (replace <device> with your device name, e.g. wlan0)
station <device> scan

# List networks found by the scan
station <device> get-networks

# Connect to your network (you'll be prompted for the password)
station <device> connect <SSID>

# Once connected, exit the iwctl prompt
exit
```

## 1. Test the Internet Connection

Let's confirm we have a working internet connection before going any further.

`ping -c 3 google.com`

## 2. Sync Time

An accurate system clock matters for verifying package signatures later, so let's sync it via NTP before continuing.

`timedatectl set-ntp true`

## 3. Partition the Drive

We're going to create two partitions: a small EFI boot partition and a larger partition that we'll encrypt with LUKS in the next step.

### Wipe the Drive and Open fdisk

Let's identify the drive and wipe any old filesystem signatures so `fdisk` doesn't get confused by leftover data.

```bash
fdisk -l # list drives

# Wipe old signature to prevent errors
wipefs -af /dev/<target_drive>

# Open drive
fdisk /dev/<target_drive>
```

### Create a New Partition Table

Now in fdisk we'll have an interactive menu, where we press a letter and then press Enter.

1. First, press `g`, then press Enter, to queue creating a GPT partition table
2. Press `w`, then press Enter, to perform the queued command. We only queued one at this point, so a new GPT partition table will be created.

Once the commands run, we'll be back at the shell prompt. Let's open fdisk again with `fdisk /dev/<target_drive>`.

### Create the Partitions

Now let's create our partitions.

1. Create Partition 1 (Boot)
   * Press `n` for new partition
   * Press Enter to accept default partition number 1.
   * Press Enter to accept default first sector
   * Type `+1G` and press Enter (sets the size to 1 Gigabyte)
   * Note: If it asks whether we want to remove a vfat signature, press `y`, since it's a remnant from the previous partition table.
   * Press `t` (to change partition type)
   * Type `1` and press Enter to set the type to "EFI System"
2. Create Partition 2 (LUKS)
   * Press `n` for new partition
   * Press Enter to accept default partition number 2.
   * Press Enter to accept default first sector
   * Press Enter to accept default last sector (uses 100% of the remaining disk space)
   * Note that the line says "Created a new partition 2 of type 'Linux filesystem'", so we do not need to change the partition type
3. Save and Exit
   * Press `w` to perform the queued commands to create partition 1 and 2.

We can run the `lsblk` command to verify our new partitions were created.

## 4. Encrypting Our Second Partition with LUKS2

With our two partitions created, we'll format the boot partition and encrypt the second one with LUKS2, so everything except the small boot partition is unreadable without our passphrase, even if the drive is removed and read elsewhere.

1. Format the boot partition with Fat32
   * `mkfs.fat -F 32 /dev/sda1`
2. Encrypt the Main Partition
   * `cryptsetup luksFormat /dev/sda2`
   * For confirmation, type "YES" in all caps and press Enter.
   * It will ask for a passphrase. **Do not forget this password.** If we lose it, our data is gone forever. (Note: The passphrase box will remain empty as we type. Type the passphrase and hit Enter.)

## 5. Opening Our Encrypted Drive

Before we can format or mount anything, we need to unlock the encrypted partition and map it to a device we can work with.

1. Open the encrypted drive
   * `cryptsetup open /dev/sda2 cryptroot`
   * Enter the passphrase to decrypt and open
   * Note: we are giving the name `cryptroot`, which is a standard naming convention
2. Verify the drive is mapped
   * `lsblk`
   * There should now be a `cryptroot` branch underneath our system partition.
   * `cryptroot` is our decrypted virtual drive
   * IMPORTANT: Now that the drive is decrypted and mapped, we will be using `/dev/mapper`.

## 6. Creating Our System Partition Subvolumes

BTRFS lets us split the filesystem into independent subvolumes, which act like separate filesystems sharing the same underlying storage. This is useful for excluding certain directories, like logs or the package cache, from snapshots, so Snapper doesn't back up data that doesn't need it.

1. Format the open LUKS container with BTRFS
   * `mkfs.btrfs /dev/mapper/cryptroot`
2. Mount the Top-Level System
   * `mount /dev/mapper/cryptroot /mnt`
   * This ensures we can create our subvolumes.
3. Create the subvolumes
   * `btrfs subvolume create /mnt/@`
   * `btrfs subvolume create /mnt/@home`
   * `btrfs subvolume create /mnt/@snapshots`
   * `btrfs subvolume create /mnt/@var_log`
   * `btrfs subvolume create /mnt/@pkg # For pacman cache`
4. Unmount the Top-Level
   * `umount /mnt`

## 7. Mounting Our Subvolumes

With the subvolumes created, let's mount them into place so pacstrap has somewhere to install the base system.

1. Mount the Root Subvolume (@)
   * `mount -o noatime,compress=zstd,space_cache=v2,subvol=@ /dev/mapper/cryptroot /mnt`
   * This mounts the `@` subvolume to `/mnt`.
   * We are using the `noatime` flag, which prevents a file's timestamp from updating when it's only read, avoiding unnecessary disk writes.
   * We are using the `compress=zstd` flag. This saves space, and on hard drives it slightly increases read and write speeds.
   * We are using the `space_cache=v2` flag. This flag maintains an optimized map of free space on the volume.
2. Create the mount points
   * `mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman/pkg}`
   * Now that our root subvolume is mounted, we create the folders where the rest of the subvolumes will connect.
3. Mount the subvolumes
   * `mount -o noatime,compress=zstd,space_cache=v2,subvol=@home /dev/mapper/cryptroot /mnt/home`
   * `mount -o noatime,compress=zstd,space_cache=v2,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots`
   * `mount -o noatime,compress=zstd,space_cache=v2,subvol=@var_log /dev/mapper/cryptroot /mnt/var/log`
   * `mount -o noatime,compress=zstd,space_cache=v2,subvol=@pkg /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg`
4. Mount the Boot Partition
   * `mount /dev/sda1 /mnt/boot`
   * This ensures the Linux kernel is installed on our unencrypted boot partition.

## Conclusion

The drive is partitioned, encrypted with LUKS2, laid out into BTRFS subvolumes, and mounted at `/mnt`, ready for the base system. In [Part 2]({{< ref "arch-linux-base-install-and-system-configuration.md" >}}), we'll install Arch itself, get it booting, and set up automatic snapshots with Snapper.
