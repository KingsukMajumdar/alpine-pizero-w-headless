# Security Notes

## macmpi Bootstrap Overlay

The [macmpi headless bootstrap overlay](https://github.com/macmpi/alpine-linux-headless-bootstrap) ships with SSH host keys that are publicly visible in the GitHub repository. These are **temporary bootstrap keys only**.

Alpine automatically regenerates fresh unique SSH host keys the first time `sshd` restarts after installation. This happens at Step 17 of this guide. The macmpi overlay is then deleted from the SD card at Step 20. At that point the device uses only its own privately generated keys.

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

## Reporting Issues

If you find a security issue in this guide -- open a GitHub issue or contact the maintainer directly.
