### ArchLinux on MacBook Pro Retina 13" (Model A1502)

#### Resources
Much of this has been adapted from https://gist.github.com/terlar/6143325. Other parts are mostly from countless Google searches and manic arch wiki scouring.

#### Pre-install on Mac
This needs revisiting. I already had a partition available from a previous install after the Yosemite update. If all else fails ``diskutil corestorage revert UUID`` will remove Apple's logical volume update and allow diskutil to resize a partition normally.

```
diskutil corestorage list # Need the bottom-most logical volume UUID
diskutil corestorage resizeStack <UUID> <size>

# If that doesn't work, some tinkering may be required similar to this:
diskutil corestorage resizeStack UUID <size>

# Once you've got a free partition, you need to
# format an JHFS+ partition for booting via rEFInd
diskutil list # Need free space partition
diskutil partionDisk 2 /dev/diskXsY JHFS+ 'rEFInd' 100M HSF+ 'doesntmatter' R
```

##### rEFInd
First, download rEFInd
```
<rEFInd>/install /Volumes/rEFInd --ownhfs /dev/diskXsY # 'rEFInd' parition
```
- Edit /Volumes/rEFInd/System/Library/CoreServices/refind.conf
  - scan_delay 1
  - dont_scan_volumes "Recovery HD"
  - timeout 5
- Theme rEFInd (currently https://github.com/EvanPurkhiser/rEFInd-minimal)

#### Installing Arch

**If tethering from a phone:**

```
ip link # get device name for USB tether
dhclient # dhcp didn't work for my phone
```

##### Bootstrapping

```
gdisk /dev/sda

mkfs.ext4 /dev/sdaY
# and any other partitions
mkswap /dev/sdaZ

mount /dev/sdaY /mnt
# if using a seperate home partition
mkdir /mnt/home && mount /dev/sdaY /mnt/home

pacstrap /mnt base base-devel wpa_actiond wpa_supplicant dialog linux-headers sudo
genfstab -p /mnt >> /mnt/etc/fstab
# SSD optimize /etc/fstab
# 'defaults,noatime,discard,data=writeback' are generally alright options

# if using the pre-built wireless driver method
cp ~/broadcom-wl-6.30.223.248-2-x86_64.pkg.tar.xz /mnt/root

arch-chroot /mnt /bin/bash
echo name > /etc/hostname
ln -s /usr/share/zoneinfo/America/Chicago /etc/localtime

echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8

vi /etc/mkinitcpio.conf
# Add resume _after_ block and lvm2, but _before_ filesystems
mkinitcpio -p linux

# rEFInd
echo '"Arch Linux" "root=/dev/sdXY resume=/dev/sdXZ rw splash loglevel=3 rootflags=data=writeback libata.force=noncq acpi_osi=Linux acpi=force acpi_enforce_resources=lax i915.modeset=1 i915.enable_rc6=1 i915.enable_fbc=1 i915.lvds_downclock=1 initrd=boot/initramfs-linux.img"' > /boot/refind_linux.conf
# NOTE: For intel microcode updates, you'll need to install the intel-ucode package
# The initrd option must also be placed early in the boot process
# NB: I'm not sure about all the ACPI options, but we'll see how they turn out

useradd -m -g users -G wheel -s /bin/bash username

# Setup passwords
passwd
passwd username
```
Before rebooting, install dhclient and the wireless driver. dhclient is for tethering in case someone goes wrong with wireless setup.

##### Wi-Fi Setup

```sh
yaourt -S broadcom-wl-dkms

sudo wifi-menu -o wlp3s0
sudo systemctl enable netctl-auto@wlp3s0.service
```

##### Basic Tools

```sh
sudo pacman -S alsa-utils powertop dnsutils net-tools acpi openssh unzip unrar cronie git ack
sudo systemctl enable sshd cronie
```

##### Yaourt (AUR)

```sh
sudo pacman -Syy

curl -O https://aur.archlinux.org/packages/pa/package-query/package-query.tar.gz
tar -xzf package-query.tar.gz && cd package-query
makepkg -s
sudo pacman -U package-query-*.pkg.tar.xz

curl -O https://aur.archlinux.org/packages/ya/yaourt/yaourt.tar.gz
tar -xzf yaourt.tar.gz && cd yaourt
makepkg -s
sudo pacman -U yaourt*.pkg.tar.xz

yaourt -S powerpill

```
- Edit `/etc/makepkg.conf` and add `MAKEFLAGS="-j4"`


##### Drivers
```sh
yaourt -S acpid xf86-video-intel broadcom-wl-dkms xf86-input-mtrack-git macfanctld-git
sudo cp /this/repo/xorg.conf.d/50-synaptics.conf /etc/X11/xorg.conf.d/
sudo systemctl enable acpid macfanctld

yaourt -S pacman -S bluez bluez-libs bluez-utils
```

**Disable that annoying red headphone jack light**
```sh
amixer -c 0 sset IEC958 off
echo "options snd-hda-intel model=mbp111" >> /etc/modprobe.d/alsa-base.conf
```
If that didn't work (it probably won't)
```sh
yaourt -S hda-verb
# As root
crontab -l | { cat; echo "@reboot /usr/bin/hda-verb /dev/snd/hwC1D0 0x0e SET_POWER_STATE 0x03"; } | crontab -
```
Make sure to have a cron demon running.

To get sound working, add this to `/etc/asound.conf`:
```
defaults.pcm.card 1
defaults.pcm.device 0
defaults.ctl.card 1
defaults.ctl.device 0
```

##### Tools
```sh
yaourt -S kbdlight xorg-xbacklight
yaourt -S zsh zsh-completions zsh-syntax-highlighting
yaourt -S gvim-pythonsix
gem install gist
```

##### Printer
```sh
sudo pacman -S cups
```
##### User
```
chsh -s /usr/bin/zsh
usermod -a -G audio,video,network,power,disk,storage,optical,lp,systemd-journal username
```

#### GUI

##### Xorg
```sh
sudo pacman -S xorg-server xorg-xrdb libnotify xbindkeys xorg-xmodmap
sudo cp /this/repo/xorg.conf.d/10-monitor.conf /etc/X11/xorg.conf.d/
```
To get notifications working from cron jobs, you might need to add to the top of your crontab or give pass it your ```DBUS_SESSIONS_BUS_ADDRESS```.
```
DISPLAY=:0.0
XAUTHORITY=/home/user/.Xauthority
```

##### Fonts
```sh
yaourt -S ttf-dejavu ttf-symbola ttf-droid adobe-source-code-pro-fonts ttf-linux-libertine ttf-ubuntu-font-family ttf-freefont wqy-zenhei ttf-mac-fonts ttf-envy-code-r ttf-opensans
yaourt -S otf-ipafont ttf-tw ttf-baekmuk ttf-mathtype
```

##### Login Manager
```sh
yaourt -S slim slim-theme-arch-triforce
sudo systemctl enable slim.service
```
- Edit /etc/slim.conf, change theme to slim-theme-arch-triforce

##### Openbox
```sh
yaourt -S openbox tint2-svn compton-git
sudo cp /this/repo/compton/compton_openbox /usr/local/bin/
```

##### Awesome WM
```sh
yaourt -S awesome obvious-git compton-git
```

##### Terminal Emulator
```sh
yaourt -S rxvt-unicode urxvt-tabbedex-git urxvt-perls-git
```

##### Apps
```sh
yaourt -S scrot feh chromium chromium-pepper-flash gimp inkscape conky-lua pidgin-mini
yaourt -S ds-digital-fonts lm_sensors # for conky

# For skype
yaourt -S skype-restricted xorg-xhost lib32-apulse
# May want to edit PKGBUILD and remove gksu for something else.
# Note, if you do this, need to also edit /usr/bin/skype wrapper script
echo "user ALL=(_skype) NOPASSWD: /var/skype/skype" >> /etc/sudoers
sudoedit /var/skype/skype
# Modify exec line to: exec /usr/bin/apulse32 "$LIBDIR/skype/skype" "$@"

```

##### Media
```sh
yaourt -S mplayer google-talkplugin
```

##### Development
```sh
yaourt -S postgresql mongodb redis heroku-client
\curl -sSL https://get.rvm.io | bash -s stable
```

### Performance
##### Power
```
sudo pacman -S laptop-mode-tools cpupower pm-utils upower profile-sync-daemon anything-sync-daemon
sudo systemctl enable laptop-mode cpupower psd psd-resync asd asd-resync
```
laptop-mode
- Edit `/etc/laptop-mode/laptop-mode.conf` with value `LM_BATT_MAX_LOST_WORK_SECONDS=15`
- Edit `/etc/laptop-mode/conf.d/runtime-pm.conf` with value `AUTOSUSPEND_TIMEOUT=1`
- Edit `/etc/laptop-mode/conf.d/intel-hda-powersave.conf` with value `INTEL_HDA_DEVICE_TIMEOUT=1`

cpupower
- Edit `/etc/defaults/cpupower` with value `governer='powersave'`
