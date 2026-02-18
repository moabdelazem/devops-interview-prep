# Linux -- Notes & Cheat Sheet

Quick-reference commands and concepts for revision, focused on RHEL/CentOS.

---

## System Information

```bash
uname -a                # Kernel version and system architecture
hostnamectl             # Hostname and OS details
uptime                  # Load average and uptime
cat /etc/os-release     # Distribution info
lscpu                   # CPU details
lsblk                   # Block devices (disks)
free -h                 # Memory usage
```

---

## File Operations

```bash
ls -la                  # List all files (including hidden) with details
cp -r source dest       # Copy directory recursively
mv old new              # Move/Rename
rm -rf dir              # Force delete directory (careful!)
find / -name "*.log"    # Find files by name
grep -r "error" /var/log/ # Search for text in files recursively
tail -f /var/log/messages # Follow log file real-time
```

---

## Permissions

```bash
chmod 755 file          # rwxr-xr-x
chmod +x script.sh      # Make executable
chown user:group file   # Change owner and group
chgrp group file        # Change group only
umask 022               # Set default permissions (755 for dirs, 644 for files)
ls -l                   # View permissions
lsattr file             # View file attributes (immutable bit)
chattr +i file          # Make file immutable
```

---

## Process Management

```bash
ps aux                  # List all running processes
top                     # Real-time process monitoring
htop                    # Improved top (if installed)
kill <PID>              # Terminate process (SIGTERM - 15)
kill -9 <PID>           # Force kill process (SIGKILL - 9)
pkill -f name           # Kill process by name
pgrep name              # Find PID by name
nice -n 10 command      # Start process with lower priority
renice 10 <PID>         # Change priority of running process
```

---

## Networking

```bash
ip a                    # Show IP addresses
ip r                    # Show routing table
ss -tlnp                # Show listening TCP ports and processes (modern netstat)
netstat -tulpn          # Legacy version of ss
ping -c 4 google.com    # Check connectivity
dig google.com          # DNS lookup
nslookup google.com     # DNS lookup (legacy)
curl -I google.com      # Check HTTP headers
nc -zv host port        # Check if port is open (netcat)
firewall-cmd --list-all # Show RHEL firewall rules
nmcli device status     # NetworkManager status
```

---

## Disk Usage

```bash
df -h                   # Filesystem usage (human readable)
du -sh directory        # Directory size summary
du -h --max-depth=1     # Size of subdirectories
mount                   # List mounted filesystems
fdisk -l                # List partition tables
mkfs.xfs /dev/sdb1      # Format disk as XFS (RHEL default)
```

---

## Package Management (RHEL / CentOS)

```bash
yum update              # Update all packages
yum install package     # Install package
yum remove package      # Remove package
yum search keyword      # Search for package
yum info package        # Show package details
rpm -qa                 # List all installed RPMs
rpm -ivh package.rpm    # Install local RPM
rpm -e package          # Erase RPM
```

_Note: `dnf` is the modern replacement for `yum`, but commands are mostly identical._

---

## Systemd (Service Management)

```bash
systemctl start httpd   # Start service
systemctl stop httpd    # Stop service
systemctl restart httpd # Restart service
systemctl reload httpd  # Reload config without stopping
systemctl enable httpd  # Enable at boot
systemctl disable httpd # Disable at boot
systemctl status httpd  # Check status
systemctl list-units    # List active units
```

---

## Logs (Journalctl)

```bash
journalctl -xe          # Show recent errors system-wide
journalctl -u httpd     # Show logs for specific service
journalctl -f           # Follow logs real-time
journalctl --since "1 hour ago" # Show logs from last hour
```

---

## Archiving

```bash
tar -czvf archive.tar.gz /path/to/dir  # Create compressed archive
tar -xzvf archive.tar.gz               # Extract archive
zip -r archive.zip /path/to/dir        # Create zip
unzip archive.zip                      # Extract zip
```

---

## User Management

```bash
useradd -m username     # Create user with home dir
passwd username         # Set password
usermod -aG group user  # Add user to group
userdel -r username     # Delete user and home dir
groupadd groupname      # Create group
id username             # Show user ID and groups
who                     # Show logged in users
w                       # Show who is logged in and what they are doing
last                    # Show login history
```
