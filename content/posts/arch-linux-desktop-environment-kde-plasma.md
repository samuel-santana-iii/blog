---
title: "Arch Linux Install: Desktop Environment (KDE Plasma)"
date: 2026-07-25
draft: false
description: "Install and configure the KDE Plasma desktop environment on top of your Arch Linux install."
series: ["Arch Linux Install"]
tags: ["arch-linux-install", "linux", "arch-linux", "kde-plasma", "sddm", "desktop-environment"]
weight: 4
---

Welcome to Part 4 of the Arch Linux installation guide! In [Part 1]({{< ref "arch-linux-disk-prep-and-encryption.md" >}}) and [Part 2]({{< ref "arch-linux-base-install-and-system-configuration.md" >}}), we built an encrypted, unbreakable file system and got it fully installed and configured. In [Part 3]({{< ref "arch-linux-package-management-and-core-services.md" >}}), we built the invisible engines that drive our audio, network, and graphics.

Now it's time to have a GUI!

We are going to install KDE Plasma, one of the most modern yet fast desktop environments in the Linux world. With the foundation we previously set, this process is going to be incredibly smooth.

## Assumptions

This post assumes:

* You've completed [Part 3]({{< ref "arch-linux-package-management-and-core-services.md" >}}): multilib and yay are set up, and your audio, bluetooth, printing, graphics drivers, and fonts are all installed.
* You're logged into your Arch installation as your user with sudo access.
* You have an active internet connection, since installing the `plasma` package group downloads a substantial amount of data.

## The Login Screen

Before we install the desktop itself, we need a "Display Manager." This is the graphical login screen that asks for our username and password when we turn on the computer.

The official login manager for KDE Plasma is called SDDM (Simple Desktop Display Manager).

Install it with pacman:

```bash
sudo pacman -S sddm
```

Next, we need to tell systemd (our service manager) to launch this login screen automatically every time the computer boots up:

```bash
sudo systemctl enable sddm
```

**Warning**: Do not run `sudo systemctl start sddm` right now! SDDM does not have anything to boot into once we log in.

## Installing the KDE Plasma Desktop

Now for the main event. In Arch Linux, we get to choose how bloated or minimal we want our system to be: `plasma-desktop` installs just the shell and window manager, while the full `plasma` group adds the applets and system settings modules most people expect out of the box, Network Manager, Audio Control, Bluetooth, and System Monitor among them.

We'll go with the full `plasma` group. I'd rather have those pieces working from the start than track down which ones are missing later.

In addition to this plasma package, we can install several KDE apps as well:

* Dolphin (`dolphin`): File browser
* Konsole (`konsole`): Terminal
* Ark (`ark`): The archive manager, for extracting .zip and .tar.gz files
* Kate (`kate`): A lightweight graphical text editor
* Okular (`okular`): The best PDF and document viewer on Linux
* Gwenview (`gwenview`): The default KDE image viewer
* Spectacle (`spectacle`): KDE's screenshot and screen recording tool
* KDE Connect (`kdeconnect`): Links an Android or iOS phone to the computer for clipboard sync, file transfers, and phone notifications.
* Filelight (`filelight`): A gorgeous graphical disk usage tool
* Elisa (`elisa`): A music player that integrates perfectly with the Plasma desktop controls
* Flatpak (`flatpak`): Download and run isolated software packages in a sandbox environment

If you want something else, check whether it's available as a Flatpak instead.

With this in mind, let's start the Plasma install!

```bash
# Install plasma with the essentials
sudo pacman -S plasma dolphin konsole ark spectacle kate gwenview okular elisa flatpak filelight kdeconnect
# Hit enter if it says there are 70 members in group plasma. We are installing everything in this package group.
# Select 'qt-multimedia-ffmpeg' and hit enter if asked for 'qt6-multimedia-backend'
# Select 'pipewire-jack' and hit enter if asked for 'jack' provider to use our PipeWire daemon
# If asked for a 'tessdata' provider, choose a language (e.g. 'tesseract-data-eng') and hit Enter
# Then hit 'y' to proceed with installation

# Reboot the system
reboot
```

We should now be booted into our new SDDM login screen. If you're a gamer who uses Steam and Proton, make sure to use Wayland, not X11. Let's log in, and we'll be presented with our new KDE desktop environment.

## Troubleshooting

### No Terminal Emulator After Login

If you log in and can't find a terminal anywhere in the applications menu, you likely left Konsole (or another terminal emulator) out of the install command above. Without one, there's no obvious way to reach pacman from inside the desktop, but you don't need the desktop for that. A TTY is still there underneath it.

1. Switch to a TTY
   * Press `Ctrl+Alt+F3` (or `F2`, `F4`, etc., SDDM usually runs on `F1`) to drop out of the graphical session
2. Log in
   * Enter your username and password at the prompt
3. Install the missing package
   * `sudo pacman -S konsole`
4. Switch back to the desktop
   * Press `Ctrl+Alt+F1` (or whichever TTY SDDM landed on) to return to the graphical session

Konsole should now show up in the applications menu. The same steps work for any package you notice is missing after the fact. The TTY is a full shell under the same user account, with the same sudo access as the desktop.

## Conclusion

That's it! You now have a fully encrypted Arch Linux desktop running KDE Plasma, built from a blank drive across four posts: full-disk encryption and BTRFS subvolumes in [Part 1]({{< ref "arch-linux-disk-prep-and-encryption.md" >}}), a configured base system with automatic snapshots in [Part 2]({{< ref "arch-linux-base-install-and-system-configuration.md" >}}), a fully capable package manager and hardware stack in [Part 3]({{< ref "arch-linux-package-management-and-core-services.md" >}}), and now a desktop to tie it all together.

From here, the rest is yours to shape: themes, additional apps, dotfiles, whatever your workflow needs. If you get stuck along the way, revisit the [series overview]({{< ref "arch-linux-encrypted-desktop.md" >}}) for the full map of what each part covers.
