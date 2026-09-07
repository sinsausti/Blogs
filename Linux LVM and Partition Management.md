# Linux LVM and Partition Management: Your Storage Flexibility Toolkit

> Storage commands can destroy data. Take a tested backup, capture the existing partition table, confirm device names with `lsblk -f`, and have recovery access before making changes. Prefer `growpart` or a partition-aware management tool when available. Deleting and recreating a partition is only safe when the starting sector remains exactly unchanged.

## What's All This About?

Managing storage in Linux can feel like a puzzle sometimes, but once you understand LVM (Logical Volume Manager) and basic partition resizing, you'll wonder how you ever lived without these tools! Think of LVM as a flexible storage layer that sits between your physical disks and your filesystems, giving you the power to resize, move, and manage storage like a pro.

Today we'll cover two essential skills: resizing regular partitions and working with LVM. Whether you're running out of space or planning for future growth, these techniques will save your day!

## Part 1: Resizing Regular Partitions

Sometimes you just need to make a partition bigger. Here's how to do it safely:

### Step 1: Modify the Partition Table

**⚠️ Warning:** Always backup your data first! Partition operations can be risky.

```bash
sudo fdisk /dev/sda
```

In fdisk, follow these steps:

```
Command (m for help): p          # Print current partition table
Command (m for help): d          # Delete the partition you want to resize
Selected partition 1

Command (m for help): n          # Create a new partition
Command action
   e   extended
   p   primary partition (1-4)
p                                # Choose primary
Partition number (1-4, default 1): 1
First sector (2048-10485759, default 2048): [Enter]  # Use default
Last sector, +sectors or +size{K,M,G} (2048-10485759, default 10485759): [Enter]  # Use all space

Command (m for help): w          # Write changes to disk
```

### Step 2: Resize the Filesystem

After modifying the partition table, you might need to reboot or run `partprobe` to make the kernel recognize the changes. Then resize the filesystem:

```bash
# For ext2/ext3/ext4 filesystems
sudo resize2fs /dev/sda1
```

That's it! Your partition should now use all available space.

## Part 2: Working with LVM - The Real Magic

LVM is where things get really flexible. Let's explore different scenarios you'll encounter.

### Adding a New Disk to an Existing LVM

Got a new disk and want to add it to your existing volume group? Here's how:

#### Step 1: Identify Your New Disk

```bash
fdisk -l
```

Look for your new disk, let's say it's `/dev/vdc`.

#### Step 2: Check Your Current Volume Group

```bash
vgdisplay
```

This shows your volume group name (e.g., "DebianTemplate").

#### Step 3: Prepare and Add the New Disk

```bash
# Create a physical volume
pvcreate /dev/vdc

# Add it to your volume group
vgextend DebianTemplate /dev/vdc

# Verify the free space
vgdisplay
```

#### Step 4: Extend Your Logical Volume

```bash
# Extend by specific size (e.g., 199GB)
lvextend -L+199G /dev/mapper/DebianTemplate-root

# Or use all available free space
lvextend -l +100%FREE /dev/mapper/DebianTemplate-root
```

#### Step 5: Resize the Filesystem

```bash
resize2fs /dev/mapper/DebianTemplate-root
```

#### Step 6: Verify

```bash
df -h
```

You should see your increased storage space!

### Expanding an Existing Disk (VMware Style)

If you expanded a disk in VMware without adding a new one:

```bash
# Rescan for disk changes
echo "- - -" > /sys/class/scsi_host/host0/scan

# Check what changed
fdisk -l

# Resize the physical volume
pvresize /dev/vdb

# Then follow steps 4-6 from above
```

### Creating a Brand New LVM Setup

Starting from scratch? Here's the complete process:

#### Step 1: Prepare Your Disk

```bash
fdisk -l  # Find your disk (e.g., /dev/vdc)
fdisk /dev/vdc
```

In fdisk:
```
n    # New partition
p    # Primary
     # Accept defaults for partition number and first sector
     # Accept default for last sector (uses whole disk)
t    # Change partition type
8e   # Linux LVM
w    # Write changes
```

#### Step 2: Create Your LVM Structure

```bash
# Create physical volume
pvcreate /dev/vdc1

# Create volume group
vgcreate binbit /dev/vdc1

# Verify everything looks good
vgscan
pvscan
```

#### Step 3: Create a Logical Volume

```bash
# Create a logical volume (leaving some space free is wise)
lvcreate -n data -L 19G binbit

# Check your work
vgdisplay
```

#### Step 4: Create and Mount the Filesystem

```bash
# Create the filesystem
mkfs.ext4 /dev/binbit/data

# Create mount point
mkdir /data

# Mount it
mount /dev/binbit/data /data

# Verify
df -h
```

#### Step 5: Make It Permanent

Add this line to `/etc/fstab` so it mounts automatically:

```
/dev/binbit/data /data ext4 defaults 0 2
```

## Real-World Example Walkthrough

Let's follow a complete example from the field:

```bash
# Create the physical volume
root@server:~# pvcreate /dev/vdc1
Physical volume "/dev/vdc1" successfully created

# Create volume group
root@server:~# vgcreate binbit /dev/vdc1
Volume group "binbit" successfully created

# Create logical volume (19GB out of 20GB, leaving some free space)
root@server:~# lvcreate -n data -L 19G binbit
Logical volume "data" created

# Format the filesystem
root@server:~# mkfs.ext4 /dev/binbit/data

# Mount it
root@server:~# mkdir /data
root@server:~# mount /dev/binbit/data /data

# Check the results
root@server:~# df -h
/dev/mapper/binbit-data   19G   44M   18G   1% /data
```

## Pro Tips for LVM Success

### Hot Disk Addition in VMware

```bash
# Rescan SCSI buses to detect new disks without rebooting
echo "- - -" > /sys/class/scsi_host/host0/scan
echo "- - -" > /sys/class/scsi_host/host1/scan
echo "- - -" > /sys/class/scsi_host/host2/scan

# Check system logs
tail -f /var/log/messages
```

### Useful LVM Commands to Remember

```bash
# Show physical volumes
pvs

# Show volume groups
vgs

# Show logical volumes
lvs

# Detailed volume group info
vgdisplay

# Check filesystem before resizing
e2fsck -f /dev/mapper/vg-lv

# Use all free space in volume group
lvextend -l +100%FREE /dev/mapper/vg-lv
```

### Safety First!

- **Always backup critical data** before partition operations
- **Test your procedures** on non-production systems first
- **Check filesystem integrity** with `e2fsck -f` before resizing
- **Leave some free space** in volume groups for flexibility
- **Document your LVM layout** for future reference

## Troubleshooting Common Issues

**"Device or resource busy"**: The partition is mounted or in use. Unmount it first or reboot.

**"Bad magic number in super-block"**: You're trying to resize the disk instead of the partition. Use `/dev/sdb1` not `/dev/sdb`.

**"Please run e2fsck first"**: The filesystem needs checking before resizing:
```bash
e2fsck -f /dev/device
resize2fs /dev/device
```

## The Bottom Line

LVM gives you incredible flexibility with storage management. You can:
- Add disks on the fly
- Resize volumes without downtime
- Move data between physical disks
- Create snapshots for backups
- Pool multiple disks together

Once you get comfortable with these commands, you'll find storage management becomes much less stressful. No more "disk full" panics at 3 AM! 🚀

Remember: practice these techniques in a test environment first, always backup your data, and when in doubt, take it slow and double-check your commands.
