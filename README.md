# Razer Blade Stealth Linux

**Razer Blade Stealth** (late 2016, UHD / HiDPI) Linux (Ubuntu & Arch) setup.
Contact me at twitter [@rolandguelle](https://twitter.com/rolandguelle) for questions.

Solved issues:
* Suspend loop
* Caps-Lock freeze
* Disabled touchpad after suspend
* Wifi connection lost randomly
* Firefox touchscreen scrolling
* Thunderbolt
* Razer Core
* Razer Core + NVIDIA GPU

Unsolved issues:
* Webcam

Current Setup:
* Ubuntu 17.10 & Wayland
* (Maybe) outdated Infos: Ubuntu X11, [Arch](#arch)

# Preparation

* Run Bios updates via installed Windows 10
	* http://www.razersupport.com/gaming-systems/razer-blade-stealth/
	* Direct Links:
		* http://dl.razerzone.com/support/BladeStealthH2/BladeStealthUpdater_v1.0.5.3_BIOS6.05.exe.7z
		* http://dl.razerzone.com/support/BladeStealthH2/BladeStealthUpdater_v1.0.5.0.zip

# Ubuntu 17.10

## Install

* Resize disk on Windows
    * https://www.howtogeek.com/101862/how-to-manage-partitions-on-windows-without-downloading-any-other-software/ 
* Fresh Ubuntu 17.10 installation, reboot
* Software & Updates
	* Additional Drivers: Using Processor microcode firmware for Intel CPUs from intel-microcode (proprietary)
        * (Secure boot was disabled during installation, but is now activated)
    * Packages: main, universe, restricted, multiverse, artful-proposed

## Works

Other tutorials reports issues for some components.
Maybe it depends on other distros, Ubuntu versions or configurations, but on my machine are no issues

### Graphic Card

Not needed any more:

* X11: "AccelMethod"  "uxa"
* Kernel: i915.enable_rc6=0 

### HDMI

Since 4.10.6 kernel HDMI out works.

### Thunderbolt

USB & video works on my 27'' Dell monitor with a (Apple) USB-C (HDMI, USB) adapter, without any modifications with kernel 4.13.x.
Including USB to ethernet :)

## Issues

### Suspend Loop

After resume, the system loops back in suspend.

The system send an ACPI event where the [kernel defaults](https://patchwork.kernel.org/patch/9512307/) are different.

#### Grub Kernel Parameter

This kernel parameter changes the defaults:
```shell
$ sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="button.lid_init_state=open"
```

Update grub
```shell
$ sudo update-grub
```

### Caps-Lock Crash

The RBS crashes ~~randomly~~ mostly if you hit "Caps Lock". The build-in driver causes the problem.

#### Disable Capslocks

Modify /etc/default/keyboard following line, replacing capslocks by a second ctrl:

```shell
$ sudo nano /etc/default/keyboard 
XKBOPTIONS="ctrl:nocaps"
```

#### X11: Disable Built-In Keyboard Driver

Get your keyboard description and use it instead of "AT Raw Set 2 keyboard":
```shell
$ xinput list | grep "Set 2 keyboard"
```

[Config](etc/X11/xorg.conf.d/20-razer.conf)
```
Section "InputClass"
    Identifier      "Disable built-in keyboard"
    MatchProduct    "AT Raw Set 2 keyboard"
#	MatchProduct    "AT Translated Set 2 keyboard"
    Option          "Ignore"    "true"
EndSection
```

Re'disable keyboard after suspend, [Script](etc/pm/sleep.d/20_razer):

```shell
#!/bin/sh
# http://askubuntu.com/questions/873626/crash-when-toggling-off-caps-lock

case $1 in
    resume|thaw)
	  xinput set-prop "AT Raw Set 2 keyboard" "Device Enabled" 0
#	  xinput set-prop "AT Translated Set 2 keyboard" "Device Enabled" 0
    ;;
esac
```

### Touchpad Suspend

Touchpad fails resuming from suspend with:

```
rmi4_physical rmi4-00: rmi_driver_reset_handler: Failed to read current IRQ mask.
dpm_run_callback(): i2c_hid_resume+0x0/0x120 [i2c_hid] returns -11
PM: Device i2c-15320205:00 failed to resume async: error -11
```

Temporary fix:

```shell
$ sudo rmmod i2c_hid && sudo modprobe i2c_hid
```

#### Libinput-gestures

[Libinput-gestures](https://github.com/bulletmark/libinput-gestures) solves the problem:

```shell
$ sudo gpasswd -a $USER input
$ sudo apt install xdotool wmctrl libinput-tools
$ git clone http://github.com/bulletmark/libinput-gestures
$ cd libinput-gestures
$ sudo ./libinput-gestures-setup install
```

Logout - Login (if not, you get an error)

```shell
$ echo "gesture swipe right     xdotool key ctrl+alt+Right" > .config/libinput-gestures.conf
$ echo "gesture swipe left     xdotool key ctrl+alt+Left" >> .config/libinput-gestures.conf
$ libinput-gestures-setup autostart
$ libinput-gestures-setup start
```
_(if you prefer natural scrolling, change up/down)_

### Touchscreen & Firefox

Firefox doesn't seem to care about the touchscreen at all.

#### Xinput2

Tell Firefox to use xinput2

```
$ sudo nano /etc/environment
```
Add the line at the end:
```
MOZ_USE_XINPUT2=1
```
Save and log off/in

### Unstable WIFI

Wireless connection gets lost randomly.

#### Update Firmware

Updating the firmeware helps.

* Backup /lib/firmware/ath10k/QCA6174/hw3.0/
* Download & Update Firmware:
```shell
$ wget https://github.com/kvalo/ath10k-firmware/raw/master/QCA6174/hw3.0/board.bin
$ sudo mv board.bin /lib/firmware/ath10k/QCA6174/hw3.0/board.bin
$ wget https://github.com/kvalo/ath10k-firmware/raw/master/QCA6174/hw3.0/board-2.bin
$ sudo mv board-2.bin /lib/firmware/ath10k/QCA6174/hw3.0/board-2.bin
$ wget https://github.com/kvalo/ath10k-firmware/raw/master/QCA6174/hw3.0/4.4.1.c1/firmware-6.bin_RM.4.4.1.c1-00031-QCARMSWP-1
$ sudo mv firmware-6.bin_RM.4.4.1.c1-00031-QCARMSWP-1 /lib/firmware/ath10k/QCA6174/hw3.0/firmware-6.bin
```

### Onscreen Keyboard

Everytime the touchscreen is used, an onscreen keyboard opens.

#### Block caribou

Blocks caribou (the on screen keyboard) from popping up when you use a touchscreen. 

Installation:
```
$ mkdir -p ~/.local/share/gnome-shell/extensions/cariboublocker@git.keringar.xyz
$ cd ~/.local/share/gnome-shell/extensions/cariboublocker@git.keringar.xyz
$ wget https://github.com/keringar/cariboublocker/raw/master/extension.js
$ wget https://github.com/keringar/cariboublocker/raw/master/metadata.json
$ cd
$ gsettings get org.gnome.shell enabled-extensions
$ gsettings set org.gnome.shell enabled-extensions "['cariboublocker@git.keringar.xyz']"
```
Logout / Login


### Multiple Monitors

Using a HiDPI and "normal" monitor works on _some_ applications, but not in Firefox & Chrome.

#### Switch to 1920x1080

Switch the internal HiDPI screen to **1920x1080** when using your RBS with a non HiDPI external monitor.
Gnome _remembers_ the monitor and switch back to 4k when unplugging the screen.


## Unsolved Issues

### Keyboard Colors

Currently not used.

Issue: (settings are lost after suspend):
* https://github.com/openrazer/openrazer/issues/342
(Gnome, Wayland)

### Webcam

Working only with 176x in cheese, or 640x480 in guvcview with 15/1 frames.

## Tweaks

### Power Management

TLP is an advanced power management tool for Linux that tries to apply tweaks for you automatically, depending on your Linux distribution and hardware.

```shell
$ sudo apt-get install tlp tlp-rdw
$ sudo systemctl enable tlp
```

### Touchpad

#### Click, Tap, Move

macOS touchpad feeling:

```shell
$ sudo apt install gnome-tweak-tool
```
* Keyboard & Mouse
* Click Method: Fingers

#### X11: Synaptics Configuration

Disable touchpad while typing and some other tunings:
* [50-synaptics-ubuntu.conf](etc/X11/xorg.conf.d/50-synaptics-ubuntu.conf)

### Display Scaling

At native resolution, the internal HiDPI 4K display with 100% scale might be too tiny and frustrating for some, and with 200% scale is too large to be useful, luckily with Ubuntu 17.10 shipping with Gnome3, a native screen scaling solution is provided, however it's limited to 2 options: `100%` and `200%`.

To enable more scaling options run the following command:

```
$ gsettings set org.gnome.mutter experimental-features "['scale-monitor-framebuffer']"
```

After login/logout, you'll get more scaling options under Settings > Devices > Displays.

If the fonts are blurry, reset this setting:
```
$ gsettings reset-recursively org.gnome.mutter
```

### Theme 

My Ubuntu/Gnome tweaks :)

#### "Capitaine" Cursors

* Install https://github.com/keeferrourke/capitaine-cursors
* Select via tweaks tool

#### Applicatioins Theme

* apt install arc-theme
* Select (Arc-Darker) via tweaks tool

#### Fonts

* Window-Title: Garuda Regular 11
* Interface: Ubuntu Regular 11
* Document: Sans Regular 12
* Monospace: Monospace Regular 12


## Razer Core

Using hardware like the Razer Core with Linux sounds like fun :)

### Thunderbolt

* BIOS Setting: Thunderbolt security: User

Authorize thunderbolt by user:
```
$ echo "1" | sudo tee /sys/bus/thunderbolt/devices/0-0/0-1/authorized 
```
or with [razercore start](#razercore).

* USB works
* Ethernet works

### Discrete NVIDIA GPU

Goal is the _same_ setup like Windows:
 * Use the normal setup (Wayland, Gnome) without the Razer Core
 * Hotplug Razer Core (without reboot, login/logout) with user interaction
 * Run selected applications on Razer Core / NVIDIA GPU
 * Unplug the Razer Core without freezing the system

#### NVIDIA Prime

Install NVIDIA Prime and set it to "intel":
```
$ sudo apt install nvidia-prime
$ sudo prime-select intel
```

#### NVIDIA GPU Driver

Update to driver (I use the latest NVIDIA drivers & Ubuntu 'pre-released updates')
```
$ sudo add-apt-repository ppa:graphics-drivers/ppa
$ sudo apt update
$ sudo apt install nvidia-387
```

Add missing NVIDIA symlinks (? not sure if this is only my local problem)
```
$ ln -s /usr/lib/nvidia-387/bin/nvidia-persistenced /usr/bin/nvidia-persistenced 
$ ln -s /usr/lib/nvidia-387/libnvidia-cfg.so.1 /usr/lib/libnvidia-cfg.so.1
```

#### Bumblebee

Install Bumblebee:
```
$ sudo apt-get install bumblebee bumblebee-nvidia primus linux-headers-generic
$ sudo gpasswd -a $USER bumblebee
```

Update bumblebee [config](etc/bumblebee/bumblebee.conf)
```
$ sudo nano /etc/bumblebee/bumblebee.conf 
Driver=nvidia
KernelDriver=nvidia-387
LibraryPath=/usr/lib/nvidia-387:/usr/lib32/nvidia-387
XorgModulePath=/usr/lib/nvidia-387/xorg,/usr/lib/xorg/modules
```

#### Test GPU With optirun

Reboot :)

Check if NVIDIA driver is used: 
```
$ optirun glxinfo | grep OpenGL
OpenGL vendor string: NVIDIA Corporation
```

#### Play Extreme Tuxracer :)

Replace this with your favorite 3D application/game ;)

```
PRIMUS_SYNC=1 vblank_mode=0 primusrun etr
```
* PRIMUS_SYNC sync between NVIDIA and Intel
    * 0: no sync, 1: D lags behind one frame, 2: fully synced
* ignore the refresh rate of your monitor and just try to reach the maximux fps
    * vblank_mode=0 

Tested with "Extremetuxracer" and different games on "Steam" (Saints Row IV, Life is Strange, ...) at 4k resolution on Wayland & X11.

### razercore

This (ugly) script helps with the typical tasks.
Copy [razercore](bin/razercore) into ~/bin or somewhere else in your path and make it executable.

Usage:
* razercore start
    * PCI rescan
    * Authorize thunderbolt
    * Check status (aka razercore status)
* razercore status
    * status of connection
* razercore stop
    * remove PCI device
* razercore restart
    * stop & start
* razercore exec <prog>
    * start prog on external gpu
    * example: razercore exec steam

# Arch (Antergos)

Tested with [Antergos](https://antergos.com/) Arch, but other Arch distros should work too.
Maybe outdated.

## Suspend Loop

```shell
$ sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet button.lid_init_state=open"
```

Update Grub
```shell
$ sudo grub-mkconfig -o /boot/grub/grub.cfg
```

## TLP

Install TLP tools:
```shell
$ sudo pacman -S tlp tlp-rdw
```

Enable TLP tools:
```shell
$ sudo systemctl enable tlp
$ sudo systemctl enable tlp-sleep
```

## Keyboard Colors

See Ubuntu Setup

## Gnome, Workspaces, Gestures

See Ubuntu Setup

## Touchpad

### Synaptics (X11)

Disable touchpad while typing and some other tunings:
* [50-synaptics-arch.conf](etc/X11/xorg.conf.d/50-synaptics-arch.conf)

### libinput (X11)

"libinput" configration for X11 [60-libinput.conf](etc/X11/xorg.conf.d/60-libinput.conf)

### libinput (Wayland)

Adjust "libinput" coordinate ranges for absolute axes:
* [61-evdev-local.hwdb](etc/udev/hwdb.d/61-evdev-local.hwdb)

```shell
$ sudo cp etc/udev/hwdb.d/61-evdev-local.hwdb /etc/udev/hwdb.d/61-evdev-local.hwdb
```

Update settings:
```shell
$ sudo systemd-hwdb update
$ sudo udevadm trigger /dev/input/event*
```

### Multiple Monitors

See Ubuntu Setup

# Credits

## References

* https://wiki.archlinux.org/index.php/Razer
* https://wayland.freedesktop.org/libinput/doc/latest/absolute_coordinate_ranges.html
* https://github.com/systemd/systemd/pull/6730
* https://wiki.archlinux.org/index.php/TLP
* http://www.webupd8.org/2016/08/how-to-install-and-configure-bumblebee.html
* https://extensions.gnome.org/extension/1326/block-caribou/
* https://github.com/bulletmark/libinput-gestures
* http://askubuntu.com/questions/849888/suspend-not-working-as-intended-on-razer-blade-stealth-running-xubuntu-16-04/849900
* http://askubuntu.com/questions/873626/crash-when-toggling-off-caps-lock

## Thanks

* https://github.com/xlinbsd
* https://github.com/tomsquest
* https://github.com/ahmadnassri
* https://github.com/lucaszanella