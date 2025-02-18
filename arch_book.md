# Arch_book

Always verify your ISO file.

"Trust, but verify" mindset should always be applied.

https://archlinux.org/download/

Tailored for February 2025 ISO (revised @ 2/18/25).

## 0 - Preliminary steps prior installation

**Erase disk**

	# blkdiscard -zf /dev/nvme0n1

**Erase disk pt.2**

	# cryptsetup open --type plain --key-file /dev/urandom --sector-size 4096 /dev/nvme0n1 to_be_erased
	# dd if=/dev/zero of=/dev/mapper/to_be_erased status=progress bs=512K
	# cryptsetup close to_be_erased
	# reboot

**After reboot, setup the font for better readability**

	# setfont ter-v24n

**Verify that the system is in UEFI mode (should return '64')**

	# cat /sys/firmware/efi/fw_platform_size

**Verify connectivity**

	# ping -c 5 archlinux.org

**Verify that UEFI is in Setup mode, for Secure Boot setup later on**

	# bootctl status | grep "Secure Boot"

## 0.5 - Partitionning (encrypted via "LVM on LUKS" method)

**Partition the disk**

	# fdisk /dev/nvme0n1

+512M (EFI System) > future /boot/ partition (nvme0n1p1)

100%FREE (Linux LVM) > the system with its data (nvme0n1p2)

**Create the LUKS encrypted container at the designated partition**

	# cryptsetup luksFormat /dev/nvme0n1p2

**Open container**

	# cryptsetup open /dev/nvme0n1p2 cryptedlvm

**Create physical volume**

	# pvcreate /dev/mapper/cryptedlvm

**Create volume group**

	# vgcreate volgroup /dev/mapper/cryptedlvm

**Create logical volumes**

	# lvcreate -L 60G volgroup -n root
	# lvcreate -l 100%FREE volgroup -n home
	# lvreduce -L -512M volgroup/home

**Formatting**

	# mkfs.ext4 /dev/volgroup/root
	# mkfs.ext4 /dev/volgroup/home
	# mkfs.fat -F32 /dev/nvme0n1p1

**Mounting**

	# mount /dev/volgroup/root /mnt
	# mount --mkdir /dev/volgroup/home /mnt/home
	# mount --mkdir /dev/nvme0n1p1 /mnt/boot

## 1 - Installation of the system

**Checking mirrors before installing base system**

	# less /etc/pacman.d/mirrorlist

**Install the base system**

	# pacstrap -K /mnt android-{tools,udev} arch-audit arp-scan bat base base-devel btop btrfs-progs curlie croc duf dust eza exfatprogs fastfetch fd fdupes fio gping grml-zsh-config gvfs-{afc,google,goa,gphoto2,mtp,nfs,smb} hdparm hyperfine intel-ucode inter-font iotop john lazygit linux-firmware linux-zen linux-zen-headers lf lvm2 man-db man-pages mcfly ncdu neovim netscanner networkmanager networkmanager-openvpn nmon nmap noto-fonts-{cjk,emoji,extra} nvtop pacman-contrib reflector ripgrep rkhunter sbctl sd sdparm signify smartmontools tcpdump texinfo tldr ttf-mononoki-nerd unrar xdg-user-dirs xfsprogs xh yt-dlp zmap zsh-{autosuggestions,completions,history-substring-search,syntax-highlighting}

**Generate fstab file**

	# genfstab -U /mnt >> /mnt/etc/fstab

**Verify it's correct (also edit fmask/dmask to '0077' to protect /boot)**

	# vim /mnt/etc/fstab

**Chroot**

	# arch-chroot /mnt
	# export PS1="(chroot) ${PS1}"

**Set timezone**

	# ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
	# hwclock --systohc

**Uncomment "en_US.UTF-8" in /etc/locale.gen**

	# nvim /etc/locale.gen
	# locale-gen

**Set LANG variable**

	# nvim /etc/locale.conf

```
LANG=en_US.UTF-8
LC_ALL=C
```

**Set console language**

	# nvim /etc/vconsole.conf

```
KEYMAP=us
```

**Hostname**

	# nvim /etc/hostname

```
twirl
```

**Adjust "Misc options", add "ILoveCandy" beneath them and uncomment "multilib" repo on /etc/pacman.conf**

	# nvim /etc/pacman.conf

```
# Misc options
UseSyslog
Color
#NoProgressBar
CheckSpace
VerbosePkgLists
ParallelDownloads = 7
DownloadUser = alpm
#DisableSandbox
ILoveCandy
```

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

**Update !**

	# pacman -Syu

**Enable misc. services**

	# systemctl enable NetworkManager
	# systemctl enable fstrim.timer
	# systemctl enable systemd-timesyncd
	# systemctl enable pcscd
	# systemctl enable reflector
	# systemctl enable reflector.timer

**Uncomment "ForwardToSyslog=no" and replace "no" by "yes"**

	# nvim /etc/systemd/journald.conf

```
[Journal]
ForwardToSyslog=yes
```

**Set time servers**

	# nvim /etc/systemd/timesyncd.conf

```
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org 0.fr.pool.ntp.org
```

**Repos optimisation**

	# nvim /etc/xdg/reflector/reflector.conf

```
--save /etc/pacman.d/mirrorlist
--protocol https
--country fr
--latest 7
--sort age
```

**AUR optimisation**

Remove any "-march" and "-mtune" CFLAGS, then add "-march=native" in /etc/makepkg.conf : 

	# CFLAGS="-march=native -O2 -pipe ..."

Adjust parallel compilation :

	# MAKEFLAGS="-j$(nproc)"

In /etc/makepkg.conf.d/rust.conf, add "-C target-cpu=native" to RUSTFLAGS :

	# RUSTFLAGS="-C opt-level=2 -C target-cpu=native"

**Modify /etc/mkinitcpio.conf HOOKS**

	# HOOKS=(base systemd autodetect microcode modconf keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)

**Regenerate initramfs**

	# mkinitcpio -P

**Don't forget to put a password to root**

	# passwd

**Setup the bootloader**

	# bootctl install

**Configure the bootloader**

	# nvim /boot/loader/loader.conf

```
default arch.conf
timeout 1
console-mode auto
editor no
```

	# nvim /boot/loader/entries/arch.conf

```
title   Arch Linux zen
linux   /vmlinuz-linux-zen
initrd  /initramfs-linux-zen.img
options rd.luks.name=device-UUID=cryptedlvm root=/dev/volgroup/root rw quiet
```

Tip : to get the device-UUID easily, you can run this into nvim or vim :

	# :read ! blkid /dev/nvme0n1p2

**Get out of there, chrooter**

	# exit

**Unmount everything, then reboot**

	# umount -R /mnt
	# reboot

## 1.5 - Post-installation

**Secure boot setup**

	# sbctl status

should return "Setup mode : enabled"

**Create keys**

	# sbctl create-keys

**Enroll keys**

	# sbctl enroll-keys -m

**Check if sbctl is installed now**

	# sbctl status

**Verifying unsigned files**

	# sbctl verify

**Sign them all**

	# sbctl sign -s @ALL

**Reboot**

	# systemctl reboot --firmware-setup

**Enable secure boot on UEFI with Windows keys, check if Secure boot is enabled now**

	# sbctl status

It should be good to go.

**Install GPU, audio and LaTeX packages**

	# pacman -S lib32-nvidia-utils nvidia-{dkms,utils} pipewire-{alsa,audio,jack,pulse} texlive-{langenglish,langfrench,meta}

**Install GUI apps**

	# pacman -S audacity chromium firefox-i18n-en-us freerdp krita-plugin-gmic libvncserver mkvtoolnix-gui obs-studio pandoc-cli qbittorrent remmina signal-desktop spice-gtk steam thunar yubikey-manager-qt

**Install KDE with its apps (exclude ^5, ^7, ^18 and ^48 from group ; I dont want RDP server / flatpak)**

	# pacman -S ark dolphin-plugins ghostwriter gwenview kate kcalc kdenlive kjournald konsole ksystemlog okular plasma spectacle

**Enable SDDM and Bluetooth at start-up**

	# systemctl enable sddm
	# systemctl enable bluetooth

**Activate sudo for "wheel" usergroup**

	# EDITOR=nvim visudo
	
**Don't forget to create an user**

	# useradd -m -G wheel -s /bin/zsh kydauran

**Password it**

	# passwd kydauran

**CTRL it**

	# su - kydauran

**Disable root user**

	$ sudo passwd -l root
	$ sudo passwd -d root

and ... reboot ? You should now have a functional GUI.

## 2 - Some tweaks here and there

**Install NvChad**

	$ git clone https://github.com/NvChad/starter ~/.config/nvim && nvim
	
**Fix ghostwriter crashing when inputting one character (needs investigation regarding why it crashes)**

In $HOME/.config/kde.org/ghostwriter.conf, add :

```
[Preview]
lastUsedExporter=cmark-gfm
```

**Improve font readability**

In /etc/environment, add :

```
FREETYPE_PROPERTIES="cff:no-stem-darkening=0 autofitter:no-stem-darkening=0"
```

**Improve ZSH**

	$ nvim .zshrc

At EOF, add :

```
eval "$(mcfly init zsh)"

source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
```

**Install AUR helper**

	$ cd /tmp
	$ git clone https://aur.archlinux.org/paru.git
	$ cd paru/
	$ makepkg -sri

**Install stuff from AUR**

	$ paru -S android-sdk-build-tools mp3tag mullvad-vpn-bin proton-ge-custom-bin protontricks r2modman-bin teamspeak visual-studio-code-bin

## 99 - Sources used 

https://wiki.archlinux.org/title/User:Bai-Chiang

https://wiki.archlinux.org/

https://blog.aktsbot.in/no-more-blurry-fonts.html

https://github.com/KDE/ghostwriter/discussions/890
