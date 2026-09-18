# 🐧 Alpine Linux Headless Setup
## Raspberry Pi Zero W v1.1 &nbsp;|&nbsp; 512MB SD Card &nbsp;|&nbsp; No Keyboard &nbsp;|&nbsp; No Display &nbsp;|&nbsp; No Serial Cable

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Alpine Linux](https://img.shields.io/badge/Alpine_Linux-3.24.2-0D597F?logo=alpinelinux&logoColor=white)](https://alpinelinux.org)
[![Raspberry Pi](https://img.shields.io/badge/Raspberry_Pi-Zero_W_v1.1-C51A4A?logo=raspberrypi&logoColor=white)](https://www.raspberrypi.com/products/raspberry-pi-zero-w/)
[![Architecture](https://img.shields.io/badge/Architecture-ARMv6_armhf-green)](https://wiki.alpinelinux.org/wiki/Raspberry_Pi)
[![Tested](https://img.shields.io/badge/Tested-2026--09--18-brightgreen)](README.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-Welcome-brightgreen.svg)](https://github.com/KingsukMajumdar/alpine-pizero-w-headless/pulls)

> **A complete, tested, community-ready guide for running Alpine Linux headless on a Raspberry Pi Zero W v1.1 with a 512MB SD card.**
> No keyboard. No display. No serial cable. Just WiFi, SSH, and a properly hardened minimal Linux node.

---

## 📁 Repository Contents

| File / Folder | Role |
|---|---|
| [`README.md`](README.md) | Complete step-by-step setup guide -- start here |
| [`SECURITY.md`](SECURITY.md) | Security notes on bootstrap keys, WiFi PSK, and SSH hardening |
| [`LICENSE`](LICENSE) | MIT License |
| [`.gitignore`](.gitignore) | Prevents accidental commit of real credentials or keys |
| [`configs/wpa_supplicant.conf.example`](configs/wpa_supplicant.conf.example) | WiFi configuration template -- copy and fill in your credentials |
| [`configs/pizw-firewall.json.example`](configs/pizw-firewall.json.example) | awall firewall policy -- SSH + Modbus TCP + mDNS |
| [`configs/ssh_config.example`](configs/ssh_config.example) | Host machine SSH config -- enables `ssh pizw` one-command connect |

---

## ⚡ Quick Summary

| Property | Value |
|---|---|
| Board | Raspberry Pi Zero W v1.1 |
| SoC | BCM2835 -- ARMv6 single-core @ 1 GHz |
| Alpine Build | `armhf` (ARMv6 hard-float) -- only supported build that boots on Pi Zero W v1.1 |
| Alpine Version | 3.24.2 |
| Boot Mode | Diskless -- OS runs entirely in RAM |
| SD Card | 512MB minimum |
| Network | WiFi only (CYW43438, 2.4GHz) -- no Ethernet |
| SSH | Key-based auth only (password login disabled after hardening) |
| Firewall | awall + iptables -- default deny inbound |
| Hostname Resolution | Avahi mDNS -- `pizw.local` |
| Connect | `ssh pizw` (one command on the same LAN with mDNS) |

---

## 🚀 Why This Guide Exists

Setting up Alpine Linux headless on a Pi Zero W v1.1 is harder than it looks:

- **ARMv6 is a dead end for most distros** -- Raspberry Pi OS Bookworm dropped it. Alpine still supports it.
- **512MB SD card means diskless mode is the only practical full-OS option** -- Alpine diskless is the only well-supported choice at this storage size.
- **No Ethernet, no display, no serial console used** -- WiFi is the only path in. But Alpine needs `setup-alpine` to configure WiFi, which needs SSH, which needs WiFi. Circular dependency.
- **The solution is not obvious** -- the macmpi overlay + exact `wpa_supplicant.conf` format on SD root, documented in the Alpine Wiki but not well-known.

This guide documents **what actually works**, learned from a 7-hour live session on real hardware in September 2026. Every failure mode, every trap, every fix is documented.

---

## 📚 References

This guide builds on and links to official documentation:

- [Alpine Linux Wiki -- Raspberry Pi](https://wiki.alpinelinux.org/wiki/Raspberry_Pi)
- [Alpine Linux Wiki -- Installation on a Headless Host](https://wiki.alpinelinux.org/wiki/Installation_on_a_headless_host)
- [Alpine Linux Wiki -- Alpine Local Backup (lbu)](https://wiki.alpinelinux.org/wiki/Alpine_local_backup)
- [Alpine Linux Wiki -- Setting up a Firewall with awall](https://wiki.alpinelinux.org/wiki/Setting_up_a_firewall_with_awall)
- [Alpine Linux Wiki -- WiFi / wpa_supplicant](https://wiki.alpinelinux.org/wiki/Connecting_to_a_wireless_accesspoint)
- [macmpi Headless Bootstrap Overlay](https://github.com/macmpi/alpine-linux-headless-bootstrap)
- [Raspberry Pi -- config.txt Documentation](https://www.raspberrypi.com/documentation/computers/config_txt.html)
- [Raspberry Pi Zero W Product Page](https://www.raspberrypi.com/products/raspberry-pi-zero-w/)

---

## Hardware Requirements

| Item | Specification |
|---|---|
| Board | Raspberry Pi Zero W v1.1 |
| SoC | BCM2835, single-core ARMv6 @ 1 GHz |
| RAM | 512 MB (shared with GPU) |
| WiFi | CYW43438 -- 2.4GHz only, no Ethernet |
| SD Card | 512MB minimum (diskless mode) |
| Power | 5V 1A via PWR IN microUSB port |
| Host OS | Any Linux with SSH client and nmap |

---

## Why Alpine Linux on Pi Zero W v1.1?

Raspberry Pi OS Bookworm and later dropped ARMv6 support. Alpine Linux 3.24.x still supports ARMv6 via the `armhf` build and runs efficiently in diskless mode -- the entire OS loads into RAM at boot, leaving the SD card almost untouched. This is ideal for a 512MB card.

**Do NOT attempt on this board:** Raspberry Pi OS Bookworm/Trixie and Ubuntu have dropped ARMv6 support. Armbian and DietPi support matrices change over time -- check their current release pages before attempting. Alpine Linux 3.24.x `armhf` is confirmed working.

---

## Architecture Overview

```
[Raspberry Pi Zero W v1.1]
  Alpine Linux 3.24.2 armhf
  Diskless mode (RAM)
  SSH server (openssh)
  Firewall (awall + iptables)
  mDNS (Avahi)
  User: userpizw
  Hostname: pizw
  Accessible as: userpizw@pizw.local

         WiFi (2.4GHz)
              |
[Host Linux Machine]
  SSH client
  User: userh
  Hostname: hostalpy
  Connect: ssh pizw
```

---

## Important Concepts Before Starting

### Diskless Mode

Alpine diskless mode loads the OS fully into RAM at boot. The SD card is only read at boot time and when you explicitly run `lbu commit -d`. All changes live in RAM -- power loss without committing reverts everything.

### lbu (Alpine Local Backup)

`lbu commit -d` saves your configuration to the SD card. Run it after every change. Without it, nothing persists across reboot.

### apkovl Overlay

Alpine reads `*.apkovl.tar.gz` from SD card root during initramfs boot. This is how we inject WiFi configuration and SSH keys before the system comes up -- solving the headless bootstrap circular dependency.

### The Circular Dependency Problem

First boot headless setup has no escape without an overlay:

```
WiFi needs setup-alpine
setup-alpine needs SSH
SSH needs WiFi
```

The [macmpi headless bootstrap overlay](https://github.com/macmpi/alpine-linux-headless-bootstrap) solves this. It is referenced in the [Alpine Linux Wiki headless installation page](https://wiki.alpinelinux.org/wiki/Installation_on_a_headless_host) as the community-standard overlay for headless Alpine setup.

---

## 🖥️ Part A: Prepare SD Card on Host Machine

### Step 1: Fix USB Autosuspend (Linux hosts with Ryzen/USB ACPI issues)

Some Linux machines suspend USB devices aggressively. This causes write failures mid-flash.

```bash
sudo bash -c 'echo -1 > /sys/module/usbcore/parameters/autosuspend'
for f in /sys/bus/usb/devices/*/power/control; do
    echo "on" | sudo tee $f > /dev/null
done

# Verify -- must return -1
cat /sys/module/usbcore/parameters/autosuspend
```

> This resets on reboot -- run it every session before SD card operations.

Insert SD card into a rear USB port directly -- never through a hub.

---

### Step 2: Identify SD Card

```bash
lsblk -o NAME,SIZE,TRAN,MODEL,FSTYPE,LABEL
```

A 512MB card appears as approximately 483.9M due to partition overhead.

```
NAME     SIZE   TRAN  MODEL
sda      483.9M usb   STORAGE DEVICE
```

> **Confirm the size matches your SD card before writing anything.**
> `sda` is an example -- your system may show `sdb` or `sdc`.
> If SIZE shows `0B` -- card not seated. Remove, reseat firmly, retry.

---

### Step 3: Format SD Card

#### 3.1 Unmount if auto-mounted

```bash
sudo umount /dev/sda1 2>/dev/null
sudo umount /dev/sda  2>/dev/null
lsblk -o NAME,MOUNTPOINT | grep sda
# MOUNTPOINT column must be empty
```

#### 3.2 Wipe existing partition table

```bash
sudo wipefs --all /dev/sda
sudo dd if=/dev/zero of=/dev/sda bs=512 count=2048 status=progress
sync
```

#### 3.3 Create MBR partition table and FAT32 partition

```bash
sudo fdisk /dev/sda
```

Inside fdisk -- type each character and press Enter:

```
o       # create new MBR partition table
n       # new partition
p       # primary
1       # partition number 1
Enter   # accept default first sector (2048)
Enter   # accept default last sector (use all space)
t       # change partition type
b       # W95 FAT32
a       # set bootable flag
w       # write changes and exit
```

#### 3.4 Format as FAT32

```bash
sudo mkfs.vfat -F 32 -n ALPINE /dev/sda1
```

#### 3.5 Verify

```bash
sudo fsck.vfat -v /dev/sda1
# Must end with: /dev/sda1: 1 files, 1/123362 clusters -- no errors

lsblk -o NAME,SIZE,FSTYPE,LABEL /dev/sda
# sda1   482.9M   vfat   ALPINE
```

---

### Step 4: Download Alpine Linux armhf

> **Only `armhf` runs on Pi Zero W v1.1.**
> `aarch64` = 64-bit ARM -- does not boot on ARMv6.
> `armv7` = requires ARMv7 minimum -- does not boot on ARMv6.
> See [Alpine Downloads](https://alpinelinux.org/downloads/) -- Raspberry Pi section.

```bash
mkdir -p ~/Downloads/alpine-pizerow
cd ~/Downloads/alpine-pizerow

# Replace 3.24.2 with the latest version from https://alpinelinux.org/downloads/
wget https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/armhf/alpine-rpi-3.24.2-armhf.tar.gz
wget https://dl-cdn.alpinelinux.org/alpine/v3.24/releases/armhf/alpine-rpi-3.24.2-armhf.tar.gz.sha256

# Verify checksum -- MUST show OK before proceeding
sha256sum -c alpine-rpi-3.24.2-armhf.tar.gz.sha256
# Expected: alpine-rpi-3.24.2-armhf.tar.gz: OK
```

> Never skip checksum verification -- a corrupted download causes random boot failures.

---

### Step 5: Write Alpine to SD Card

> Alpine for Raspberry Pi is a **tarball**, not a disk image. Use `tar`, not `dd`.

```bash
sudo mkdir -p /mnt/alpinesd
sudo mount /dev/sda1 /mnt/alpinesd

# Single line -- no line breaks
sudo tar -xzf ~/Downloads/alpine-pizerow/alpine-rpi-3.24.2-armhf.tar.gz -C /mnt/alpinesd/

# Confirm correct build -- this file must exist
ls /mnt/alpinesd/bcm2835-rpi-zero-w.dtb
```

`bcm2835-rpi-zero-w.dtb` confirms the Pi Zero W v1.1 BCM2835 build is correct.
If only `bcm2711-*.dtb` files are present -- wrong build downloaded.

> **Do not add WiFi modules to `cmdline.txt` on this Alpine armhf image.**
> Adding `brcmfmac` or `brcmutil` to the modules list broke boot during testing on Alpine 3.24.2 armhf.
> Alpine initramfs loads the WiFi driver automatically -- no `cmdline.txt` modification needed.
> Leave it exactly as extracted: `modules=loop,squashfs,sd-mod,usb-storage quiet console=tty1`

---

### Step 6: Add Minimal `usercfg.txt`

```bash
sudo bash -c 'echo "gpu_mem=16" > /mnt/alpinesd/usercfg.txt'
```

`gpu_mem=16` gives 496MB RAM to the CPU -- maximum on a 512MB board.

> `dtoverlay=disable-bt` and LED control lines in `usercfg.txt` caused boot failures during testing -- do not add them until WiFi is confirmed working.

---

### Step 7: Download macmpi Headless Overlay

The [macmpi headless bootstrap](https://github.com/macmpi/alpine-linux-headless-bootstrap) is referenced in the [Alpine Linux Wiki headless installation page](https://wiki.alpinelinux.org/wiki/Installation_on_a_headless_host) as the community-standard overlay for headless Alpine setup. It is open source -- full source visible on GitHub.

```bash
# Download directly from GitHub
wget -O /tmp/headless.apkovl.tar.gz \
    https://github.com/macmpi/alpine-linux-headless-bootstrap/raw/main/headless.apkovl.tar.gz

sudo cp /tmp/headless.apkovl.tar.gz /mnt/alpinesd/
ls -lh /mnt/alpinesd/headless.apkovl.tar.gz
```

> **Security note on macmpi SSH host keys:**
> The overlay ships with bootstrap SSH host keys that are publicly visible in the GitHub repo.
> According to the [macmpi README](https://github.com/macmpi/alpine-linux-headless-bootstrap), these temporary keys live in RAM `/tmp` and are discarded once the system is rebooted after the actual install.
> After installing `openssh` and rebooting, Alpine/OpenSSH will have fresh host keys in `/etc/ssh/`.
> Verify with: `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`
> The overlay is deleted in Step 20 -- from that point on, the device uses only its own keys.
> The risk window is first boot only, on your private local network.

---

### Step 8: Create `wpa_supplicant.conf` on SD Card Root

> **Critical -- exact format required.**
> This file must be on the SD card **root** alongside the overlay tarball.
> macmpi reads it from SD root directly -- not from inside the overlay.
> Wrong format = WiFi fails silently = no SSH access.
> See [Alpine WiFi documentation](https://wiki.alpinelinux.org/wiki/Connecting_to_a_wireless_accesspoint).

```bash
sudo bash -c 'cat > /mnt/alpinesd/wpa_supplicant.conf << EOF
country=XX
network={
	key_mgmt=WPA-PSK
	ssid="YourSSID"
	psk="YourWiFiPassword"
}
EOF'
```

Replace `XX` with your two-letter country code (e.g. `IN`, `US`, `GB`).
Replace `YourSSID` and `YourWiFiPassword` with your actual WiFi credentials.

> **Recommended: use `wpa_passphrase` for a hashed PSK (no plain text password on SD card):**
>
> ```bash
> wpa_passphrase "YourSSID" 'YourWiFiPassword'
> ```
>
> Copy the 64-character `psk=` hash line. Use it WITHOUT quotes in the conf file:
>
> ```
> psk=a3f8c2d1e4b5a6f7...  (no quotes around hash)
> ```
>
> Use single quotes around the password in the `wpa_passphrase` command to prevent
> bash from interpreting special characters (`!`, `@`, `#` etc.).
>
> For interactive entry (password never visible in terminal or bash history):
>
> ```bash
> wpa_passphrase "YourSSID"
> # Press Enter -- type password -- press Enter again
> ```

> **Clear bash history after typing any plain text password:**
>
> ```bash
> for i in $(history | grep "wpa_passphrase" | awk '{print $1}' | sort -rn); do
>     history -d $i
> done
> history -w
> ```
>
> Then close the terminal -- scrollback buffer still shows the command.

---
> If you use plain text `psk="..."`, the password is stored in cleartext on the SD card.

### Step 9: Verify SD Card and Unmount

```bash
# Verify all required files
ls -lh /mnt/alpinesd/headless.apkovl.tar.gz
ls -lh /mnt/alpinesd/wpa_supplicant.conf
ls -lh /mnt/alpinesd/usercfg.txt
cat /mnt/alpinesd/cmdline.txt
# Must show: modules=loop,squashfs,sd-mod,usb-storage quiet console=tty1

# Safe unmount
sync && sudo umount /mnt/alpinesd
lsblk -o NAME,MOUNTPOINT | grep sda
# MOUNTPOINT column must be empty
```

---

## 🔌 Part B: First Boot

### Step 10: Boot Raspberry Pi Zero W

- Remove SD card from host reader
- Insert into Pi Zero W v1.1 -- SD slot on underside of board
- Connect power to **PWR IN** microUSB port -- closest port to SD card slot
- Wait **120 seconds minimum** -- single-core ARMv6 boots slowly

> **Power supply is critical.**
> PC USB port provides only 500mA -- may not be enough for Pi Zero W + WiFi initialisation.
> Use a phone charger: 5V 1A minimum, short thick cable.
> Undervoltage causes LED stable but WiFi hardware never initialises.

> **Normal boot LED sequence:**
> Rapid flash -- bootloader
> Irregular flash -- kernel + Alpine initramfs
> Slower flash -- overlay extraction, WiFi connect attempt
> Steady slow flash -- SSH ready

---

### Step 11: Find Pi Zero W IP Address

Host machine must be on the same WiFi network as Pi Zero W.

```bash
# Check your current subnet
ip addr show wlo1 | grep "inet "

# Scan your subnet -- replace with your actual subnet
nmap -sn 192.168.1.0/24

# Look for Raspberry Pi MAC prefix b8:27:eb
ip neigh show | grep "b8:27"
```

> Pi Zero W v1.1 MAC address always starts with `B8:27:EB` (Raspberry Pi Foundation OUI).

> If Pi appears in phone hotspot connected devices but not in nmap:
>
> ```bash
> sudo ip neigh flush all && nmap -sn 192.168.1.0/24
> ```

> **Verify hardware first if Pi Zero W never appears:**
> Flash Raspberry Pi OS Lite via Raspberry Pi Imager with WiFi pre-configured.
> If Pi OS connects -- hardware and WiFi chip are working.
> Problem is always Alpine WiFi bootstrap configuration, not hardware.

---

### Step 12: First SSH Connection

```bash
ssh root@<PiZeroW-IP>
# No password -- press Enter
```

Expected output:

```
Alpine Linux headless bootstrap v1.9 by macmpi
You may want to delete/rename .apkovl file before reboot...
Welcome to Alpine!
alpine-headless:~#
```

---

## ⚙️ Part C: Initial Configuration

### Step 13: Run `setup-alpine`

See [Alpine installation documentation](https://docs.alpinelinux.org/user-handbook/0.1a/Installing/setup_alpine.html).

```bash
setup-alpine
```

Recommended responses:

```
Hostname:              pizw
Interface:             wlan0        (only interface -- no Ethernet on Pi Zero W)
SSID:                  YourSSID
WiFi password:         YourWiFiPassword
IPv4:                  dhcp
IPv6:                  none
Manual network config: n
Root password:         <set a strong password>
Timezone:              Your/Timezone
Proxy:                 none
NTP:                   chrony
Mirror:                skip         (SSL error from clock skew -- fix after)
User:                  no
SSH server:            openssh
Allow root SSH:        yes          (temporary -- disabled after hardening)
SSH key:               none
Disk:                  n
Store configs:         mmcblk0p1
APK cache:             /media/mmcblk0p1/cache
```

> **Mirror shows SSL error on first boot -- this is normal.**
> Clock skew causes SSL certificate verification failure.
> Choose `skip`. Fix clock and set mirror in next step.

> **Disk -- always choose `n` then `mmcblk0p1`.**
> `n` = diskless mode -- OS runs in RAM.
> `mmcblk0p1` = save configs to SD card.
> All changes persist only after `lbu commit -d`.

---

### Step 14: Fix Clock and Set APK Mirror

```bash
# Fix clock
ntpd -d -q -n -p pool.ntp.org
date  # verify correct date and time

# Set APK repositories
cat > /etc/apk/repositories << 'EOF'
/media/mmcblk0p1/apks
https://dl-cdn.alpinelinux.org/alpine/v3.24/main
https://dl-cdn.alpinelinux.org/alpine/v3.24/community
EOF

apk update
```

---

### Step 15: Add Multiple WiFi Networks (Optional)

```bash
# Generate hashed PSK for each network
wpa_passphrase "YourSSID" 'YourWiFiPassword'
wpa_passphrase "YourSSID2" 'YourWiFiPassword2'

# Write complete wpa_supplicant.conf
cat > /etc/wpa_supplicant/wpa_supplicant.conf << 'EOF'
country=XX
network={
	key_mgmt=WPA-PSK
	ssid="YourSSID"
	psk=<64-char-psk-hash>
	priority=10
}

network={
	key_mgmt=WPA-PSK
	ssid="YourSSID2"
	psk=<64-char-psk-hash>
	priority=5
}
EOF
```

Higher `priority` = connects first. Auto-fallback to lower priority if primary unavailable.

---

## 🔐 Part D: Security Hardening

### Step 16: Create Non-Root User

```bash
adduser userpizw
addgroup userpizw wheel
```

`wheel` group allows `su -` to become root.
This is needed because `PermitRootLogin no` is set next.

---

### Step 17: Harden SSH Configuration

Alpine minimal has no `nano` -- use `sed`:

```bash
# Disable root login
sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Ensure password auth enabled (until key auth confirmed working)
sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config

# Add security settings
cat >> /etc/ssh/sshd_config << 'EOF'
AllowUsers userpizw
MaxAuthTries 6
ClientAliveInterval 120
ClientAliveCountMax 2
EOF

# Verify
grep -E "PermitRootLogin|PasswordAuthentication|AllowUsers|MaxAuthTries|ClientAlive" \
    /etc/ssh/sshd_config

# Restart SSH -- this generates fresh unique host keys
rc-service sshd restart
```

> **Fresh SSH host keys after setup.**
> After installing openssh via `setup-alpine` and removing the macmpi overlay, Alpine/OpenSSH generates new host keys in `/etc/ssh/` if they are absent or replaced.
> During testing, `rc-service sshd restart` output showed: `ssh-keygen: generating new host keys: RSA ECDSA ED25519`
> Verify with: `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`
> The macmpi temporary bootstrap keys (held in RAM `/tmp`) are discarded after reboot.
> The overlay is deleted from SD card in Step 20 -- at that point the device uses only its own keys.

> **`MaxAuthTries 6` not 3 during setup.**
> SSH clients try multiple keys from `~/.ssh/` automatically.
> With `MaxAuthTries 3` -- SSH hits the limit before trying the correct key.
> Reduce to 3 after adding `IdentitiesOnly yes` to client SSH config (Step 27).

---

### Step 18: Install Avahi for mDNS Hostname Resolution

See [Alpine Avahi documentation](https://wiki.alpinelinux.org/wiki/Avahi).

```bash
apk add avahi dbus

# D-Bus must start before Avahi
rc-service dbus start
rc-update add dbus boot

# Start Avahi
rc-service avahi-daemon start
rc-update add avahi-daemon default

# Verify
rc-service avahi-daemon status
# Must show: status: started
```

After this step -- connect via `userpizw@pizw.local` from any host on the same LAN.

---

### Step 19: Set Correct Hostname

```bash
echo "pizw" > /etc/hostname
hostname pizw
hostname  # verify
```

---

### Step 20: Remove macmpi Overlay from SD Card

```bash
mount -o remount,rw /media/mmcblk0p1
rm /media/mmcblk0p1/headless.apkovl.tar.gz
ls /media/mmcblk0p1/headless.apkovl.tar.gz 2>/dev/null || echo "Removed"
```

Bootstrap overlay removed. The macmpi public keys are gone. SSH now uses the fresh unique keys generated in Step 17.

---

### Step 21: Add SSH Public Key from Host Machine

**On host machine -- open a second terminal:**

```bash
# Copy your public key to Pi Zero W
ssh-copy-id -i ~/.ssh/id_ed25519.pub userpizw@<PiZeroW-IP>
# Enter userpizw password when prompted
```

If `Too many authentication failures` error:

```bash
ssh-copy-id -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519.pub userpizw@<PiZeroW-IP>
```

**Alternative -- add key manually on Pi Zero W as root:**

```bash
mkdir -p /home/userpizw/.ssh
chmod 700 /home/userpizw/.ssh

# Paste your public key from: cat ~/.ssh/id_ed25519.pub on host machine
cat > /home/userpizw/.ssh/authorized_keys << 'EOF'
<paste-your-public-key-here>
EOF

chmod 600 /home/userpizw/.ssh/authorized_keys
chown -R userpizw:userpizw /home/userpizw
```

---

### Step 22: Configure `lbu` to Save `/home`

> **Critical -- without this, `authorized_keys` and home directory are lost every reboot.**
> `lbu commit` does not save `/home` by default.

```bash
echo "/home" >> /etc/lbu/include

# Verify lbu can see home directory
lbu list | grep home
# Must show: home/userpizw/, home/userpizw/.ssh/, home/userpizw/.ssh/authorized_keys
# If empty -- create the files first (Step 21), then check again
```

---

### Step 23: Install and Configure Firewall (awall + iptables)

See [Alpine awall documentation](https://wiki.alpinelinux.org/wiki/Setting_up_a_firewall_with_awall).

```bash
apk add awall iptables ip6tables
rc-update add iptables boot
rc-update add ip6tables boot
```

Create firewall policy:

```bash
cat > /etc/awall/optional/pizw-firewall.json << 'EOF'
{
  "description": "Pi Zero W Base Firewall",

  "zone": {
    "local": {
      "iface": "wlan0"
    }
  },

  "service": {
    "modbus": [
      { "proto": "tcp", "port": 502 }
    ],
    "mdns": [
      { "proto": "udp", "port": 5353 }
    ]
  },

  "policy": [
    { "in": "local", "action": "drop" },
    { "action": "accept" }
  ],

  "filter": [
    {
      "in": "local",
      "service": "ssh",
      "action": "accept",
      "conn-limit": { "count": 3, "interval": 60 }
    },
    {
      "in": "local",
      "service": "modbus",
      "action": "accept"
    },
    {
      "in": "local",
      "service": "mdns",
      "action": "accept"
    }
  ]
}
EOF
```

> **mDNS rule (UDP port 5353) is mandatory.**
> Without it -- `pizw.local` stops resolving. Avahi uses multicast UDP 5353.
> Verified in testing -- SSH via `.local` fails without this rule.
> Customize port 502 (Modbus TCP) -- replace with whichever ports your project needs.

Enable and activate:

```bash
awall enable pizw-firewall
awall list
# Must show: pizw-firewall  enabled  Pi Zero W Base Firewall

awall activate
# Press Enter when prompted

/etc/init.d/iptables save
/etc/init.d/ip6tables save

# Verify rules
iptables -L -n --line-numbers
```

> **To add more ports later (e.g. MQTT port 1883):**
> Add to `service` block: `"mqtt": [ { "proto": "tcp", "port": 1883 } ]`
> Add to `filter` block: `{ "in": "local", "service": "mqtt", "action": "accept" }`
> Then `awall activate` and `lbu commit -d`.

---

### Step 24: Commit Everything and Reboot

```bash
lbu commit -d
reboot
```

> **`lbu commit -d` order matters.**
> SSH keys, home directory, and all config changes must be committed BEFORE reboot.
> `PermitRootLogin no` is active -- if keys are not committed, locked out after reboot.
> Sequence: keys ready --> `lbu commit -d` --> reboot.

---

## ✅ Part E: Post-Reboot Verification

### Step 25: Verify SSH Connection

```bash
ssh -4 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes userpizw@pizw.local
# Enter key passphrase if set
```

Expected:

```
Welcome to Alpine!
pizw:~$
```

Verify:

```bash
whoami     # userpizw
hostname   # pizw
echo $HOME # /home/userpizw
```
### Step 26: Disable SSH Password Authentication

> **Only do this after Step 25 confirmed key-based login works.**
> If you disable password auth before your key is confirmed, you may lock yourself out.

**Open a new SSH session and become root:**

```bash
ssh -4 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes userpizw@pizw.local
su -
```
Then:
```bash
sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
grep -E 'PasswordAuthentication|PermitRootLogin|AllowUsers' /etc/ssh/sshd_config
# Must show:
#   PermitRootLogin no
#   PasswordAuthentication no
#   AllowUsers userpizw

# Optional: reduce MaxAuthTries now that client uses IdentitiesOnly yes
sed -i 's/^MaxAuthTries 6/MaxAuthTries 3/' /etc/ssh/sshd_config

rc-service sshd restart
lbu commit -d
```
From this point on: key-based auth only. No password login.
Test a fresh login before closing your current session:
```bash
# In a new terminal on host
ssh -4 -i ~/.ssh/id_ed25519 -o IdentitiesOnly=yes userpizw@pizw.local
```
---


### Step 27: Configure SSH Alias on Host Machine

```bash
nano ~/.ssh/config
```

Add:

```
Host pizw
    HostName pizw.local
    User userpizw
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    AddressFamily inet
```

From now on -- connect with:

```bash
ssh pizw
```

> `IdentitiesOnly yes` -- use only the specified key -- prevents MaxAuthTries failures.
> `AddressFamily inet` -- force IPv4 -- prevents IPv6 resolution issues with `.local`.

---

## 🛡️ Security Summary

### Measures Implemented

| Security Measure | Implementation | Purpose |
|---|---|---|
| Root login disabled | `PermitRootLogin no` | Root SSH is the primary attack vector |
| Non-root user | `userpizw` with wheel group | Principle of least privilege |
| WiFi PSK hashed (recommended) | wpa_passphrase hash in Step 8 | Plain text password not on SD card (only if hashed form used) |
| SSH host keys replaced | openssh install + overlay removal + reboot -- verify with `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` | macmpi temporary bootstrap keys discarded after reboot |
| macmpi overlay deleted | Removed from SD card root | Public keys gone from device |
| `AllowUsers userpizw` | sshd_config | Only named user can SSH in |
| SSH rate limiting | `conn-limit: 3 per 60s` in awall | Blocks brute force attempts |
| Firewall default deny | awall policy | Only explicitly allowed ports open |
| Avahi mDNS | `pizw.local` | No IP address exposure needed |
| `IdentitiesOnly yes` | SSH client config | Prevents key enumeration |
| `/home` in lbu include | `/etc/lbu/include` | Keys and config persist reboots |
| WiFi PSK hashed | `wpa_passphrase` hash | Plain text password not on SD card |

### What the Firewall Allows

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| 22 | TCP | Inbound | SSH (rate limited: 3/60s) |
| 502 | TCP | Inbound | Modbus TCP (customise as needed) |
| 5353 | UDP | Inbound | Avahi mDNS `.local` resolution |
| All | Any | Outbound | apk updates, NTP, DHCP |
| All others | Any | Inbound | Dropped (default deny) |

---

## 📡 Daily Use

### Connect to Pi Zero W

```bash
ssh pizw
```

### Become Root on Pi Zero W

```bash
su -
```

### After Any Configuration Change

```bash
lbu commit -d
```

**Never skip this -- all changes live in RAM only.**

---

## 🔧 Useful Alpine Commands

```bash
# System info
whoami && hostname && uname -r
free -h && df -h /

# WiFi
ip addr show wlan0
iwconfig wlan0 power off    # prevent power management drops

# Package management
apk update && apk upgrade
apk add <package>
apk search <keyword>

# OpenRC services (Alpine does NOT use systemd)
rc-service <name> start|stop|restart|status
rc-update add <name> default    # enable at boot
rc-update del <name> default    # disable

# lbu
lbu commit -d    # save all changes to SD card
lbu status       # show uncommitted changes
lbu list         # list all committed files

# Firewall
awall list
awall activate   # re-apply after policy changes
iptables -L -n --line-numbers
```

---

## 🎯 Suitable Workloads

| Workload | Package | Notes |
|---|---|---|
| SSH gateway / jump host | openssh | Already installed |
| MQTT broker | mosquitto | Lightweight, excellent fit |
| Modbus TCP RTU | python3, py3-pip, pymodbus | Edge RTU for ICS/SCADA research |
| Sensor data logger | python3, sqlite | Script-based, no GUI |
| Lightweight web server | nginx | Static files only |
| DNS sinkhole | unbound or dnsmasq | Excellent fit |

Avoid on this hardware:

| Workload | Reason |
|---|---|
| Docker | ARMv6 not supported |
| Node.js v11+ | Dropped ARMv6 support |
| OpenPLC Runtime v3 | Requires Node.js -- broken on ARMv6 |
| Python scipy/numpy | RAM insufficient, compile time extreme |
| Any GUI | No display, 512MB RAM |

---

## 🩺 Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| SD card SIZE 0B | Card not seated | Reseat firmly, try different USB port |
| Write fails mid-way | USB autosuspend | Re-run Step 1 |
| Two LED blinks, stops | usercfg.txt conflict | Remove all lines except `gpu_mem=16` |
| LED stable, no WiFi | Wrong wpa_supplicant format | Use exact minimal format from Step 8 |
| LED stable, no WiFi | Hotspot on 5GHz | Force 2.4GHz -- Pi Zero W is 2.4GHz only |
| Pi not in nmap | ARP cache stale | `sudo ip neigh flush all` then rescan |
| Pi not in nmap | Different subnet | Host and Pi must be on same network |
| SSH: Too many auth failures | Multiple keys tried | Add `IdentitiesOnly yes` to SSH config |
| SSH: host key changed | sshd regenerated keys | `ssh-keygen -R <IP>` then reconnect |
| Home directory lost on reboot | `/home` not in lbu | `echo "/home" >> /etc/lbu/include` |
| authorized_keys lost on reboot | `/home` not committed | `lbu list` -- recreate and `lbu commit -d` |
| `.local` not resolving | Avahi not running | Check dbus first, then avahi-daemon |
| `.local` stops after firewall | mDNS port blocked | Add UDP 5353 to awall policy |
| SSH blocked after firewall | Rule missing | Check `iptables -L -n` -- `awall activate` |
| APK mirror SSL error | Clock skew | `ntpd -d -q -n -p pool.ntp.org` first |
| wpa_passphrase: event not found | `!` in password | Use single quotes: `'pass!word'` |
| Config lost after reboot | Forgot lbu commit | Always `lbu commit -d` before poweroff |
| Locked out after Step 26 | Key not confirmed before disabling password auth | Boot with overlay removed, re-add key, retry |

---

## 📓 Key Lessons Learned

| Finding | Detail |
|---|---|
| macmpi overlay is the correct method | Referenced in Alpine Wiki headless installation page / community-standard overlay |
| `wpa_supplicant.conf` must be on SD root | Not inside overlay -- macmpi reads SD root |
| Exact minimal format required | `country=XX` + `network{}` with `key_mgmt=WPA-PSK` |
| Do not add WiFi modules to `cmdline.txt` | Broke boot on Alpine 3.24.2 armhf during testing -- environment-specific finding |
| macmpi SSH keys discarded after reboot | Temporary keys live in RAM `/tmp` -- verify fresh keys in `/etc/ssh/` after setup |
| `/home` not saved by lbu default | Must add to `/etc/lbu/include` explicitly |
| Clock skew breaks APK mirror | Fix clock before setting mirror |
| MaxAuthTries 3 too low during setup | Set 6 until `IdentitiesOnly yes` in client config |
| IPv6 causes SSH issues | `AddressFamily inet` in SSH config |
| mDNS needs firewall rule | UDP 5353 must be open for `.local` resolution |
| Verify hardware with Pi OS first | Pi OS Lite via Imager confirms hardware working |
| Disable password auth only after key confirmed | Prevents lockout -- do Step 26 only after Step 25 succeeds |

---
## 🧪 Test Environment

This guide was validated on the following setup:

| Component | Value |
|---|---|
| Board | Raspberry Pi Zero W v1.1 (BCM2835, ARMv6) |
| Alpine Version | 3.24.2 |
| Alpine Build | `armhf` |
| Kernel | Alpine `linux-rpi` (armhf) |
| SD Card | 512MB microSD (FAT32, label `ALPINE`) |
| Boot Mode | Diskless (`lbu commit` saves to `mmcblk0p1`) |
| Host OS | Linux x86_64 with `openssh-client` and `nmap` |
| Network | 2.4GHz WiFi (WPA2-PSK), same LAN as host |
| SSH Client | OpenSSH with ed25519 key |
| Session | ~7 hours live hardware bring-up, September 2026 |

**Verified after reboot:**

- SSH key-based login to `userpizw@pizw.local` works
- Password login is rejected (`Permission denied (publickey)`)
- `pizw.local` resolves via Avahi mDNS
- awall default-deny policy is active (`iptables -L -n` shows DROP policy)
- `/home/userpizw/.ssh/authorized_keys` persists across reboot
- Fresh SSH host keys in `/etc/ssh/` are used (not macmpi bootstrap keys)

**Known environment-specific findings (may not reproduce elsewhere):**

- Adding `brcmfmac`/`brcmutil` to `cmdline.txt` broke boot on Alpine 3.24.2 armhf
- `dtoverlay=disable-bt` in `usercfg.txt` caused boot failure before WiFi was up
- `MaxAuthTries 3` caused failures while `IdentitiesOnly yes` was not yet configured

---

## 🤝 Contributing

Issues and pull requests are welcome.

Please test on **actual Pi Zero W v1.1 hardware** before submitting a PR -- emulation and Pi 4 behave differently on ARMv6 edge cases.

If this guide saved you time -- a GitHub star helps others find it.

---

## 📜 License

MIT License -- free to use, modify, and distribute with attribution.
See [`LICENSE`](LICENSE) for full text.

---

## 🔗 Official References

| Resource | Link |
|---|---|
| Alpine Linux -- Raspberry Pi | https://wiki.alpinelinux.org/wiki/Raspberry_Pi |
| Alpine Linux -- Headless Installation | https://wiki.alpinelinux.org/wiki/Installation_on_a_headless_host |
| Alpine Linux -- Local Backup (lbu) | https://wiki.alpinelinux.org/wiki/Alpine_local_backup |
| Alpine Linux -- awall Firewall | https://wiki.alpinelinux.org/wiki/Setting_up_a_firewall_with_awall |
| Alpine Linux -- WiFi Setup | https://wiki.alpinelinux.org/wiki/Connecting_to_a_wireless_accesspoint |
| macmpi Headless Bootstrap | https://github.com/macmpi/alpine-linux-headless-bootstrap |
| Raspberry Pi -- config.txt | https://www.raspberrypi.com/documentation/computers/config_txt.html |
| Raspberry Pi Zero W | https://www.raspberrypi.com/products/raspberry-pi-zero-w/ |

---

<p align="center">
<i>Tested on Alpine Linux 3.24.2 armhf &nbsp;|&nbsp; Raspberry Pi Zero W v1.1 &nbsp;|&nbsp; 512MB SD card &nbsp;|&nbsp; September 2026</i>
</p>

---

## 📈 Version History

- **V4.0** (2026-09-18): Added Test Environment section, Version History, and consistency pass — password auth disabled after key confirmation, host key wording aligned with macmpi README, duplicate license/footer removed
- **V3.0** (2026-09-18): Added Step 26 to disable SSH password authentication; renumbered SSH alias step to 27; qualified overbroad claims ("any network", "only supported build", "no serial port")
- **V2.0** (2026-09-18): Security hardening pass — awall firewall policy, Avahi mDNS, `lbu` include for `/home`, key-based SSH auth, root login disabled
- **V1.0** (2026-09-18): Initial release — macmpi headless bootstrap overlay, `wpa_supplicant.conf` on SD root, `setup-alpine` diskless install, diskless mode + `lbu commit -d` workflow

---

<div align="center">

**⭐ Star this repository if it helped you! ⭐**

*Made with ❤️ for the Alpine Linux and Raspberry Pi community*

</div>
