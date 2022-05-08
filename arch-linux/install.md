# Forward

This is meant to be a supplimentary guide to the already excellent wiki article found [here](https://wiki.archlinux.org/title/installation_guide)

My experience comes from installing arch on a laptop (TongFang something or other), so most of the docs here will be about how to set that up and the problems that I ran into.

Last qualifier: These instructions could be very out of date.

------


# Partitioning the boot disk

I used `parted` because it seemed like a simple option. `parted` will satisfy most needs.

## EFI Partition

The guide says this, and I agree: Don't delete you're EFI partition. Keep it instead. If it isn't broken, don't fix it.

## Usefule `parted` commands

- `parted --list`
    lists all partitions in all present disks
- `parted /dev/[your install disk here]`
    starts a live session of `parted`.

The following commands can be run inside of a `parted` live session (see previous command)

- `print all`
    list all partitions
- `rm [partition number]`
    remove the given partition
- `mkpart PART-TYPE [FS-TYPE] START END`
    - PART-TYPE - usually primary (for swap partition and root partition)
    - FS-TYPE - depends, but for swap it will be `linux-swap`, `ext4` is what I chose for my root partition
    - START/END - These are in Megabytes/Gigabytes by default, but you can also use percentages (useful if you want to say: until the end of the disk, you just put 100% as END)
- `resizepart NUMBER END`
    - NUMBER the partition number
    - END the new "end" of the partition
    - Note: You really ought to run `resize2fs -M` before making a partition smaller. After you are done resizing the partition (to be bigger or smaller) run `resize2fs` again to fill up the available space
    - Note: In a normal install, this isn't really needed.

## Partitions and their sizes

I ended up making the following partitions:

Unless stated otherwise, the following commands are run inside of a `parted` live session (see previous command)

Also, I would've ordered my partitions differently if I was restarting, so don't just follow this next part blindly

- `efi` partition (1)
    I did not edit this partition at all, but left the preexisting one

- `swap` partition (2)
    - command: `mkpart primary linux-swap [end of efi partition] [end of efi partition + ram size]`
    - I made my swap as large as my available RAM, but am not sure if that's the recommended move

- `root` partition (3)
    - command: `mkpart primary ext4 [end of swap] [100% - 1GB]`
    - ouside of `parted` live session run: `e2label /dev/[your device and partition here] arch_root` (in my case, it was `e2label /dev/nvme0n1p3 arch_root`)
    - Again for documentation's sake, I'm putting this here, this is not how I would've ordered my partitions if I was restarting

- `xbootldr` partition (3)
    - command: `mkpart xboot_ldr fat32 [end of root] 100%`
    - I'll talk about why I made this in a bit
    - Additional config was needed, see [Reccomendation](#reccomendation) section

## What I would've done differently (also, wtf is the `xbootldr` partition?)

I didn't know this at the time, but, my efi partition was too small to fit the kernel, initramfs, initramfs-fallback and the cpu microcode modules. This eventually lead to me not being able to boot after finishing the install, and I had to look for a solution.

The solution I went with is using `XBOOTLDR`. Not sure if it's a program, or just a spec, but the main idea is this: you load up your efi partition, which in turn loads up further modules in an `XBOOTLDR` partition. This allows you to have more space that you usually would.

If I was to reinstall `arch` again, I would've planned to have an `XBOOTLDR` partition from the start, probably as the second partition.

so, I would've done this:

- `efi` partition (1)
    Leave this partition as-is

- `xbootldr` partition (2)
    - command: `mkpart xboot_ldr fat32 [end of efi] [end of efi + 1GB]`
    - Additional config was needed, which will be in the next section

- `swap` partition (3)
    - command: `mkpart primary linux-swap [end of xbootldr partition] [end of xbootldr + ram size]`

- `root` partition (4)
    - command: `mkpart primary ext4 [end of swap] 100%`
    - ouside of `parted` live session run: `e2label /dev/[your device and partition here] arch_root` (in my case, it was `e2label /dev/nvme0n1p3 arch_root`)


### How to make a `XBOOTLDR` partition

If your `efi` partition is small, ~100MB and you need to install nvidia drivers, I would just plan on making this partition from the beginning.

Here are the steps on doing so:

- (in parted live session): `mkpart xbootldr fat32 [end of efi] [end of efi + 1GB]` (see the list of partitions just before this section)
- start a gdisk session (by running `gdisk` after exiting the `parted` live session), and select yoru disk (mine was `/dev/nvme0n1`) and your partitioning scheme (You probably won't need to do this, it should auto select the correct one `GPT` in my case)
- enter `t`
    - enter your partition number (mine was `4`, if you're following my suggestion, it will be `2`, but either way, whatever partition number goes to your `xbootldr` partition)
    - for the hex code you can enter: `EA00`, or you can enter `L` and search for `XBOOTLDR`
    - `gparted` should tell you it changed the partition type guid to this (uppercase/lowercase not important): `bc13c2ff-59e6-4262-a352-b275fd6f7172`
- enter `w` to write changes and exit (`gdisk` doesn't actually save changes until you do that)

Now go follow the guide until you get to mounting your partitions (You'll run `mkfs.fat` for the `XBOOTLDR` partition during the `mkfs` part, in addition to setting up the `root` partition fs)

### Where to mount your partitions when using an `XBOOTLDR` partition

Welcome back

Since you've made the `XBOOTLDR` partition, you're going to want to mount your partitions differently than the installation guide suggests

Here's how you would mount them (again after following the `mkfs` and `mkswap` part of the installation guide)

If your device was `/dev/nvme0n1`

and your partitions were as follows:

1: `efi` partition
2: `xbootldr` partition
3: `swap` partition
4: `root` partition

You would mount your partitions in this manner

1. `mount /dev/nvme0n1p4 /mnt`
1. probably need to run `mkdir /mnt/boot` and `mkdir /mnt/efi`
1. `mount /dev/nvme0n1p1 /mnt/efi`
1. `mount /dev/nvme0n1p2 /mnt/boot`
1. `swapon /dev/nvme0n1p3`

Continue with the installation guide, and come back here when you get to the bootloader section. (note, I do have some tips on what to install with `pacstrap` below as well, so check those out before you come back too)

### Bootloader setup using `XBOOTLDR`

Welcome back.

I'm assuming that you've `arch-chroot`ed into your new install at this point.

For the boot manager I decided to use [`systemd-boot`](https://wiki.archlinux.org/title/Systemd-boot) which only works for `efi` systems wight `GPT` drives (aka, like most computers built after about 2010)

Setup is fairly straight forward, and you can follow the [installation guide for `XBOOTLDR`](https://wiki.archlinux.org/title/Systemd-boot#Installation_using_XBOOTLDR), but I've included the command to install below:

`bootctl --esp-path=/efi --boot-path=/boot install` (assuming that you followed my mounting pattern above)

Now it's time for configuration. There's a lot of stuff in the guide here, but I'll include my own config files as well here. (note, the `arch.conf` config file includes the flag for using `nvidia`'s `drm` module, see the section on installing the `nvidia` drivers)

`/boot/loader/loader.conf`
```
default arch.conf
timeout 4
console-mode max
editor no
```

`/boot/loader/entries/arch.conf`
```
title	Arch Linux
linux	/vmlinuz-linux
initrd	/amd-ucode.img
initrd	/intel-ucode.img
initrd	/initramfs-linux.img
options root="LABEL=arch_root" rw nvidia-drm.modeset=1
```

***Note:*** Make sure to remove the line with `/amd-ucode.img` or `/intel-ucode.img` depending on which ucode you have installed (or just install both, `pacman -S amd-ucode intel-ucode`, it doesn't hurt to have them both)


`/boot/loader/entries/arch-fallback.conf`
```
title	Arch Linux (fallback)
linux	/vmlinuz-linux
initrd	/amd-ucode.img
initrd	/initramfs-linux-fallback.img
options root="LABEL=arch_root" rw
```

***Note:*** Make sure to remove the line with `/amd-ucode.img` or `/intel-ucode.img` depending on which ucode you have installed (or just install both, `pacman -S amd-ucode intel-ucode`, it doesn't hurt to have them both)

And that about wraps it up for configuring your boot loader, as well as configuring you `XBOOTLDR` partition

------

# `pacstrap` installation suggestions

Your initial install of `arch` is extremely bare bones. I rebooted into the install and found out that I had no way of connecting to the internet, and was missing a few necessary packages

Here's a list of packages that I would suggest looking over for ideas on what to install using `pacstrap` (in addition to what the installation guide suggests):

- an editor (I like vim)
- `curl`
- `efibootmgr` - helps with deleting old windows stuff from the efi partition, also useful for changing the "boot" order
- `git`
- `iwd` - for wifi
- `man` - Just install it.
- `man-db`
- `sudo`
- `which`
- `zsh` - For installing ohmyzsh later if you like that

The following are only for graphics (I would read the graphics driver section just to make sure these are right):
- `nvidia`
- `lib32-nvidia-utils` - may need to enable multilib repository in `/etc/pacman.conf`

# Setting up wifi

## Reference Guides

Arch has an extremely good wiki, so if in doubt, here are the different articles that I used when setting this up:

- [iwd](https://wiki.archlinux.org/title/Iwd) - wifi setup, `dhcp` setup
- [systemd-resolved](https://wiki.archlinux.org/title/Systemd-resolved) - `dns` setup


## What we're setting up
Make sure you're `arch-chroot`ed into your new install environment

So, there are a few different ways of doing wifi (this is linux after all), but I went with `iwd` seeing as that's what the install disk had installed.

We will configure `iwd` to use its own `dhcp` client (without this, you can connect to wifi, but won't get an `ip` address without some manual work)

We will also configure `dns` avia `systemd-resolved` (not sure if this is the best way, but its how I did it, and it works, you can use [`resoleconf`](https://wiki.archlinux.org/title/Openresolv) if you want, but this guide won't help much)


## setting up `iwd` and `dhcp`

- Last warning, be `arch-chroot`ed into your new install (or be booted into it, either way)
- Make sure iwd is installed (`pacman -S iwd` or you may have installed this with `pacstrap`)
- enable the iwd daemon - `systemctl enable iwd.service`
- create a file at `/etc/iwd/main.conf` with the following:

```
[General]
EnableNetworkConfiguration=true
[Network]
NameResolvingService=systemd

```

That should get the wifi/dhcp side of things set up, now to configure `dns`

## setting up `systemd-resolved` and `dns`

- installation- there is none, it's pre-installed with `systemd`
- enable `dns` daemon - `systemctl enable systemd-resolved.service`
- configure dns - I needed to do this in order for `dns` to start working, but the `systemd-resolved` guide makes me think you might not, either way, it won't hurt to do this.
- create/edit the file at `/etc/systemd/resolved.conf` Here is a copy of mine.
    - DO NOT just copy paste, it may not work, and there are some good comments in the file that I've removed for clarity:
    - If anything, just append this onto the end of your file (make sure there aren't duplicate `[Resolve]` blocks by commenting out any extra ones)

```
[Resolve]
# The ip address of your router assuming that it runs a dns server/service, starts with either 10.#.#.1 or 192.168.#.1
DNS=192.168.1.1
FallbackDNS=1.1.1.1#cloudflare-dns.com 9.9.9.9#dns.quad9.net 8.8.8.8#dns.google 2606:4700:4700::1111#cloudflare-dns.com 2620:fe::9#dns.quad9.net 2001:4860:4860::8888#dns.google

```

- Link the `resolf.conf` stub - This allows other programs that usually use `resolv.conf` to work with `systemd-resolved` and is important
    - if you are `arch-chroot`ed into your install, exit to the install shell (type `exit` to exit), and run: `ln -sf /run/systemd/resolve/stub-resolv.conf /mnt/etc/resolv.conf`
    - if you are actually booted into your install (not `arch-chroot`ed), run: `ln -rsf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf`

All done. When you reboot into your install, you should now be able to set up a wifi connection.

## connecting to wifi after rebooting into your arch installation (`iwctl`)

Here are the steps to connect to wifi again just in case (they are almost the same as what you would've needed to do while installing `arch`)

```
#> iwctl
[iwd]# device list
 --- [a list of wifi devices will be populated, select your wifi device.]
 --- [mine was wlan0, and I will be using wlan0 as the device name for the rest of this section, insert your device name instead]
[iwd]# station wlan0 scan
 --- [a list of wifi networks will appear, if you're desired network shows up, you're good to continue]
[iwd]# station wlan0 connect [network name]
 --- [Unless you have a crazy network setup, a password prompt will appear, enter the wifi network's password and you should connect]
 --- [Optionally, you can run the following commmand if you want to auto connect to the network on startup:]
[iwd]# known-networks [network name] set-property AutoConnect yes
```

## Notes

I found that, for some reason, my wifi device wouldn't show up if I rebooted the computer using `reboot`. Instead, I had to use `shutdown now` and power the computer back up

-------

# Graphics drivers (nvidia only)

Because I only have nVidia graphics (well, at least that's all I have set up at the moment), I can only help with those

For simplicity, I will also assume the following:

1. You aren't using a custom kernel. If you are, you really shouldn't need this guide anyway
1. You aren't using `linux-lts`
1. You're using `systemd-boot`. If you aren't, you'll need to figure out how to add the correct kernel flags to your boot config

## Referenced guides

- [Nvidia](https://wiki.archlinux.org/title/NVIDIA)
- [multilib](https://wiki.archlinux.org/title/Official_repositories#multilib)

## Installing the drivers

- Unless you're running an older nvidia graphics card, you can just run `pacstrap /mnt nvidia` or `pacman -S nvidia` to install the drivers needed. (gtx1000 series and up)
- Also install the 32 bit drivers (things like `steam` need these anyway, so best to get it done)
    - enable the `multilib` repo by uncommenting the following lines in your `/etc/pacman.conf`:
    - Note: you don't want to accidentally uncomment out the `multilibtesting` list, so be careful
    - For formatting's sake, here's the command you'll run after you enable the `multilib` repo: `pacman -S lib32-nvidia-utils`

```
[multilib]
Include = /etc/pacman.d/mirrorlist

```

## Enabling DRM (Direct Render Mode)

You should do this.

### Adding kernel flag

If you followed the `XBOOTLDR` guide, you've already done part of this, but if not, here's how you include the kernel flag using `systemd-boot`:

In your boot config for arch (probably `/boot/loader/entries/arch.conf`), you probably have a line that looks similar to this:

```
initrd	/initramfs-linux.img
options root="LABEL=[root lable]" rw
```

add the following to the `options` line like so:

```
initrd	/initramfs-linux.img
options root="LABEL=[root lable]" rw nvidia-drm.modeset=1
```

I opted to not do this to my `fallback` config, but have no idea why (I haven't read or heard that you should omit that, I just assumed that I wouldn't be using a desktop environment if I was in fallback mode)

### Early loading DRM modules

Edit your `/etc/mkinitcpio.conf` file:

There should be a line that looks like this:

```
MODULES=()
# Or it could have some modules in it (probably not)
MODULES=([modules in here])
```

Add the following inbetween the parenthesis:

```
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
# Or if there were already modules in there:
MODULES=([modules in here] nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

After doing that, you will need to run the following:

```
mkinitcpio -P
```

Here is a copy of my `/etc/pacman.d/hooks/nvidia.hook` file for running this command every time the drivers are updated (basically a copy paste outta the guide):

```
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux

[Action]
Description=Update Nvidia module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esace; done; /usr/bin/mkinitcpio -P'

```

You are good to go! (well, not really)

## Things that still haven't been configured

So, most modern laptops with nvidia/amd descrete gpu's also have integrated low-power graphics (usually intel), and right now, if you've been following along, the intel graphics haven't been configured to work at all. Refer to the NVIDIA guide for how to do that.

If you, like me, decide it just isn't worth it, boot into your `efi` configuration and disable the integrated graphics. If you don't, you will run into problems getting a desktop environment set up.


# Desktop environment

While `Waylan` is both newer and better than `xOrg`, I opted to just go with `xOrg` because it seemed more straight forward at the time.

I suggest doing the same unless you are feeling adventurous or know more about it than me (why are you looking at this guide then?)

Note: You should also have a non-root user set up for your desktop environment. 

- arch has a pretty good guide on this, it's a little vague on how to enable said non-root user to be a `sudoer`, but it boils down to using `visudo` and adding a line that allows anyone in the user's group to run sudo commands (you can copy from the examples in the file)
- yes there are better ways of doing this, no I didn't bother to figure them out

Note: This won't cover how to autostart your desktop environment at boot, because I personally don't have a problem with signing in before the desktop environment loads (That said, after sign-in it does auto load)

Note: Not gonna cover multimonitor

## Referenced guides

- [xorg](https://wiki.archlinux.org/title/Xorg)
- [Nvidia](https://wiki.archlinux.org/title/NVIDIA) - Has a section on configuring xorg settings

## The guide

run `pacman -S xorg-server xorg-apps`

backup your `/etc/x11/xorg.conf` file (maybe copy it somewhere)

Add the following to either your `~/.bashrc` or `~/.zshrc`:
```
export DISPLAY=":0.0"

run `nvidia-xconfig` fixing any errors, until it succeeds (may have a few errors, I didn't really pay too much attention at this part)

There is some more stuff, but I have to stop the guide at the moment, after youre done with this, I suggest installing `xfce4` as it's been pretty nice.
