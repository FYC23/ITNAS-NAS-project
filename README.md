# Build a Raspberry Pi NAS in Under 50 Minutes

A quick tutorial for setting up network file sharing with read-only guest access and full access for authenticated users.

**Source:** Based on [PCMag's Raspberry Pi NAS tutorial](https://www.pcmag.com/how-to/how-to-turn-a-raspberry-pi-into-a-nas-for-cheap-file-storage) by Whitson Gordon

**Time:** ~50 minutes | **Difficulty:** Beginner-friendly

---

## What You'll Need

- **Raspberry Pi** (any model, Pi 4+ recommended)
- **Power supply & microSD card** (8GB+)
- **External USB drive** (wall-powered or via powered USB hub)
- **Ethernet cable** (optional but recommended for speed)
- **Keyboard, mouse, monitor** (for initial setup)

---

## Part 1: Initial Setup (15 min)

### 1. Install Raspbian OS
Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to flash Raspbian to your microSD card. Boot up the Pi, create a password, and run initial updates.

**Connect via Ethernet** for faster file transfers. SSH works too if you prefer remote setup.

### 2. Update System & Install Samba
```bash
sudo apt update
sudo apt upgrade
sudo apt install samba samba-common
```
**What this does:** Updates package lists, upgrades installed packages, then installs Samba (the file-sharing software).

When prompted about WINS settings, choose **Yes**.

---

## Part 2: Prepare Your Drive (15 min)

### 3. Find Your Drive
```bash
sudo fdisk -l
```
**What this does:** Lists all connected disks. Identify your external drive (e.g., `/dev/sda`).

⚠️ **Warning:** The following steps will ERASE your drive. Back up any important data first.

### 4. Unmount the Drive
```bash
umount /dev/sda1
```
**What this does:** Disconnects the drive so you can modify it. Adjust numbers if you have multiple partitions (`sda2`, `sda3`, etc.).

### 5. Partition the Drive
```bash
sudo parted /dev/sda
```

In the Parted wizard, enter these commands one by one:
```
mklabel gpt
mkpart
```

When prompted, enter:
- **Partition name:** `MyExternalDrive` (or your choice)
- **File system type:** `ext4`
- **Start:** `0%`
- **End:** `100%`

Type `quit` to exit.

**What this does:** Creates a new GPT partition table and one partition using the entire drive.

### 6. Format the Partition
```bash
sudo mkfs.ext4 /dev/sda1
sudo e2label /dev/sda1 MyExternalDrive
```
**What this does:** Formats the partition with ext4 filesystem and labels it. This takes a few minutes for large drives.

### 7. Reboot and Set Permissions
```bash
sudo shutdown -r now
```

After reboot:
```bash
sudo chown -R pi:pi /media/pi/MyExternalDrive/
sudo chmod -R 755 /media/pi/MyExternalDrive/
sudo chmod 755 /media/pi/
```
**What this does:** Gives your user ownership of the drive and makes directories readable by guests.

---

## Part 3: Configure File Sharing (15 min)

### 8. Edit Samba Configuration
```bash
sudo nano /etc/samba/smb.conf
```

Scroll to the bottom and add:

```ini
[MyMedia]
path = /media/pi/MyExternalDrive/
writeable = no
read only = yes
create mask = 0555
directory mask = 0555
public = yes
guest ok = yes
guest only = yes
browseable = yes
force user = pi

[MyMedia-Write]
path = /media/pi/MyExternalDrive/
writeable = yes
read only = no
create mask = 0775
directory mask = 0775
valid users = pi
browseable = yes
guest ok = no
```

**What this does:**
- **MyMedia** = Read-only share for guests
- **MyMedia-Write** = Full access share requiring authentication
- `force user = pi` = Guests borrow your user's read permissions
- `guest ok = yes` = Allows passwordless guest access

Press `Ctrl+X`, then `Y`, then `Enter` to save and exit.

### 9. Create Samba Password
```bash
sudo smbpasswd -a pi
```
**What this does:** Sets a password for Samba access. Can be different from your Pi's login password.

*Optional:* Add more users with `sudo adduser username` then `sudo smbpasswd -a username`

### 10. Restart Samba
```bash
sudo systemctl restart smbd nmbd
```
**What this does:** Applies your configuration changes.

### 11. Verify Configuration
```bash
testparm -s
```
**What this does:** Checks for syntax errors in your Samba config.

---

## Part 4: Connect & Test (5 min)

### From Windows:
1. Open File Explorer
2. Type in address bar: `\\raspberrypi\MyMedia` (guest access) or `\\raspberrypi\MyMedia-Write` (full access)

### From Mac:
1. Finder → `Cmd+K`
2. Enter: `smb://raspberrypi/MyMedia` (guest) or `smb://raspberrypi/MyMedia-Write` (authenticated)
3. Select **Guest** or enter your username/password

**Troubleshooting:** If hostname doesn't work, use IP address instead: `\\192.168.1.X\MyMedia` or `smb://192.168.1.X/MyMedia`

Find your Pi's IP with: `hostname -I`

---

## Understanding Key Concepts

**Samba:** Open-source implementation of SMB/CIFS file sharing protocol (Windows-compatible)

**Guest Access:** Passwordless, read-only access for casual viewing

**Force User:** Makes guests act as your user for file permissions (but Samba still prevents writing)

**Valid Users:** Restricts share access to specific authenticated users

---

## What's Next?

- Add more drives and create separate shares
- Set up automatic backups with `rsync`
- Configure static IP address for easier access
- Add time machine support for Mac backups

**Important:** This setup isn't RAID-protected. Always backup critical data elsewhere!

---

## Quick Reference

```bash
# Find Pi IP address
hostname -I

# Check drive status
df -h

# View Samba logs
sudo tail -f /var/log/samba/log.smbd

# Restart Samba after changes
sudo systemctl restart smbd nmbd

# Test configuration
testparm -s
```

