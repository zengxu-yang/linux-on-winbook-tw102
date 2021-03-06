# Linux Installation on Winbook TW102 Tablet
This is a record for my attempt of installing Linux on Winbook TW102.
# Hardware

Winbook TW102 is an inexpensive Intel Atom based tablet sold by
[Micro Center](https://www.microcenter.com/product/496688/winbook-tw102-101-quot) with Windows 10 Home 32 bit preinstalled.

Winbook TW102 Specs:
* Intel Atom x-5 Z8350 processor 1.44 GHz
* Intel HD graphics (8086:22b0)
* Emdoor H8811C motherboard
* 2GB LP-DDR3 RAM
* 32GB eMMC Storage
* 10.1" 1280x800 IPS Touch Display
* 802.11b/g/n Wireless
* Bluetooth
* Micro-SD card reader
* 6000 mAh Lithium battery

It is recommended that buying the separately sold compatible keyboard makes it much easier to use Linux on it.

# Linux Installation
## What you need
* A USB drive of at least 8 GB that you can wipe the data.
* A computer with Debian Buster installed.

## System Recovery Drive
This tablet has only 32 GB storage so I suggest wiping out the preinstalled Windows 10 before installing Linux. It is simply
too restrictive for a dual boot. So before installing Linux, it is recommended to backup your personal data in
Windows 10. Find a USB drive of 8 - 16 GB and create a recovery drive to the USB drive in case you need to reinstall
Windows 10. Note that in Windows you can link your license to your Microsoft account as a digital license so you don't need
to enter your product key for Windows 10 reinstallation.

## Creating Linux Installation USB Drive
One caveat is that Winbook TW102 processor supports 64 bit instructions but the UEFI firmware is 32 bit. To install a 64 bit
Linux system you need multiarch installation files. I used the Debian Buster multiarch CD image. Use a working Linux
computer and follow the official Debian instructions to create the Debian installation USB drive. Note that currently Debian
Buster uses kernel 4.19.0-5-amd64.

## Enter BIOS and Change Boot Order
If you don't have the compatible Winbook TW102 keyboard then you need to plug a USB hub and a USB keyboard. Plug the Linux
installation USB drive into the tablet USB port. Power off the tablet. Power on the tablet by pressing the
power button and then immediately press the DEL key to enter BIOS settings. Change the boot order and set the USB drive as
the first booting device. Save and exit.

## GRUB Menu
The tablet should boot from the USB drive now. GRUB menu is displayed but distorted and not readable. Just press Enter key
and you will proceed to the graphical installation dialogs. By default, the display is in portrait mode during installation.

## Touchpad and Network Issue
The touchpad is not working in the installation. You can either just use the keyboard to navigate or plug in a USB mouse.
During the network configuration, the WiFi does not work even you have the required non-free firmware and access points
can be
scanned and displayed. It just fails to configue the wireless network after entering the WPA passphrase. It might work
if there is an open access point but I have not tried. So I used a USB to Ethernet adapter instead and it successfully
detected the wired network and the rest of the installation goes smoothly. When selecting Desktop environment, it is
recommended to use a lightweight desktop manager such as Xfce as this tablet is a low end product.

# Post Installation
## Update Linux Kernel
It is strongly recommended that you update the Linux kernel to the latest version. I updated the kernel to 5.2.6 from Debian Sid.

You need to change the kernel configuration to make it work. Steps:
1. Install `linux-source-5.2` with current version 5.2.6 from Debian sid.
1. `sudo apt-get install linux-source-5.2`
2. `tar xaf /usr/src/linux-source-5.2`
3. `cd linux-source-5.2`
4. Copy the provided `.config` file to the current directory
5. `make deb-pkg` ( or use `make bindeb-pkg` to save compilation time)
6. Use dpkg to install the generated 3 kernel deb files.
## Screen Orientation
### Console Orientation
Edit `/etc/default/grub` and change

`
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
`

to

`
GRUB_CMDLINE_LINUX_DEFAULT="quiet fbcon=rotate:1"
`
### LightDM Greeter Orientation
Edit `/etc/lightdm/lightdm.conf` and uncomment and edit one line below `[Seat:*]`:

`
greeter-setup-script=xrandr --output DSI-1 --rotate right
`
### Desktop Orientation
Open a terminal and run the following command:

`
xrandr --output DSI-1 --rotate right
`

## Things Work
Note that I used the unofficial Debian installation CD image with non-free firmware built in. If you use the official CD
image then you probably need to add non-free firmware for some of the components.
* Intel Graphics works out of the box. By default it uses the portrait mode. In a terminal run
`xrandr --output DSI-1 --rotate right` to switch to the horizontal mode.
* Touchpad works out of the box. It is displayed as "SIPODEV
USB Composite Device Touchpad" in settings with Vendor ID 0x0603 and Product ID 0x0002 in `lsusb`. To enable tap to click,
create a file `/etc/X11/xorg-conf.d/40-libinput.conf` with the following content:
```
Section "InputClass"
        Identifier "libinput touchpad catchall"
        MatchIsTouchpad "on"
        MatchDevicePath "/dev/input/event*"
        Driver "libinput"
        Option "Tapping" "on"
EndSection
```
Then restart lightdm by `systemctl restart lightdm`.
* WiFi works out of the box.
* Audio works out of the box.
* Bluetooth. You need to copy two firmware files to make it work.
```
cd /lib/firmware/rtl_bt sudo wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/rtl_bt/rtl8723bs_fw.bin
cd /lib/firmware/rtl_bt sudo wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain/rtl_bt/rtl8723bs_config-OBDA8723.bin
```

Then reboot and use `blueman-manager` to set up Bluetooth devices.
* Micro-SD card. Tested with a 128 GB micro-SD card.
* Battery monitor. Kernel reconfiguration and rebuilding needed to make it work. See issues below.
* Backlight brightness control. It works after rebuilding the kernel. See issues below.
* USB charging.
* Touchscreen works with issues. See below.
* Suspend/Wakeup works with issues. Suspend should work but hibernation does not work properly. Kernel complains that
it cannot find the image when resuming from hibernation. You can disable hibernation in systemd settings.
## Things Don't Work
* Cameras

## Issues
* GRUB menu is not readable, possibly due to framebuffer display problems. Workaround: Uncomment line

`GRUB_TERMINAL=console`

then run `sudo update-grub` and reboot.
* Framebuffer drive has some issues. Winbook automatically suspends in framebuffer mode if there is just a few seconds of inactivity. This is determined to be s2idle state support issues. Update kernel to 5.2.6 to solve it.
* Battery cannot be detected with the official Debian 10 kernel. [Rebuilding and updating kernel](https://kernel-team.pages.debian.net/kernel-handbook/ch-common-tasks.html#s-common-official) is needed.
* Screen backlight brightness adjustment. There are brightness settings but they have no effect. `dmesg` shows the following error messages:
```
[drm:pwm_setup_backlight [i915]] *ERROR* Failed to own the pwm chip
```
According to online discussions, it is because i915 tends to start before pwm_crc, which causes the issue.

It works after:
1. recompile and upgrade the kernel.
2. Add both pwm-lpss and pwm-lpss0-platform to /etc/initramfs-tools/modules
3. sudo update-initramfs -u
* When both WiFi and Bluetooth are used ( e.g. with Bluetooth headsets), WiFi speed reduces. Possible [analysis](https://h30434.www3.hp.com/t5/Tablets-and-Mobile-Devices-Archive-Read-Only/HP-Stream-7-WiFi-slowdown-when-using-bluetooth/td-p/4859579).
* (This is a general issue of Linux and is not device related) If you set to lock screen after a few minutes of inactivity, then Xfce screen turns black and moving mouse or pressing a key does not bring to the lock screen. The only way to return to the lock screen is by either entering your password blindly or by pressing "Ctrl-Alt-F1" and then "Ctrl-Alt-F8". Xfce uses xflock4 to lock the screen and by default xflock4 uses light-locker. In Debian Buster, light-locker version is 1.8.0. This may be a [bug](https://github.com/the-cavalry/light-locker/issues/114) ([bug report 2](https://bugs.launchpad.net/ubuntu/+source/xorg-server/+bug/1801609)). According to the discussion, there are several possible workaround options:
1. Uninstall light-locker and install xscreensaver.
2. Use slick-greeter instead of lightdm-gtk-greeter.
3. Install xfce4-screensaver. There is a package in Debian sid. You can download the source package and compile it. Note
that you need to edit /usr/bin/xflock4 to add it. See [official doc](https://docs.xfce.org/apps/screensaver/faq#why_doesn_t_the_screensaverlock_screen_activate_when_i_attempt_to_lock_my_computer).
* Touchscreen. Winbook TW102 uses Silead GSL1680 touch controller. First of all, you need the firmware for it to work. I have extracted and uploaded the required firmware
file from its Windows driver file. You need to:
1. Copy `gsl1680-winbook-tw102.fw` to `/lib/firmware/silead`.
2. Patch `drivers/platform/x86/touchscreen-dmi.c` and update your kernel.
3. Use the provided Udev configuration file for calibration. `cp 98-touchscreen-cal.rules /etc/udev/rules.d`. Then reboot.
You need this because Debian Buster uses libinput and the traditional Xorg calibration configuration does not work any more.
* Scroll tearing in Firefox. Disabling smooth scrolling works.
* Syslog reports missing libbd_mdraid.so.2. Install libblockdev-mdraid2 package.
# References and Acknowlegements
* Thank [divVerent](https://github.com/divVerent/linux-on-winbook-tw102) for the original post of installing Linux on Winbook TW102.
* Bluetooth [firmware](https://www.reddit.com/r/linuxmint/comments/aothqi/bluetooth_not_working/) for Winbook TW102.
* LightDM [orientation](https://askubuntu.com/questions/408302/rotated-monitor-login-screen-needs-rotation) settings.
* [Kernel configuration](https://www.spinics.net/linux/fedora/fedora-kernel/msg06764.html) for Cherry Trail support.
* [Backlight](https://techtablets.com/forum/reply/69539/) support.
* [Silead touchscreen firmware extraction and kernel patching](https://github.com/onitake/gsl-firmware).
* [Touchscreen calibration](https://wiki.archlinux.org/index.php/Talk:Calibrating_Touchscreen).
* [Kernel patch](https://bugzilla.kernel.org/show_bug.cgi?id=202543) to fix the INT33FF shared GPIO bug.
