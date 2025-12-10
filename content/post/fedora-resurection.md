---
title: "Fedora Resurection"
description: "How to rescue Fedora Workstation from kernel panic with chroot"
date: 2025-12-10
draft: false
slug: "fedora-kernel-panic-rescue-chroot"
cover:
  image: "https://us1.discourse-cdn.com/fedoraproject/optimized/3X/9/0/904469c0912fe84c998ee91985f6ffedcb8fce44_2_690x388.jpeg"
categories: ["Tech"]
tags: ["fedora", "linux", "terminal", "eng"]
---

This should have happened to me someday, and it finally did. I got `Kernel panic` on my Fedora 43 workstation.
Apparently, a software update went south, and the PC was turned off before it properly finished.

[The general advice](https://www.reddit.com/r/Fedora/comments/1k36l57/kernel_panic_after_update/) would be to boot from a previous kernel and regenerate everything. Unfortunately, this wasn't an option for me, because I couldn't boot into older kernels as well. Luckily, it's easy to overcome this whole situation with a LiveUSB stick and a couple of terminal commands.

This article is mostly a reminder to my future self on what to do, if it happens again, so I don't need to sneak peek the commands on multiple web pages.

## Creating a LiveUSB Stick

My another computer is my work MacBook, so the steps below are relevant to MacOS.

1. Download the latest ISO from [the official website](https://www.fedoraproject.org/). It doesn't really matter, which edition, since we only need a terminal.
2. Convert the ISO into DMG format that MacOS understands:
   ```bash
   # You can use any output path instead of ~/Downloads, you don't need to provide the .dmg extension
   hdiutil convert </path/to/Fedora.iso> -format UDRW -o ~/Downloads/Fedora
   ```
3. Attach a USB stick and check its device identifier (the external one). You may want to format the disk into the ExtFAT filesystem beforehand
   ```bash
   diskutil list
   /dev/disk0 (internal, physical):
   ...
   /dev/disk3 (synthesized):
   ...
   /dev/disk4 (external, physical):
   ...
   ```
4. Unmount the disk:
   ```bash
   diskutil unmountDisk /dev/disk4
   Unmount of all volumes on disk4 was successful
    ```
5. Burn the newly converted `.dmg` image on the USB stick with `dd` (you need `sudo` here):
   ```bash
   sudo dd if=/path/to/Fedora.dmg of=/dev/disk4 bs=1m
   ```
   This takes some time, and by default, `dd` doesn't produce any output until it's done. If you're using the GNU version of `dd`, you can add `status=progress` to be sure that `dd` is doing something.

Now, you should have a LiveUSB stick to boot into your poor PC!

## Booting from a LiveUSB

This part is pretty straightforward:
1. Attach the LiveUSB to your PR.
2. Enter the BIOS (UEFI) settings on boot (`DEL` on my MSI motherboard).
3. Select USB as a primary boot device.
4. Reboot into Live Fedora USB.
5. You can skip whatever messages it throws into you, you only need the terminal.
## Chrooting Into the Old System

Here's where the ~~magic~~ main thing happens. In the Terminal

1. Check where the existing installation is located:
   ```bash
   lsblk
   NAME                MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
   zram0               251:0    0     8G  0 disk [SWAP]
   nvme0n1             259:0    0   1.8T  0 disk
   ├─nvme0n1p1         259:1    0 953.7M  0 part /boot/efi
   ├─nvme0n1p2         259:2    0 953.7M  0 part /boot
   ├─nvme0n1p3         259:3    0   7.5G  0 part [SWAP]
   └─nvme0n1p4         259:4    0   1.8T  0 part
   └─vg_root-lv_root 252:0    0   1.8T  0 lvm  /
   ```
   Here, we need both `/` and `/boot` partitions.
2. Create a mount point:
   ```bash
   sudo mkdir /mnt/fedora
   ```
3. Mount the partitions:
   ```bash
   sudo mount /dev/vg_root/lv_root /mnt/fedora
   sudo mount /dev/nvme0n1p2 /mnt/fedora/boot
   sudo mount /dev/nvme0n1p1 /mnt/fedora/boot/efi
   ```
4. We also need to bind mount virtual filesystems that `dnf` will use:
   ```bash
   sudo mount --bind /dev /mnt/fedora/dev
   sudo mount --bind /proc /mnt/fedora/proc
   sudo mount --bind /run /mnt/fedora/run
   ```
5. Check the Internet connection with `ping` or whatever. It may be that the DNS or other settings are also required.
## Reinstalling Things

I would clean the `dnf` cache first, although it's not really necessary:
```bash
dnf clean all
```

The trickiest part here is to know, what to reinstall. At first, I've intuitively reinstaled the kernel, but it didn't help. Eventually, I've managed to boot after reinstalling the `mesa*` packages:
```
sudo dnf reinstall kernel* mesa*
```

If that doesn't work, you can also try a nuclear option as described [here](https://discussion.fedoraproject.org/t/reboot-stuck-in-failed-to-allocate-manger-object/162763/5):
```bash
sudo dnf reinstall $(rpm -qa --qf "%{NAME}\n") --skip-unavailable
```

## Taming SELinux

When working with `chroot`, you need to perform system relabeling before you reboot. Here's [the official guide](https://docs.fedoraproject.org/en-US/quick-docs/selinux-changing-states-and-modes/). Otherwise, SELinux will block the booting process. If you forgot to do it right away, it's not a big deal, though. It only means that you will need to boot from a LiveUSB once again and mount the old system again.
1. Put SELinux into the `permissive` mode by modifying `/etc/selinux/config` (assuming you're still in `chroot`). Set `SELINUX=permissive` instead of `SELINUX=enforcing`.
2. Create an autorelabeling file (assuming still in `chroot`):
   ```bash
   touch /.autorelabel
   ```
3. Now, you can reboot. It will take SELinux some time to relabel everything and then PC will reboot again.

## Broken Packages

Now, you hopefully can log in into your system! Unfortunately, this is not the end. If it was a broken software upgrade, some packages may have broken dependencies or `dnf` may think that they come from unknown sources.

You can check if there are any problems by simply running `dnf update` and seeing if it succeeds. Here's an example of errors I've encountered on update:

```
Transaction failed: Rpm transaction failed.
  - file /usr/share/doc/gnutls/AUTHORS from install of gnutls-3.8.11-5.fc43.i686 conflicts with file from package gnutls-3.8.10-3.fc43.x86_64
  - file /usr/share/doc/gnutls/NEWS from install of gnutls-3.8.11-5.fc43.i686 conflicts with file from package gnutls-3.8.10-3.fc43.x86_64
  - file /usr/share/doc/libxcrypt/NEWS from install of libxcrypt-4.5.2-1.fc43.i686 conflicts with file from package libxcrypt-4.4.38-10.fc43.x86_64
```

To check what's going on with each package, you can run a command like this:
```bash
# Example for libxcrypt
sudo dnf -C list libxcrypt --showduplicates
Updating and loading repositories:
Repositories loaded.
Installed packages
libxcrypt.i686   4.4.38-10.fc43 <unknown>
libxcrypt.x86_64 4.4.38-10.fc43 <unknown>
libxcrypt.x86_64 4.5.2-1.fc43   <unknown>

Available packages
libxcrypt.i686   4.4.38-8.fc43  fedora
libxcrypt.x86_64 4.4.38-8.fc43  fedora
libxcrypt.i686   4.5.2-1.fc43   updates
libxcrypt.x86_64 4.5.2-1.fc43   updates
```

Those `<unknown>` sources are no good. There's probably a smart way to fix it, but I simply reinstalled all the problematic packages.

```bash
sudo dnf reinstall libxcrypt*
```

Once I reinstalled all these problematic packages one by one, my system went back to normal!

---

Future me, I hope you will never need this article, but if you do, I hope it's useful! If anything changes in this process, make sure to update it!

Cheers!

## References
1. [Kernel panic after update (Reddit)](https://www.reddit.com/r/Fedora/comments/1k36l57/kernel_panic_after_update/)
2. [How to Create a Bootable Linux Live USB on Your Mac](https://www.howtogeek.com/741125/how-to-create-a-bootable-linux-live-usb-on-your-mac/)
3. [Reviving Your Fedora System: A Comprehensive Guide to Reinstalling Base Packages from a Live USB](https://itsfoss.gitlab.io/blog/reinstalling-all-base-fedora-packages-from-live-usb-or-distro-repair/)
4. [My Fedora workstation doesn’t boot (Fedora Forum)](https://discussion.fedoraproject.org/t/my-fedora-workstation-doesnt-boot/148945)
5. [Troubleshooting Fedora: Kernel Panic Recovery and Boot Issues after Corrupted Kernel (Fedora Forum)](https://discussion.fedoraproject.org/t/troubleshooting-fedora-kernel-panic-recovery-and-boot-issues-after-corrupted-kernel/128643)
6. [Reboot stuck in “Failed to allocate manger object” (Fedora Forum)](https://discussion.fedoraproject.org/t/reboot-stuck-in-failed-to-allocate-manger-object/162763)
7. [Changing SELinux States and Modes](https://docs.fedoraproject.org/en-US/quick-docs/selinux-changing-states-and-modes/)
8. [Help with conflicting libstdc++ packages (Reddit)](https://www.reddit.com/r/Fedora/comments/1iaxfol/help_with_conflicting_libstdc_packages/)
9. [Front image source](https://discussion.fedoraproject.org/t/kernel-panic/144772)
