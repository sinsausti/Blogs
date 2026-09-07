---
title: "How to Build a Home Virtualization Lab with Proxmox VE"
description: "Install Proxmox VE and build a practical home virtualization lab with this concise step-by-step guide."
author: "Sebastian Insausti"
date: "2026-08-27"
tags: ["Infrastructure", "Linux"]
canonical_url: "https://insaustis.com/blog/proxmox-home-virtualization-lab.html"
---

# How to Build a Home Virtualization Lab with Proxmox VE

Proxmox Virtual Environment (VE) turns a spare PC or small server into a web-managed virtualization host. It runs virtual machines with KVM and Linux containers with LXC, making it a practical foundation for a home lab.

> Important: installing Proxmox VE erases the selected system disk. Back up anything you need and disconnect disks that should not be touched.

## 1. Check the Hardware

Use a 64-bit x86 system with Intel VT-x or AMD-V enabled in the BIOS/UEFI. For a useful small lab, aim for at least 8 GB of RAM, an SSD, and a wired Ethernet connection. Assign the host a static IP or reserve its DHCP address in your router.

## 2. Download and Verify Proxmox VE

Download the current ISO from the [official Proxmox VE download page](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso). Verify the file before writing it to USB:

```
# Linux
sha256sum proxmox-ve_*.iso

# macOS
shasum -a 256 proxmox-ve_*.iso
```

Compare the result with the SHA-256 value published beside the download.

## 3. Create the Installer USB

On Linux, identify the USB device and write the hybrid ISO directly to it:

```
lsblk

# Replace /dev/sdX with the whole USB device, not a partition.
# This command permanently erases that device.
sudo umount /dev/sdX* 2>/dev/null || true
sudo dd bs=1M conv=fdatasync \
  if=./proxmox-ve_*.iso of=/dev/sdX
```

Double-check the output device before running `dd`. Proxmox explicitly warns that tools such as UNetbootin do not work with its installer image.

## 4. Install Proxmox VE

1. Boot the future host from the USB drive.
2. Select **Install Proxmox VE (Graphical)**.
3. Choose the dedicated system disk and accept the license.
4. Set the country, time zone, root password, and administrator email.
5. Select the wired network interface and configure a static IP, gateway, DNS server, and hostname such as `pve.home.arpa`.
6. Review the summary, install, remove the USB drive, and reboot.

## 5. Log In and Update the Host

From another computer, open `https://PROXMOX-IP:8006` and sign in as `root` using the Linux PAM realm. The initial certificate warning is expected because the host uses a self-signed certificate.

For a non-production home lab without a subscription, open **Node → Updates → Repositories**, disable the enterprise repository, and add **No-Subscription**. Then update from the shell:

```
apt update
apt full-upgrade
pveversion -v
ip -brief address
```

These commands run in the Proxmox root shell. Review the proposed package changes before confirming, and reboot if the update installed a new kernel:

```
reboot
```

## 6. Create the First Virtual Machine

1. Download a guest ISO to your computer.
2. In Proxmox, select **local → ISO Images → Upload**.
3. Click **Create VM**, select the ISO, and assign CPU, memory, storage, and the default `vmbr0` bridge.
4. Start the VM, open its console, and install the guest operating system.

You can confirm the new VM from the host shell:

```
qm list
qm status 100
qm config 100
```

Replace `100` with the VM ID shown in the web interface.

## Next Steps

Create a separate backup target before the lab becomes important, keep the Proxmox host off the public Internet, and avoid installing general-purpose applications directly on the hypervisor. Run services inside VMs or containers instead.

---

Building a home lab and need help with the network or storage layout? [Get in touch.](https://insaustis.com/#contact)
