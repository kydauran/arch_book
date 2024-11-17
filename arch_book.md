# Arch_book

Always verify your ISO file (b2sum/sha256 ; gpg/pacman-key).
I won't cover it because it should be automatic anyway thanks to torrent, however it's good habit to verify everything you download, when it's possible.

"Trust, but verify" mindset should always be applied.

https://archlinux.org/download/

Tailored for November 2024 ISO (revised @ 11/17/24).

## 0 - Preliminary steps prior installation

**Erase disk**

	# blkdiscard -zf /dev/nvme0n1

**Erase disk pt. 2**

	# cryptsetup open --type plain --key-file /dev/urandom --sector-size 4096 /dev/nvme0n1 to_be_erased
	# dd if=/dev/zero of=/dev/mapper/to_be_erased status=progress bs=512K
	# cryptsetup close to_be_erased
	# reboot

**After reboot, setup the font for better readability**

	# setfont ter-v24n

**Verify the system is in UEFI mode (should return '64')**

	# cat /sys/firmware/efi/fw_platform_size

**Verify connectivity**

	# ping -c 5 archlinux.org

**Verify UEFI is in Setup mode, for later**

	# bootctl status | grep "Secure Boot"

## 0.5 - Partitionning (encrypted via LVM on LUKS method)

	# fdisk /dev/nvme0n1

+512M (EFI System) > future /boot/ partition (nvme0n1p1)
100%FREE (Linux LVM) > the system with its data (nvme0n1p2)

**Layout setup**

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

**Formating**

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

**Install the base system (please don't get a seizure over this one)**

	# pacstrap -K /mnt base base-devel linux-zen linux-zen-headers linux-firmware yt-dlp nvtop ncdu lf tldr btop arch-audit rkhunter ripgrep duf dust intel-ucode zmap nmap john android-tools android-udev signify fdupes eza bat hdparm sdparm smartmontools fd mcfly sd hyperfine gping xdg-user-dirs curlie xh dog neovim grml-zsh-config lvm2 arp-scan fastfetch tcpdump reflector pacman-contrib man-db man-pages texinfo zsh-{completions,autosuggestions,history-substring-search,syntax-highlighting} networkmanager lazygit gvfs-{afc,goa,google,gphoto2,mtp,nfs,smb} croc exfat-utils btrfs-progs xfsprogs firewalld dnsmasq swtpm sbctl inter-font noto-fonts-{cjk,extra,emoji} ttf-mononoki-nerd nmon iotop fio netscanner unrar

**Generate fstab file**

	# genfstab -U /mnt >> /mnt/etc/fstab

**Verify it's correct (also edit fmask/dmask to 0077)**

	# vim /mnt/etc/fstab

**Chroot**

	# arch-chroot /mnt
	# export PS1="(chroot) ${PS1}"

**Timezone**

	# ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
	# hwclock --systohc

**Uncomment en_US.UTF-8 on /etc/locale.gen**

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

**Uncomment multilib repo and add ILoveCandy to "Misc options" on /etc/pacman.conf**

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
	# systemctl enable firewalld
	# systemctl enable systemd-timesyncd
	# systemctl enable pcscd
	# systemctl enable reflector
	# systemctl enable reflector.timer

**Uncomment "ForwardToSyslog=no" and replace 'no' by 'yes'**

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

Remove any -march and -mtune CFLAGS, then add -march=native in /etc/makepkg.conf. 

	# CFLAGS="-march=native -O2 -pipe ..."

Add -C target-cpu=native to RUSTFLAGS:

	# RUSTFLAGS="-C opt-level=2 -C target-cpu=native"

Adjust parallel compilation :

	# MAKEFLAGS="-j$(nproc)"

**Modify /etc/mkinitcpio.conf HOOKS**

	# HOOKS=(base systemd autodetect microcode modconf keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)

**Regenerate initramfs**

	# mkinitcpio -P

**Don't forget to put a password to root**

	# passwd

**Bootloader**

	# bootctl install

**Configure bootloader**

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
options rd.luks.name=device-UUID=cryptedlvm root=/dev/volgroup/root rw quiet splash nvidia_drm.modeset=1
```

Tip : to get the device-UUID easily, you can run this into nvim or vim :

	# :read ! blkid /dev/nvme0n1p2

**Get out of there, chrooter**

	# exit

**Unmount everything, then reboot**

	# umount -R /mnt
	# reboot

## 1.5 - Post-installation (the fun begins)

**Secure boot setup**

	# sbctl status

should still return "Setup mode"

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

**Enable secure boot on UEFI with "Windows keys", check if Secure boot is enabled now**

	# sbctl status

Should be good to go.

**Install "fire and forget" things**

	# pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils texlive-meta texlive-langfrench texlive-langenglish pipewire-{audio,alsa,pulse,jack}

**Install apps**

	# pacman -S yubikey-manager-qt obs-studio qbittorrent steam krita-plugin-gmic signal-desktop audacity pycharm-community-edition thunar pandoc-cli chromium mkvtoolnix-gui teamspeak3 virt-manager

**Install KDE with its apps (exclude ^5, ^7, ^18 and ^48 from group ; I dont want RDP / flatpak on a mutable system)**

	# pacman -S plasma ark dolphin-plugins gwenview kcalc kdenlive konsole ksystemlog kjournald okular kate spectacle ghostwriter

**Enable SDDM at start-up**

	# systemctl enable sddm

**Also Bluetooth**

	# systemctl enable bluetooth

**Activate sudo for wheel usergroup, or use run0**

	# EDITOR=nvim visudo
	
**Don't forget to create an user**

	# useradd -m -G wheel -s /bin/zsh kydauran

**Password it**

	# passwd kydauran

**CTRL it**

	# su - kydauran

now he's CTRL'd

**Punish root user**

	$ sudo passwd -l root
	$ sudo passwd -d root

and ... reboot ? you should now have a functional GUI.

## 2 - Some tweaks here and there

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

	$ paru -S mullvad-vpn-bin mp3tag android-sdk-build-tools qflipper-bin protontricks proton-ge-custom-bin r2modman-bin

**Install NvChad**

	$ git clone https://github.com/NvChad/starter ~/.config/nvim && nvim

**Install customizable taskbar for KDE**

Panel Colorizer is pretty good (via "Install a new widget" within KDE)

**Improve font readability**

In /etc/environment, add :

```
FREETYPE_PROPERTIES="cff:no-stem-darkening=0 autofitter:no-stem-darkening=0"
```

## 99 - Sources used 

https://wiki.archlinux.org/

https://wiki.archlinux.org/title/User:Bai-Chiang/Arch_Linux_installation_with_unified_kernel_image_(UKI),_full_disk_encryption,_secure_boot,_btrfs_snapshots,_and_common_setups

https://wiki.archlinux.org/title/Libvirt

https://blog.aktsbot.in/no-more-blurry-fonts.html
