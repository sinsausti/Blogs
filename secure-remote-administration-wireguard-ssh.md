---
title: "Secure Remote Administration with WireGuard and SSH"
description: "Build a private WireGuard management network and restrict SSH administration to key-based access through the VPN."
author: "Sebastian Insausti"
date: "2026-08-21"
tags: ["Linux", "Networking"]
canonical_url: "https://insaustis.com/blog/secure-remote-administration-wireguard-ssh.html"
---

# Secure Remote Administration with WireGuard and SSH

Exposing SSH directly to the Internet increases the attack surface of every managed server. A simple alternative is to place administration on a private WireGuard network and permit SSH only through that path.

> Keep console or provider recovery access available while changing VPN, firewall, or SSH settings. Test a second session before closing the original connection.

## 1. Define the Management Network

This example uses a split tunnel: only `10.20.0.0/24` travels through WireGuard. The server is `10.20.0.1`, the administrator laptop is `10.20.0.2`, and WireGuard listens on `51820/UDP`.

Open the WireGuard UDP port on the upstream firewall or router. Do not expose TCP port 22 publicly once VPN access is proven.

## 2. Install WireGuard and Generate Keys

On Debian or Ubuntu, install WireGuard on the server and client:

```
sudo apt update
sudo apt install wireguard
```

Generate each peer's key pair locally. Never send or copy a private key to the other peer:

```
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

Move the server keys to a root-only location such as `/etc/wireguard/`. Keep the laptop private key only on the laptop.

## 3. Configure the Server

Create `/etc/wireguard/wg0.conf` with permissions `600`:

```
[Interface]
Address = 10.20.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

[Peer]
PublicKey = LAPTOP_PUBLIC_KEY
AllowedIPs = 10.20.0.2/32
```

On the server, `AllowedIPs` associates the laptop peer with its single VPN address. Give every administrator a separate peer and address so access can be revoked individually.

```
sudo chmod 600 /etc/wireguard/wg0.conf
sudo systemctl enable --now wg-quick@wg0
sudo wg show
```

## 4. Configure the Laptop

```
[Interface]
Address = 10.20.0.2/32
PrivateKey = LAPTOP_PRIVATE_KEY

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 10.20.0.0/24
PersistentKeepalive = 25
```

`PersistentKeepalive` is useful when the client is behind NAT and must remain reachable after idle periods; it is not required for every peer. Start the interface and verify the handshake:

```
sudo wg-quick up wg0
sudo wg show
ping -c 3 10.20.0.1
ssh admin@10.20.0.1
```

## 5. Harden SSH Without Locking Yourself Out

Create a non-root administrative account and install its public key in `~/.ssh/authorized_keys`. Confirm key-based login through WireGuard in a second terminal. Then add a drop-in such as `/etc/ssh/sshd_config.d/60-remote-admin.conf`:

```
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
AllowUsers admin
```

If SSH is connected to centralized identity or MFA through keyboard-interactive authentication, do not disable that method without adapting the design. Validate before reloading:

```
sudo sshd -t
sudo systemctl reload ssh
```

The service is commonly named `ssh` on Debian/Ubuntu and `sshd` on other distributions.

## 6. Enforce the Network Boundary

Configure the host and upstream firewalls to allow `51820/UDP` from the Internet, allow `22/TCP` only from the WireGuard interface or management subnet, and deny public SSH. Remove any older broad allow rule only after the VPN path is tested.

```
# Confirm the services and their listening addresses
sudo ss -lunp | grep ':51820'
sudo ss -lntp | grep ':22'

# Confirm the effective SSH configuration
sudo sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication|permitrootlogin|pubkeyauthentication'
```

WireGuard authenticates devices through keys, but it does not replace SSH accounts, authorization, MFA, logging, patching, or secret management.

## 7. Operate It Safely

- Use one WireGuard key pair per device; never share private keys.
- Remove lost or retired peers immediately and rotate exposed keys.
- Back up the configuration securely, but protect private keys as credentials.
- Monitor WireGuard handshakes and SSH authentication logs.
- Test emergency console access and document the recovery path.
- Use a bastion or SSH certificates when the environment grows beyond a few administrators.

## References

Primary documentation: the [WireGuard Quick Start](https://www.wireguard.com/quickstart/), [`wg-quick(8)`](https://man7.org/linux/man-pages/man8/wg-quick.8.html), and the [`sshd_config(5)` manual](https://man.openbsd.org/sshd_config).
