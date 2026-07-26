---
title: "Arch Linux Install: Package Management & Core Services"
date: 2026-07-25
draft: false
description: "Expand pacman with multilib and the AUR, and bring up audio, bluetooth, printing, and graphics drivers."
series: ["Arch Linux Install"]
tags: ["arch-linux-install", "linux", "arch-linux", "pacman", "aur", "pipewire", "bluetooth", "cups"]
weight: 3
---

This is Part 3 of the [Arch Linux Install]({{< ref "arch-linux-encrypted-desktop.md" >}}) series. With a bootable, networked system from [Part 2]({{< ref "arch-linux-base-install-and-system-configuration.md" >}}), we'll now expand pacman with multilib and the AUR, and bring up the core hardware services (audio, bluetooth, printing, and graphics), everything the desktop in [Part 4]({{< ref "arch-linux-desktop-environment-kde-plasma.md" >}}) will need.

## Why PipeWire and yay?

**PipeWire over PulseAudio:** PulseAudio has been the default Linux audio server for over a decade, but it only handles audio. Getting professional audio routing meant running JACK alongside it, often fighting over the same devices. PipeWire replaces both: it speaks PulseAudio's protocol for regular desktop audio and JACK's protocol for pro-audio and screen-sharing pipelines, all from one daemon. That's why we install `pipewire-pulse` instead of `pulseaudio` itself. It's a compatibility layer, not a second audio server.

**yay over other AUR helpers:** paru and trizen are both solid alternatives, and either would work here. I went with yay because it's the most widely documented and actively maintained option, and its flags closely mirror pacman's own, so it doesn't introduce new syntax on top of what pacman already teaches.

## Assumptions

This post assumes:

* You've completed [Part 2]({{< ref "arch-linux-base-install-and-system-configuration.md" >}}): you've rebooted into your new Arch installation and are logged in as your user (not root, not chrooted).
* Your user is in the `wheel` group with sudo access, since every command here uses `sudo`.
* You have an active internet connection via NetworkManager, since pacman and the AUR both need it.

## Enable the Multilib Repository

Open `/etc/pacman.conf` and uncomment the `[multilib]` section. This allows the system to run 32-bit software, which is mandatory if we ever want to use Steam or play games.

`sudo nano /etc/pacman.conf`

Scroll down to the `[multilib]` section and uncomment it by deleting the `#` symbols. Our `[multilib]` section should look like:

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Once that change is made, save and exit the file. Then force pacman to download the newly unlocked 32-bit databases.

`sudo pacman -Sy`

## Install yay, an AUR Helper

The Arch User Repository (AUR) is where all the community packages live. A helper like yay automates downloading and building these packages.

```bash
# Clone the yay repository
git clone https://aur.archlinux.org/yay.git ~/yay

# Navigate into the build folder
cd yay

# Compile and install (do not use sudo)
makepkg -si

# Once installed, cd out of the folder and delete the yay folder
cd ~ && sudo rm -r yay
```

Once installed, we can install anything with the command: `yay -S <package_name>`.

## Core Hardware Daemons

This is where we'll set up the universal system services.

### Audio

PipeWire is the modern standard for Linux audio and handles both sound and screen-sharing flawlessly.

```bash
sudo pacman -S pipewire pipewire-pulse pipewire-alsa wireplumber
```

No `systemctl enable` is needed here. PipeWire starts automatically for the user.

### Bluetooth

`bluez` is the Linux Bluetooth protocol stack, and `bluez-utils` provides the command line and system tray tools to manage it.

```bash
sudo pacman -S bluez bluez-utils
sudo systemctl enable bluetooth
```

### Network Discovery

Avahi will scan our network for driverless printers, and `nss-mdns` allows connecting to printers and devices with `.local` network addresses.

```bash
sudo pacman -S avahi nss-mdns
sudo systemctl enable avahi-daemon
```

Configure the name resolver for Avahi:

```bash
sudo nano /etc/nsswitch.conf
```

Find the line that starts with `hosts:` and add `mdns_minimal [NOTFOUND=return]` right before `resolve` and `dns`. It should look like:

```
hosts: mymachines mdns_minimal [NOTFOUND=return] resolve [!UNAVAIL=return] files myhostname dns
```

Then save and exit.

### Printing

CUPS (Common Unix Printing System) handles print jobs and driver management. Combined with the Avahi setup above, most modern printers will be detected automatically.

```bash
sudo pacman -S cups
sudo systemctl enable cups
```

## Setting up the Graphics Drivers

Without the right drivers, your desktop will either fail to start or fall back to slow software rendering. My system has Intel integrated graphics, so the commands below are Intel-specific. If you're on AMD or NVIDIA, search the Arch Wiki for your GPU vendor's driver installation guide.

```bash
sudo pacman -S mesa vulkan-intel intel-media-driver
```

## Setting up Universal Fonts

Before installing a GUI, we need to install a robust set of universal fonts. This prevents random symbols from rendering as tofu-looking blocks if the system doesn't know how to render them.

We will install Google's Noto font family. The name literally stands for "No Tofu," a nod to the missing-glyph blocks described above.

`sudo pacman -S ttf-dejavu noto-fonts noto-fonts-cjk noto-fonts-emoji`

## Conclusion

We now have set up our system services, graphics, and drivers. In [Part 4]({{< ref "arch-linux-desktop-environment-kde-plasma.md" >}}) we will tackle setting up the KDE Plasma desktop. If you don't want KDE, you can skip [Part 4]({{< ref "arch-linux-desktop-environment-kde-plasma.md" >}}) and install your own display manager and desktop environment.
