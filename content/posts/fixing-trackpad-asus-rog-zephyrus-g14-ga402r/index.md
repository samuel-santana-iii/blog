---
title: "Fixing a Dead Trackpad on the ASUS ROG Zephyrus G14 (GA402R)"
date: 2026-07-30
draft: false
description: "How to fix a non-working trackpad on the AMD-only ASUS ROG Zephyrus G14 GA402R by updating the chipset drivers through AMD Software Adrenalin Edition."
tags: ["asus", "rog-zephyrus", "amd", "trackpad", "windows", "drivers"]
---

If you're here, you're probably fighting with a Zephyrus G14 trackpad that just stopped responding, maybe right out of the box, maybe after a random Windows update. I went through the exact same thing on my ASUS 2022 GA402R, the AMD 6000 series configuration with an AMD CPU and a dedicated AMD GPU, and it turns out the fix has nothing to do with the physical trackpad itself, but instead a software issue.

This fix applies specifically to systems with an AMD CPU, since the driver involved ships as part of the AMD chipset package rather than through Windows Update or the ASUS driver page. Grab a USB or Bluetooth mouse to get through the next few steps, since the trackpad won't respond to anything until the fix is applied.

## Identifying the Issue

Right-click the Start menu and open **Device Manager**. Expand **Human Interface Devices** and look for an **I2C HID Device** with a yellow warning triangle over its icon. That entry is your trackpad, and the warning triangle confirms this is a driver problem, not a hardware failure.

{{< figure src="trackpad-01-i2c-hid-device-broken.webp" caption="I2C HID Device showing a yellow warning triangle in Device Manager." >}}

## Install AMD Software: Adrenalin Edition

The fix lives in **AMD Software: Adrenalin Edition**. Download and install it from [AMD's website](https://www.amd.com/en/support).

Once it's installed, open it up. You'll land on the main screen with a red **Manage Updates** button.

{{< figure src="trackpad-02-amd-adrenalin.webp" caption="AMD Software: Adrenalin Edition main screen with the Manage Updates button." >}}

## Update the Chipset Drivers

Click **Manage Updates**. This opens a window listing the available software and driver updates for your system.

{{< figure src="trackpad-03-amd-chipset-update.webp" caption="Available updates in AMD Software, including the chipset driver." >}}

Find the chipset driver in the list and install the latest version. This is the piece that actually contains the fix.

## Reboot

Once the chipset driver finishes installing, reboot the laptop. When it comes back up, open Device Manager again and check **Human Interface Devices**. The I2C HID Device should now show up clean, no warning triangle, and the trackpad should be responding.

{{< figure src="trackpad-04-i2c-hid-device-working.webp" caption="I2C HID Device working correctly, warning triangle gone." >}}

That's it. The external mouse was only needed to get through the install and reboot; once the chipset driver is in place, the trackpad works on its own from then on.
