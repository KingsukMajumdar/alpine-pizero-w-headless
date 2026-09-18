# Security Notes

## macmpi Bootstrap Overlay

The [macmpi headless bootstrap overlay](https://github.com/macmpi/alpine-linux-headless-bootstrap) ships with SSH host keys that are publicly visible in the GitHub repository. These are **temporary bootstrap keys only**.

Per the macmpi documentation, these temporary keys live in RAM (`/tmp`) and are discarded after the actual install and reboot. Alpine/OpenSSH generates fresh unique host keys in `/etc/ssh/` once the openssh package is properly installed.

Verify fresh host keys exist after setup:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

The macmpi overlay is deleted from the SD card at Step 20 of this guide. After that point the device uses only its own privately generated keys -- none shared with anyone.

## WiFi PSK Security

`wpa_passphrase` uses PBKDF2-SHA1 to hash the WiFi password. The output hash is stored in `wpa_supplicant.conf`. While safer than plain text, weak passwords can be cracked offline using tools like hashcat if an attacker gains access to the SD card.

Use strong WiFi passwords -- minimum 12 characters, mixed case, numbers, and symbols.

For production or research deployments -- use a dedicated IoT VLAN isolated from your main network.

## SSH Key Security

The private key (`id_ed25519`) never leaves the host machine. The Pi Zero W holds only the public key in `authorized_keys`. Even if the Pi Zero W is fully compromised, an attacker cannot derive the private key from the public key.

Add a passphrase to your SSH private key for an additional layer of protection:

```bash
ssh-keygen -p -f ~/.ssh/id_ed25519
```

## Firewall

The awall policy in this guide uses default-deny inbound. Only SSH (port 22), Modbus TCP (port 502), and mDNS (UDP 5353) are explicitly allowed. All other inbound traffic is dropped.

SSH connections are rate-limited to 3 attempts per 60 seconds per source IP -- this limits brute-force attempts even if password authentication were enabled.

## Reporting Issues

If you find a security issue in this guide -- open a GitHub issue or contact the maintainer directly.
