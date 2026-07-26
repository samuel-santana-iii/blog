---
title: "Building an Encrypted Arch Linux Setup"
date: 2026-07-25
draft: false
description: "An overview of this 4-part series on installing Arch Linux with full-disk encryption, BTRFS snapshots, and a KDE Plasma desktop."
series: ["Arch Linux Install"]
tags: ["arch-linux-install", "linux", "arch-linux", "btrfs", "luks", "kde-plasma"]
weight: 0
---

Welcome! This series walks through installing Arch Linux from scratch with full-disk encryption, BTRFS subvolumes with automatic snapshots, and a complete KDE Plasma desktop, split into four parts you can follow start to finish or jump into individually.

## What You'll Build

By the end of this series, you'll have a fully encrypted Arch Linux desktop: LUKS2 full-disk encryption, BTRFS subvolumes with automatic Snapper snapshots, a properly configured package manager with AUR support, working audio, bluetooth, printing, and graphics, and a KDE Plasma desktop on top of it all.

## Before You Start

This series assumes:

* A UEFI system (not legacy BIOS).
* Basic comfort with the command line. This isn't a Linux-from-scratch tutorial, but you'll be typing a lot of commands.
* A target drive you're okay wiping completely.
* The Arch Linux installation media, already booted.

## The Series

1. [Arch Linux Install: Disk Prep & Encryption]({{< ref "arch-linux-disk-prep-and-encryption.md" >}}) — Partition the drive, set up LUKS2 encryption, and lay out BTRFS subvolumes.
2. [Arch Linux Install: Base Install & System Configuration]({{< ref "arch-linux-base-install-and-system-configuration.md" >}}) — Install the base system, configure the bootloader, create a user, enable networking, and set up Snapper.
3. [Arch Linux Install: Package Management & Core Services]({{< ref "arch-linux-package-management-and-core-services.md" >}}) — Expand pacman with multilib and the AUR, and bring up audio, bluetooth, printing, and graphics drivers.
4. [Arch Linux Install: Desktop Environment (KDE Plasma)]({{< ref "arch-linux-desktop-environment-kde-plasma.md" >}}) — Install and configure the KDE Plasma desktop.

Each post lists its own assumptions at the top, so if you've already got a working Arch install, feel free to jump straight to whichever part you need.
