# Arch-Installation Schritt-für-Schritt:
1. `loadkeys de-latin1` zum umstellen des Tastaturlayouts auf Deutsch
2. Verifiziere, dass wir im UEFI Modus gebootet haben via `ls /sys/firmware/efi/efivars`
	wenn der Befehl (ohne Fehler) eine Dateiliste ausgibt, dann haben wir korrekt gebooted
3. (Falls nötig) die font-size vergrößern: `setfont ter-132b` 
4. connect to the internet check interfaces with `ip link`
5. Zeitzone setzen (Zeitzonen kann man ansehen via `timedatectl list-timezones | grep Europe` ) und dann so setzen:	`timedatectl set-timezone ...`
6. `timedatects set-ntp true` um NTP Sync sicherzustellen
7. Partitionierung (via `lsblk` aktuelle Partitionen ansehen)
	- z.B. mittels `gdisk` partitionieren
	- Ggf. neue GPT Partitions-Tabelle anlegen
	- drei Partitionen:
		- 1: boot partition (ef00 - EFI System Partition) 1 GB (+1G)
		- ~~2: 16 GB Swap Partition~~ -> nutze stattdessen ein Swap File in dem verschlüsselten Luks Container (eigenes btrfs subvolume)
		- 3: Root (für LUKS container) (8300 - Linux Filesystem) restliche Plattenkapazität
8. Dateisysteme erzeugen, Swap aktivieren und LUKS Container erstellen:
	- 1: `mkfs.fat -F32 /dev/...`
	- ~~2: `mkswap /dev/...`~~
	- ~~2: `swapon /dev/...`~~ (später swapfile auf eingenem btrfs subvolume - NACH DER BASIS INSTALLATION!)
	- 3: Mittels `cryptsetup benchmark` kann man testen, ob sich gewisse Einstellungen besser eignen (CPU Beschleunigung für gewisse Ciphers z.B.)
	- 3: `cryptsetup --verbose --cipher=aes-xts-plain64 --key-size=512 --hash=sha512 --iter-time=5000 --type=luks2 luksFormat /dev/...`
	- 3: `cryptsetup luksOpen /dev/... rootpartition` zum mounten, bzw. besser `cryptsetup --allow-discards --persistent open /dev/sdaX rootpartition` um TRIM Support nicht unmöglich zu machen, siehe (https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD))
	- 3: BTRFS Dateisystem erstellen: `mkfs.btrfs /dev/mapper/rootpartition`
9. BTRFS Subvolumes erstellen:
	[[BTRFS Subvolumes Layout Recherche]]
	1. Zuerst mounten: `mount /dev/mapper/rootpartition /mnt`
	2. `cd /mnt`
	3. `btrfs subvolume create @`
 	4. (`btrfs subvolume create @swap`) später
	5. `btrfs subvolume create @home`
	6. `btrfs subvolume create @var_log` (für /var/log)
	7. `btrfs subvolume create @var_cache` (für /var/cache)
	8. (ignore)`btrfs subvolume create @var_lib_docker` (für /var/lib/docker)
	9. (ignore, snapper init erzeugt dann später subvolume)`btrfs subvolume for snapshots @.snapshots` (für /.snapshots)
	10. unmounten: `umount /mnt`
10. Mounten aller Subvolumens:
	1. `mount -o noatime,space_cache=v2,compress=zstd,ssd,discard=async,subvol=@ /dev/mapper/rootpartition /mnt`
	2. Erstellen der Ordner für die anderen Subvolumes + boot:
			`mkdir -p /mnt/{boot,home,var/log,var/cache}`
	3. (swap subvolume mounten nach /mnt/swap)
 	4. (swapfile erstellen: btrfs filesystem mkswapfile --size 16g --uuid clear /mnt/swap/swapfile)
	5. `mount -o noatime,space_cache=v2,compress=zstd,ssd,discard=async,subvol=@home /dev/mapper/rootpartition /mnt/home`
	6. `mount -o noatime,space_cache=v2,compress=zstd,ssd,discard=async,subvol=@var_log /dev/mapper/rootpartition /mnt/var/log`
	7. `mount -o noatime,space_cache=v2,compress=zstd,ssd,discard=async,subvol=@var_cache /dev/mapper/rootpartition /mnt/var/cache`
	8. ~~`mount -o noatime,space_cache=v2,compress=zstd,ssd,discard=async,subvol=@.snapshots /dev/mapper/rootpartition /mnt/.snapshots`~~ (wird später beim snapper konfigurieren erstellt
	9. Noch boot Partition mounten: `mount /dev/... /mnt/boot` 
11. Reflector mirror list anpassen, damit wir nur von den nähesten/schnellsten Mirrors Pakete laden: `reflector --country Germany -i "(\.netcologne\.de|\.uni\-|\.tu\-|\.hs\-|\.fu\-berlin|\.rz\.rub\.de|\.rwth\-aachen\.de|\.oth\-regensburg\.de|\.fau\.de|\.gwdg\.de)" --latest 20 --sort rate --save /etc/pacman.d/mirrorlist && pacman -Syy` 
12. mit pacstrap die grundlegenden Dateien installieren: `pacstrap -K /mnt base base-devel linux linux-firmware linux-headers linux-lts linux-lts-headers vim git networkmanager iwd` 
13. fstab erstellen lassen mit `genfstab -U /mnt >> /mnt/etc/fstab` 
14. chroot in unser neu erstelltes System: `arch-chroot /mnt` 
15. Localtime anpassen: `ln -sf /usr/share/zoneinfe/Europe/Berlin /etc/localtime`
	`hwclock --systohc`
16. Locals anpassen: in `/etc/locale.gen` die entsprechenden Zeilen auskommentieren, z.B.
    de_DE.UTF-8
    en_US.UTF-8
	dann mit `locale-gen` neu generieren.
17. Weitere Anpassungen:
	-  Datei erstellen `vim /etc/locale.conf` und Sprache einfügen: `LANG=en_US.UTF-8`
	- Genauso `vim /etc/vconsole.conf` und hinzufügen von `KEYMAP=de-latin1`
	- In `/etc/hostname` einen Hostname setzen
	- `vim /etc/hosts` folgende Zeile(n) hinzufügen
	```127.0.0.1 localhost ::1 localhost 127.0.1.1 archkiste.localdomain archkiste```
18. Root Passwort setzen: `passwd` 
19. User anlegen und Passwort setzen: `useradd -m -g users -G wheel jo && passwd jo`
20. Zusätzliche Pakete installieren: `pacman -S sudo`
21. `EDITOR=vim visudo` und folgende Zeile unkommentieren: `%wheel ALL=(ALL:ALL) ALL`, dadurch erhalten alle User der Gruppe wheel sudo Privilegien
22. Zusätzliche Pakete installieren:
	`pacman -S bash-completion dosfstools grub efibootmgr nano neovim mtools reflector rsync networkmanager os-prober btrfs-progs man-db man-pages`
23. Boot Optionen setzen: `vim /etc/mkinitcpio.conf` und Zeile `MODULES=()` in `MODULES=(btrfs)` und in Zeile `HOOKS=(...)` vor filesystems `encrypt` einfügen, bzw. laut Arch-wiki eigentlich `sd-encrypt` wenn systemd initramfs verwendet wird (siehe https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition).
	Danach `mkinitcpio -p linux` und danach noch `mkinitcpio -p linux-lts`
	
24. Grub Bootloader installieren: `grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`
25. Grub-Config erstellen: `grub-mkconfig -o /boot/grub/grub.cfg`
26. Via `blkid` die UUID der LUKS Partition (Crypto_LUKS) holen (kopieren) und `vim /etc/default/grub` editieren. Dort in Zeile `GRUB_CMDLINE_LINUX_DEFAULT="...` (oder ohne _DEFAULT) folgendes hinten anfügen:
	`cryptdevice=UUID="UUID-hier-einfügen":rootpartition root=/dev/mapper/rootpartition`.
Falls sd-encrypt Hook in mkinitcpio.conf gesetzt ist heißt es stattdessen:
`rd.luks.name=UUID-hier-einfügen=rootpartition root=/dev/mapper/rootpartition`
 Danach `grub-mkconfig -o /boot/grub/grub.cfg`
	Zusätzlich die Zeile `GRUB_DISABLE_OS_PROBER=false` auskommentieren
28. Noch einige Services enablen: 
		`systemctl enable NetworkManager` (für Netzwerkverbindungen + DHCP Client)
		~~`systemctl enable iwd.service` (für WLAN - in Verbindung mit NetworkManager)~~ iwd service nicht starten! Wird gleich als Backend für NetworkManager konfiguriert.
		`systemctl enable systemd-resolved` (für dns auflösung)
		`systemctl enable cups` (not installed yet!)
		`systemctl enable fstrim.timer` (zuerst sicherstellen, dass meine SSD trim unterstützt! siehe https://wiki.archlinux.org/title/Solid_state_drive)
29. iwd als NetworkManager Backend konfigurieren (https://wiki.archlinux.org/title/NetworkManager#Using_iwd_as_the_Wi-Fi_backend)
30. pacman -S amd-ucode (oder intel-ucode) zum patchen der CPU firmware
31. Für neues Lenovo Notebook noch installieren: sof-firmware mesa vulkan-intel
30. Reboot
31. Snapper installieren: `pacman -S snapper` und initialisieren
	`snapper -c root create-config /`
32. Ersten Snapper Snapshot machen: `snapper -c root create --description Erster Snapshot`
33. Snapper so konfiguriert, dass automatisch ein stündlicher Snapshot gemacht wird (sicherstellen, dass systemd timer `snapper-timeline.timer` läuft und folgende Retention-Policy konfiguriert
    In: /etc/snapper/configs/config
	TIMELINE_MIN_AGE="1800"
	TIMELINE_LIMIT_HOURLY="5"
	TIMELINE_LIMIT_DAILY="3"
	TIMELINE_LIMIT_WEEKLY="0"
	TIMELINE_LIMIT_MONTHLY="2"
	TIMELINE_LIMIT_YEARLY="0"
ODER: `snap-pac` installieren, und snapper-timeline snapshots ausschalten. Snap-pac macht bei jedem Pacman Befehl (oder zumindest denen, die das System verändern) ein snapshot.
35. Die Reihenfolge der zu bootenden kernel (wenn z.B. noch lts kernel installiert ist) kann so geändert werden (https://wiki.archlinux.org/title/GRUB/Tips_and_tricks#Changing_the_default_menu_entry)
    - in /etc/default/grub `GRUB_DISABLE_SUBMENU=y` auskommentieren
    - und dann unter `GRUB_DEFAULT=` auf die Zahl setzen (z.B. 1)
35. WindowManager/DE installieren:
    1. Folgende Pakete installieren:
       - wayland niri kitty waybar xwayland-sattelite rofi swaybg swaylock
       - in ~/.config/niri/config.kdl folgende Änderungen machen:
         - Unter keyboard `layout de` einfügen
         - Die Applikationen anpassen, z.B. kitty anstall alacritty als Terminal-Emulator, oder Rofi statt fuzzel als app launcher
36. Noch Swapfile erstellen und aktivieren.
37. Weitere Pakete installieren:
	- bash-completion
 	- firefox
  	- bluez und bluez-utils (bluetooth) pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber (alle für Audio) xdg-user-dirs (user ordner gemäß spezifikation)
  	- reflector (um mirrorliste automatisch zu erstellen) 
    - ttf-jetbrains-mono-nerd ttf-firacode-nerd (fonts)
    - Position der Bildschirme in Niri anpassen (Laptop links mit x=-1920 bei scale 1 für beide, ext. Monitor x=0 y=0)
    - wlsunset (augenfreundliche farbtöne in den Abendstunden) (mit den Argumenten -l 50.1, -L 8.7 -L (l = latitude, L = Longitude), ist ungefähr Frankfurt)
      Dann auch in niri config zum start hinzufügen
    - evolution
    - syncthing installieren
    - qt5-wayland und qt6-wayland
    - kitty statt alacritty als terminal emulator
    - keepassxc
    - obsidian installieren (achtung wayland support muss aktiv eingeschaltet werden)
    - kwallet als keyring (aus kde) (achtung ohne kwalletmanager und noch zusätzlich kwallet-pam + das noch konfigurieren)
	- spotify-launcher
  	- discord
38. Bildschirmhintergrund mittels Eintrag in niri config (und installiertem swaybg) setzen:
    `spawn-sh-at-startup "swaybg -i ~/Downloads/wallpaper_japan_wave_01.jpg -m center"`
