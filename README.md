# linuxXPS9570

My installation of ArchLinux with KDE (Plasma and applications) in Dell XPS 9570. I am writing these notes for personal use and **I have not completed them yet**. If you are here by chance, you may make use of them but they are fitted to my specific experience and needs and I would not be able to help with anything (because I am not a particularly experienced linux user).

Some general characteristics assumed:

+ Keyboard = UK; language = UK English; timezone = UK London.
+ Using wired internet connection for installation

The document has three sections: *Installation*, initial *Set up and tunning* , and *Personalization*; increasingly fitted to my personal needs.

# Installation

Set keyboard layout
```
loadkeys uk
```
Check internet connection
```
ping -c 3 archlinux.org
```
Update System Clock
```
# timedatectl set-ntp true
```
## Partitioninng and mounting

Partition disk (mine is `nvme0n1`)
```
# cgdisk /dev/nvme0n1
```
Use existing EFI for dual-boot with Windows 10 (in my case `nvme0n1p2`) and no `swap`. Check EFI partition is at least 500mb! Mine wasn't and, as I do not know much about disk management, I used MiniTool Partition Wizard from Windows to

1. Shrink Win10 Partition "upwards"
2. Copy the empty 128mb reserved windows partition next to the new start of the main partition, and delete the old.
3. Extend the EFI partition

Format main partition (*nvme0n1p5*) with ext4

```
# mkfs.ext4 /dev/nvme0n1p5
```

Mount main and EFI (*nvme0n1p2*) partitions in `/mnt` and `/mnt/boot` respectively

```
# mount /dev/nvme0n1p5 /mnt && mkdir /mnt/boot && mount /dev/nvme0n1p2 /mnt/boot
```
## Installing system

Use `reflector` package to set local fast mirrors. (This is 'automated' and many advise against it)
```
# pacman -Syy && pacman -S reflector && reflector -c "United Kingdom" -f 12 -l 10 -n 12 --save /etc/pacman.d/mirrorlist
```
Install `base` and `base-devel`
```
# pacstrap -i /mnt base base-devel
```
## Generate fstab
```
# genfstab -U -p /mnt >> /mnt/etc/fstab
```
## Login through chroot
```
# arch-chroot /mnt /bin/bash
```
## TZ and locale
Set the time zone:
```
# ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
```
Run `hwclock` to generate `/etc/adjtime`:
```
# hwclock --systohc --utc
```

## Generate locale
Uncomment `en_GB.UTF-8` in `/etc/locale.gen`, generate locale, and set permanently language and keyboardlayout
```
# sed -i 's/#en_GB.UTF-8/en_GB.UTF-8/g' /etc/locale.gen
# locale-gen
# echo 'LANG=EN_GB.UTF-8' > /etc/locale.conf
# echo 'KEYMAP=uk' > /etc/vconsole.conf
```
## Network configuration
```
# echo 'archXPS' > /etc/hostname
```
Edit `/etc/hosts`
```
127.0.0.1  localhost
::1        localhost
127.0.1.1	 archXPS.localdomain	archXPS
```
Enable `dhcpcd` (disable later when enabling `NetworkManager`)
```
# systemctl enable dhcpcd
```
## Passwd for root
```
# passwd
```
## GRUB
```
# pacman -S grub os-prober efibootmgr
# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ArchLinux
# grub-mkconfig -o /boot/grub/grub.cfg
```
## Finish
```
# exit
# umount -R /mnt
# reboot
```

# Set up

## Root

Login as `root` 

```
# useradd -g users -G wheel,storage,power -m manuel
# passwd manuel
```
uncomment `# %wheel ALL=(ALL) ALL` — gives sudo permisison to group `wheel` — and add `Defaults !tty_tickets` — avoids entering password several times per session — in
```
# EDITOR=nano visudo
```

### Optimise building

Modify `CFLAGS` and `CXXFLAGS` in `/etc/makepkg.conf` replacing any `-march` and `-mtune` with `march=native`.

 REPLACE `update-grub` WITH ARCH COMMAND
```
sudo sed -i 's/CFLAGS="-march=x86-64 -mtune=generic/CFLAGS="-march=native/g' /etc/makepkg.conf
sudo sed -i 's/CXXFLAGS="-march=x86-64 -mtune=generic/CXXFLAGS="-march=native/g' /etc/makepkg.conf
sudo sed -i 's/#MAKEFLAGS="-j2"/MAKEFLAGS="-j$(nproc)"/g' /etc/makepkg.conf
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet/GRUB_CMDLINE_LINUX_DEFAULT="quiet CONFIG_DEVMEM=y CONFIG_X86_MSR=y/g' /etc/default/grub && sudo update-grub
```

## Pamac and yay

```
sudo pacman -S --needed base-devel git wget yajl
cd /tmp
git clone https://aur.archlinux.org/package-query.git
cd package-query/
makepkg -si && cd /tmp/
git clone https://aur.archlinux.org/yaourt.git
cd yaourt/
makepkg -si

yaourt -S pamac-aur yay

```


## Desktop Environment

### KDE

#### Xorg
```
# pacman -Syu xorg xorg-init
# localectl set-x11-keymap gb
```
#### Plasma and KDE Apps
```
# yay -Syu kde-applications plasma plasma-wayland-session
```
#### SDDM
```
# yay -Syu sddm archlinux-themes-sddm
# systemctl enable sddm
```
Edit `/usr/lib/sddm/sddm.conf.d/default.conf` changing:

+ In `[Theme]` section:
 ```
 # Current theme name
 Current=archlinux-simplyblack
 
 # Cursor theme used in the greeter
 CursorTheme=breeze-dark
 ```
+ Under both `[Wayland]` and `[X11]`:
 ```
 EnableHiDPI=true
 ```
 Then create
 ```
 sudo mkdir /etc/sddm.conf.d
 ```
 And make file
 ```
 sudo nano /etc/sddm.conf.d/hidpi.conf
 ```
 containing
 ```
 [Wayland]
EnableHiDPI=true

[X11]
EnableHiDPI=true

 ```

#### Latte-Dock
Install and set up super key to open app launcher. (Configure in between, most likely.)
```
sudo pacman -Syu latte-dock
kwriteconfig5 --file ~/.config/kwinrc --group ModifierOnlyShortcuts --key Meta "org.kde.lattedock,/Latte,org.kde.LatteDock,activateLauncherMenu"
```

### GNOME

```
# pacman -S gnome gnome-extra
# systemctl enable gdm
```


## Grub customizer

```
pamac install grub-customizer
```

Run `sudo nano /etc/pacman.conf` and uncomment [multilib] and the next (not the testing!).



## CPU and Thermals

Some tools:

```
yay -S thermald powertop s-tui mprime xsensors
# sudo systemctl enable --now thermald
# sudo systemctl start thermald
# sudo powertop
```
## Disable Goodix Fingerprint Reader (not supported as of 2019.May.04
As explained [here](https://www.reddit.com/r/Dell/comments/alwour/xps_9570_disabling_fingerprint_reader_in_linux/)
```
# echo "1-7" | tee /sys/bus/usb/drivers/usb/unbind
```
In case you want to re-enable it (as there is development in progress (e.g. [Goodix92](https://github.com/IDerr/Goodix92), [goodix_fp_dump](https://gitlab.collabora.com/ao2/goodix_fp_dump) and [this libfprint issue](https://gitlab.freedesktop.org/libfprint/libfprint/issues/43):
```
# echo "1-7" | tee /sys/bus/usb/drivers/usb/bind
```

## Disable discrete GPU whwn not in use with Optimus/Bumblebee
The optimus configuration is a technology that allows an Intel integrated GPU and discrete NVIDIA GPU to be built into and accessed by a laptop. As the discret NVIDIA GPU card eats lots of power, we want to use the intergrated Intel card most of the time and activate/desactivate the NVIDIA GPU card only when required by a defined application.

Regarding to this conversation [3], an optimus configuration well working on the xps 15 9570 is:

Install nvidia, bumblebee, powertop and unigine-valley
```
pacman -S nvidia bumblebee
yay unigine-valley # to test the NVIDIA GPU later
```
Make sure that bbswitch is uninstalled or at least disabled

### Bumblebee configuration

In the `[driver-nvidia]` section of `/etc/bumblebee/bumblebee.conf`
```
# [driver-nvidia] section
Driver=nvidia
PMMethod=none
```
### Server X configuration

for not auto adding gpu edit `/etc/X11/xorg.conf.d/01-noautogpu.conf`
```
Section "ServerFlags"
	Option "AutoAddGPU" "off"
EndSection
```

ipmi modules are loaded together with nvidia and block its unloading. Therefore create `/etc/modprobe.d/disable-ipmi.conf` including
```
  install ipmi_msghandler /usr/bin/false
  install ipmi_devintf /usr/bin/false
```
and `/etc/modprobe.d/disable-nvidia.conf` including:
```
install nvidia /bin/false
```
Finally add nvidia and ipmi in the modprobe.d blacklist to disable this functionality, edit `/etc/modprobe.d/blacklist.conf`
```
blacklist nouveau
blacklist rivafb
blacklist nvidiafb
blacklist rivatv
blacklist nv
blacklist nvidia
blacklist nvidia-drm
blacklist nvidia-modeset
blacklist nvidia-uvm
blacklist ipmi_msghandler
blacklist ipmi_devintf
```
Create 2 GPU management scripts for enabling and disabling discret NVIDIA graphical card [4]:

#### `enablegpu.sh`:
```
#!/bin/sh
# allow to load nvidia module
mv /etc/modprobe.d/disable-nvidia.conf /etc/modprobe.d/disable-nvidia.conf.disable

# Remove NVIDIA card (currently in power/control = auto)
echo -n 1 > /sys/bus/pci/devices/0000\:01\:00.0/remove
sleep 1
# change PCIe power control
echo -n on > /sys/bus/pci/devices/0000\:00\:01.0/power/control
sleep 1
# rescan for NVIDIA card (defaults to power/control = on)
echo -n 1 > /sys/bus/pci/rescan
# someone said that modprobe nvidia is needed also to load nvidia, to check
# modprobe nvidia
```
#### `disablegpu.sh`
```
#!/bin/sh

modprobe -r nvidia_drm
modprobe -r nvidia_uvm
modprobe -r nvidia_modeset
modprobe -r nvidia

# Change NVIDIA card power control
echo -n auto > /sys/bus/pci/devices/0000\:01\:00.0/power/control
sleep 1
# change PCIe power control
echo -n auto > /sys/bus/pci/devices/0000\:00\:01.0/power/control
sleep 1

# Lock system form loading nvidia module
mv /etc/modprobe.d/disable-nvidia.conf.disable /etc/modprobe.d/disable-nvidia.conf
```
### Create service locking NVIDIA GPU on shutdown

A service which locks GPU on shutdown / restart when it is not disabled by disablegpu.sh script is necessary. Otherwise, on next boot nvidia will be loaded together with ipmi modules (even if blacklisted with install command) and it won't be possible to unload them then.
```
sudo nano /etc/systemd/system/disable-nvidia-on-shutdown.service
```
```
[Unit]
Description=Disables Nvidia GPU on OS shutdown

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/bin/true
ExecStop=/bin/bash -c "mv /etc/modprobe.d/disable-nvidia.conf.disable /etc/modprobe.d/disable-nvidia.conf || true"

[Install]
WantedBy=multi-user.target
```
Reload systemd daemons and enable the disable-nvidia-on-shutdown service:
```
 sudo systemctl daemon-reload
 sudo systemctl enable disable-nvidia-on-shutdown.service
```
Allow gpu to poweroff on boot
```
sudo nano /etc/tmpfiles.d/nvidia_pm.conf
```
```
w /sys/bus/pci/devices/0000:01:00.0/power/control - - - - auto
```
Finally check that everything is well configured:

Reboot and verify that nvidia is not loaded by running:
```
        lsmod | grep nvidia
```
Should not return anything

Disconnect / unplug charger and verify the power consumption with powertop is around 5W — FHD screen with no touchscreen — or 7.5W — 4K with touchscreen — on idle (`sudo powertop --auto-tune` previously launched).

Enable GPU by using the `enablegpu.sh` script

Check that the GPU is loaded by using:
```
        nvidia-smi
```
If good, launch unigine-valley with optirun:
```
        optirun unigine-valley
```
unigine-valley needs a GPU to work. Should activate the fans quickly.

Close all nvidia applications and disable gpu with the `disablegpu.sh` script
Check again power consumption, it should have gone back to a similar value as before

### Dell Power Management

```
pamac install libsmbios
yay i8kutils dell-smm-hwmon-i8kutils
```

```
sudo smbios-thermal-ctl -g
sudo smbios-thermal-ctl -i
sudo smbios-thermal-ctl --set-thermal-mode=Balanced
```

### Dell Hwmon sensors

```
modinfo dell-smm-hwmon | grep '^description'
modprobe dell-smm-hwmon ignore_dmi=1
```
You can make these settings permanent by adding the following to `/etc/modprobe.d/dell.conf`:

```
sudo su
echo 'options dell-smm-hwmon ignore_dmi=1' >> /etc/modprobe.d/dell.conf
```

And also by making the HWMON_MODULES variable appears like so in `/etc/conf.d/lm_sensors`:
```
sudo su
echo 'HWMON_MODULES="coretemp dell-smm-hwmon"' >> /etc/conf.d/lm_sensors
```


### Fan Control

```
yay i8kutils dell-bios-fan-control
sudo systemctl enable dell-bios-fan-control

sudo modprobe -v i8k
sudo nano /etc/modprobe.d/dell-smm-hwmon.conf
```
```
sudo modprobe dell-smm-hwmon restricted=0
```

```
sudo echo 'options i8k force=1' >> /etc/modprobe.d/i8k.conf

sudo echo 'i8k' >> /etc/modules-load.d/i8k.conf
```

```
sudo nano /etc/i8kutils/i8kmon.conf

sudo systemctl enable --now i8kmon
sudo systemctl start i8kmon

```

### Suspend
```
cat /sys/power/mem_sleep
echo deep|sudo tee /sys/power/mem_sleep

sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="quiet/GRUB_CMDLINE_LINUX_DEFAULT="quiet mem_sleep_default=deep/g' /etc/default/grub  && sudo update-grub
```
#### In case of instantaneous wakeup
Review the current configuration of wakeup devices:
```
cat /proc/acpi/wakeup

Device  S-state   Status   Sysfs node
...
EHC1      S3    *enabled  pci:0000:00:1d.0
EHC2      S3    *enabled  pci:0000:00:1a.0
XHC       S3    *enabled  pci:0000:00:14.0
...
```

The relevant devices are `EHC1`, `EHC2` and `XHC` (for USB 3.0). To toggle their state you have to echo the device name to the file as root.

```
# echo EHC1 > /proc/acpi/wakeup
# echo EHC2 > /proc/acpi/wakeup
# echo XHC > /proc/acpi/wakeup
```

#### After suspend

Edit `/etc/systemd/system/root-suspend.service` and `systemctl enable --now root-suspend`.

```
[Unit]
Description = Local system resume actions
After = suspend.taget sleep.target

[Service]
Type = simple
ExecStart = /usr/bin/sleep .3
ExcecStartPost = /usr/bin/dell-bios-fan-control 1 ; /usr/bin/dell-bios-fan-control 0 ; echo -e 'power on' | bluetoothctl

[Install]
WantedBy = suspend.target sleep.target

```

### TLP, Thermald and Powertop

```
yay -S tlp thermald powertop

sudo systemctl enable --now tcl
sudo systemctl enable --now tlp
sudo systemctl enable --now tlp-sleep
sudo systemctl enable --now thermald
sudo systemctl enable --now powertop
```
#### Thermald
```
sudo nano /etc/thermald/thermal-conf.xml
```

### Throttled or `lenovo-throttling-fix-git`

```
yay -S throttled
sudo systemctl enable --now lenovo_fix.service
```
And edit `/etc/lenovo_fix.conf` (at least that's its name by 26 April 2019) as desired. My current setup is

```
[GENERAL]
# Enable or disable the script execution
Enabled: True
# SYSFS path for checking if the system is running on AC power
Sysfs_Power_Path: /sys/class/power_supply/AC*/online

## Settings to apply while connected to Battery power
[BATTERY]
# Update the registers every this many seconds
Update_Rate_s: 30
# Max package power for time window #1
PL1_Tdp_W: 44
# Time window #1 duration
PL1_Duration_s: 28
# Max package power for time window #2
PL2_Tdp_W: 44
# Time window #2 duration
PL2_Duration_S: 0.002
# Max allowed temperature before throttling
Trip_Temp_C: 85
# Set cTDP to normal=0, down=1 or up=2 (EXPERIMENTAL)
cTDP: 2

## Settings to apply while connected to AC power
[AC]
# Update the registers every this many seconds
Update_Rate_s: 5
# Max package power for time window #1
PL1_Tdp_W: 44
# Time window #1 duration
PL1_Duration_s: 28
# Max package power for time window #2
PL2_Tdp_W: 44
# Time window #2 duration
PL2_Duration_S: 0.002
# Max allowed temperature before throttling
Trip_Temp_C: 95
# Set HWP energy performance hints to 'performance' on high load (EXPERIMENTAL)
HWP_Mode: True
# Set cTDP to normal=0, down=1 or up=2 (EXPERIMENTAL)
cTDP: 2

[UNDERVOLT.BATTERY]
# CPU core voltage offset (mV)
CORE: -60
# Integrated GPU voltage offset (mV)
GPU: -25
# CPU cache voltage offset (mV)
CACHE: -60
# System Agent voltage offset (mV)
UNCORE: 0
# Analog I/O voltage offset (mV)
ANALOGIO: 0

[UNDERVOLT.AC]
# CPU core voltage offset (mV)
CORE: -60
# Integrated GPU voltage offset (mV)
GPU: -25
# CPU cache voltage offset (mV)
CACHE: -60
# System Agent voltage offset (mV)
UNCORE: 0
# Analog I/O voltage offset (mV)
ANALOGIO: 0

# [ICCMAX.AC]
# # CPU core max current (A)
# CORE: 
# # Integrated GPU max current (A)
# GPU: 
# # CPU cache max current (A)
# CACHE: 

# [ICCMAX.BATTERY]
# # CPU core max current (A)
# CORE: 
# # Integrated GPU max current (A)
# GPU: 
# # CPU cache max current (A)
# CACHE: 

```

Monitor with
```
sudo /usr/lib/throttled/lenovo_fix.py --monitor
```

### Powertop

```
cat << EOF | sudo tee /etc/systemd/system/powertop.service
[Unit]
Description=PowerTOP auto tune

[Service]
Type=idle
Environment="TERM=dumb"
ExecStart=/usr/sbin/powertop --auto-tune

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable powertop.service
```

## Devices


to `/etc/modules-load.d/webcam.conf` seems to be in the right direction.

### Display



### Bluetooth

Change`#AutoEnable=false` to `AutoEnable=true` in `/etc/bluetooth/main.conf` so bluetooth would be enabled at login screen (for mouse and keyboard when docked) and after suspension.

```
sudo sed -i 's/#AutoEnable=false/AutoEnable=true/g' /etc/bluetooth/main.conf
```

## Software

## Nordvpn

```
yay -Syu nordvpn
sudo systemctl enable --now nordvpnsd
systemctl --user enable --now nordvpnud
sudo systemctl start nordvpnsd
systemctl --user start nordvpnud

```

## General (R, LaTeX, VirtualBox, LibreOffice, ...)
```
yay -Syu r texlive-bin texlive-core texlive-latexextra texlive-bibtexextra biber texlive-langextra texlive-fontsextra texlive-formatsextra texlive-humanities texlive-publishers texlive-pstricks texlive-science texlive-science virtualbox virtualbox-guest-iso virtualbox-host-modules-arch virtualbox-sdk telegram-desktop libreoffice-fresh libreoffice-fresh-en-gb
```
```
yay -Syu tk gcc-fortran ed dialog java-runtime perl-tk psutils python2-pygments texlive-pictures java-environment virtualbox-ext-vnc ttf-opensans pstoedit libmythes beanshell unixodbc postgresql-libs mariadb-libs coin-or-mp gnome-shell-extension-appindicator-git libappindicator-gtk3
```
```
yay -Syyu rstudio-desktop dropbox mendeleydesktop spotify skypeforlinux-stable-bin sublime-text-3-imfix tixati whatsapp-nativefier
```

### Sublime-text

#### Package Control

```
import urllib.request,os,hashlib; h = '6f4c264a24d933ce70df5dedcf1dcaee' + 'ebe013ee18cced0ef93d5f746d80ef60'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by) 
```

## Firmware updates
```
yay -S fwupd
```

# NO LONGER NEEDED/USED (2019 May 04)

### Display
#### HiDPI — GNOME

Since GNOME only manages scaling by integers, a screen like this one (relatively not that HiDPI in relatively small form factor) is better setup combining xrandr with GNOME scaling. First set scaling at the maximum convenient (2) and then 'zooming out' with xrandr (1.75x1.75). See the [Arch wiki for HiDPI](https://wiki.archlinux.org/index.php/HiDPI#Desktop_environments)
```
gsettings set org.gnome.settings-daemon.plugins.xsettings overrides "[{'Gdk/WindowScalingFactor', <2>}]"
gsettings set org.gnome.desktop.interface scaling-factor 2
xrandr --output eDP1 --scale 1.75x1.75
```

### Bluetooth

As of 25th of April 2019, it is necessary to install `bluez-git` (5.50.r295.g9e6da22ed-1 > 5.50-264-g750a26cd9) instead of `bluez` to make A2DP audio work with bluetooth headphones. Then, as commented [here](https://aur.archlinux.org/packages/bluez-git/#comment-688092), a manual fix is still required, modifying `/usr/lib/systemd/system/bluetooth.service` to change `ExecStart=/usr/lib/bluetooth/bluetoothd` to `ExecStart=/usr/lib/bluetoothd`. I also found necessary to change `#AutoEnable=false` to `AutoEnable=true` in `/etc/bluetooth/main.conf` so bluetooth would be enabled after suspension.

```
yay -S bluez-git
sudo sed -i 's_ExecStart=/usr/lib/bluetooth/bluetoothd_ExecStart=/usr/lib/bluetoothd_g' /usr/lib/systemd/system/bluetooth.service
```
