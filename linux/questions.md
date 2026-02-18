# Linux -- Interview Questions

General and conceptual interview questions with clear answers, biased towards RHEL/CentOS.

---

## Fundamentals

### Q1: Explain the Linux Boot Process (RHEL 7/8/9).

**Answer:**

The boot process consists of 4 main stages:

1.  **BIOS/UEFI**
    - Performs POST (Power On Self Test).
    - Identifies the boot device (Disk, Network, USB).
    - Loads the Bootloader (GRUB2) from the MBR (Legacy) or EFI System Partition (UEFI).

2.  **Bootloader (GRUB2)**
    - Displays the splash screen and kernel selection menu.
    - Loads the Kernel (`vmlinuz`) and Initial RAM Disk (`initramfs`) into memory.

3.  **Kernel**
    - Initializes hardware and mounts the root filesystem as read-only.
    - Extracts `initramfs` to load necessary drivers (LVM, RAID, filesystem drivers).
    - Starts the first process: `systemd` (PID 1).

4.  **Init (systemd)**
    - Reads configuration from `/etc/systemd/`.
    - Mounts filesystems defined in `/etc/fstab`.
    - Starts services and reaches the `default.target` (usually `graphical.target` or `multi-user.target`).

---

### Q2: What is `systemd` and how does it differ from SysVinit?

**Answer:**

`systemd` is the system and service manager for modern Linux distributions (RHEL 7+).

| Feature              | SysVinit (Legacy)                   | systemd (Modern)                              |
| :------------------- | :---------------------------------- | :-------------------------------------------- |
| **Parallelism**      | Serial processing (slower boot)     | Parallel start of services (faster boot)      |
| **Dependency Mgmt**  | Shell scripts with manual numbering | Declarative unit files with auto-dependencies |
| **Process Tracking** | PIDs (processes can escape)         | Cgroups (tracks all child processes)          |
| **Recovery**         | Manual restarts usually required    | Auto-restart capabilities                     |
| **Command**          | `service httpd start`               | `systemctl start httpd`                       |

---

### Q3: What is a Hard Link vs. a Soft (Symbolic) Link?

**Answer:**

| Feature         | Hard Link                                           | Soft Link (Symlink)                    |
| :-------------- | :-------------------------------------------------- | :------------------------------------- |
| **Pointer**     | Points to the **inode** (same file data)            | Points to the **file path** (shortcut) |
| **Inode**       | Shares the same Inode number                        | Has a different/unique Inode number    |
| **Partitions**  | Must be on the **same** partition                   | Can cross partitions/filesystems       |
| **Deletion**    | File data persists until all hard links are deleted | File deleted = Broken link (dangling)  |
| **Directories** | Cannot link directories (usually)                   | Can link directories                   |
| **Command**     | `ln file link`                                      | `ln -s file link`                      |

---

### Q4: Explain the permissions `755` and `644`. What is `umask`?

**Answer:**

Permissions are represented by octal numbers: Read (4), Write (2), Execute (1).

- **755 (`rwxr-xr-x`)**:
  - **User (Owner):** Read + Write + Execute (4+2+1 = 7)
  - **Group:** Read + Execute (4+0+1 = 5)
  - **Others:** Read + Execute (4+0+1 = 5)
  - _Common for directories and executable scripts._

- **644 (`rw-r--r--`)**:
  - **User:** Read + Write (4+2 = 6)
  - **Group:** Read only (4)
  - **Others:** Read only (4)
  - _Standard for regular files._

**Umask (User File Creation Mode Mask):**
Determines default permissions for new files/directories. It is "subtracted" from the base permission.

- Base for files: `666`
- Base for directories: `777`
- Default RHEL umask: `0022`
  - New File: 666 - 022 = 644
  - New Dir: 777 - 022 = 755

---

### Q5: What is LVM (Logical Volume Management)?

**Answer:**

LVM allows flexible disk management, separating the filesystem layout from physical disk partitions.

**Key Components:**

1.  **Physical Volume (PV):** The raw disk or partition (e.g., `/dev/sdb1`).
2.  **Volume Group (VG):** A pool of storage created by combining one or more PVs.
3.  **Logical Volume (LV):** A partition created from the VG pool. This is where you put the filesystem (e.g., xfs/ext4).

**Advantages:**

- Resize filesystems online (extend without unmounting).
- Span a filesystem across multiple physical disks.
- Snapshot capabilities for backups.

**Basic Commands:**

```bash
pvcreate /dev/sdb1              # Create PV
vgcreate my_vg /dev/sdb1        # Create VG
lvcreate -L 10G -n my_lv my_vg  # Create LV
mkfs.xfs /dev/my_vg/my_lv       # Format
```

---

### Q6: What is an Inode?

**Answer:**

An Inode (Index Node) is a data structure on a filesystem that stores metadata about a file, but **NOT** the filename or the data content itself.

**It stores:**

- File size
- Owner (UID/GID)
- Permissions
- Timestamps (mtime, ctime, atime)
- Location of data blocks on disk

**Inode exhaustion:** You can run out of inodes even if you have disk space left (common with millions of tiny files).
Check inodes with: `df -i`

---

### Q7: What is Load Average?

**Answer:**

Load average represents the average number of processes that are **runnable** (using CPU) or **uninterruptible** (waiting for I/O, like disk or network) over a period of time.

Output from `uptime` or `top`: `load average: 0.50, 1.20, 3.00`

1.  1-minute average
2.  5-minute average
3.  15-minute average

**Interpretation:**

- **Load = 0:** Completely idle.
- **Load = 1.0 (on single CPU):** 100% utilization.
- **Load > 1.0 (on single CPU):** Processes are waiting (system is overloaded).
- **Load = 4.0 (on 4 CPU core system):** 100% utilization (optimal).

---

### Q8: What is the difference between `dmesg`, `/var/log/messages`, and `/var/log/secure`?

**Answer:**

- **`dmesg`**: Displays messages from the kernel ring buffer. Used to debug hardware issues, boot output, and driver failures generally _before_ syslog starts or for kernel-level events.
- **`/var/log/messages`** (or `/var/log/syslog` on Debian/Ubuntu): The main global system log. Contains info from systemd, services, and general errors.
- **`/var/log/secure`** (or `/var/log/auth.log`): Contains security and authentication logs. Tracks SSH logins, `sudo` usage, and failed password attempts.

---

### Q9: How do you harden SSH access?

**Answer:**

Edit `/etc/ssh/sshd_config`:

1.  **Disable Root Login:** `PermitRootLogin no`
2.  **Disable Password Auth:** `PasswordAuthentication no` (Force key-based auth)
3.  **Change Default Port:** `Port 2222` (Reduces noise from automated scanners)
4.  **Limit Users:** `AllowUsers user1 user2`
5.  **Disable Empty Passwords:** `PermitEmptyPasswords no`
6.  **Protocol 2 only:** (Default in modern SSH)

Don't forget to restart sshd: `systemctl restart sshd`

---

### Q10: What is Swap memory?

**Answer:**

Swap is space on the disk used as virtual memory when physical RAM is full.

- When RAM is utilized (near 100%), the kernel moves inactive pages from RAM to Swap to free up space for active processes.
- **Disadvantage:** Disk I/O is significantly slower than RAM, so heavy swapping kills performance.
- **Swappiness:** Kernel parameter (`/proc/sys/vm/swappiness`) that controls how aggressive the system is about swapping (0-100). Default is usually 60.

Check swap usage: `free -h` or `swapon --show`

---

### Q11: Explain the difference between `kill`, `kill -9`, and `kill -15`.

**Answer:**

`kill` sends a signal to a process.

- **`kill -15` (SIGTERM):** The default signal. Requests the process to stop securely. Allows the application to close files, clean up locks, and shut down gracefully.
- **`kill -9` (SIGKILL):** Forcefully kills the process immediately. The process cannot catch or ignore this signal. No cleanup happens (can lead to data corruption).
- **`kill -1` (SIGHUP):** Hang-up signal. Often used to instruct a service to reload its configuration files without restarting the process.

---

### Q12: What is the difference between TCP and UDP?

**Answer:**

| Feature         | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol)      |
| :-------------- | :---------------------------------- | :-------------------------------- |
| **Connection**  | Connection-oriented (Handshake)     | Connection-less (Fire and forget) |
| **Reliability** | Reliable (ACKs, Retransmission)     | Unreliable (No guarantees)        |
| **Ordering**    | Guarantees packet order             | Packets may arrive out of order   |
| **Speed**       | Slower (Overhead)                   | Faster (Low overhead)             |
| **Use Cases**   | HTTP, SSH, FTP, Email               | DNS, Streaming, VoIP, Gaming      |

---

### Q13: How do you use `awk`, `sed`, and `cut`?

**Answer:**

These are text-processing tools. Each has a different strength.

**`cut` -- Extract columns/fields from each line**

```bash
# Extract the 1st and 3rd field (default delimiter: TAB)
cut -f1,3 file.txt

# Extract using a custom delimiter (colon)
cut -d: -f1,7 /etc/passwd        # username and shell

# Extract character positions
cut -c1-8 /etc/passwd             # first 8 characters of each line
```

**`awk` -- Pattern scanning and field processing**

`awk` splits each line into fields (`$1`, `$2`, ...) by whitespace (default).

```bash
# Print the 1st and 3rd column
awk '{print $1, $3}' file.txt

# Use a custom delimiter
awk -F: '{print $1, $7}' /etc/passwd     # username and shell

# Filter by condition
awk '$3 > 1000' /etc/passwd              # lines where 3rd field > 1000

# Sum a column
awk '{sum += $5} END {print sum}' file.txt

# Print lines matching a pattern
awk '/error/ {print $0}' /var/log/messages

# Print line number and line
awk '{print NR, $0}' file.txt

# Print specific fields with formatting
awk -F: '{printf "%-20s %s\n", $1, $7}' /etc/passwd
```

**`sed` -- Stream editor for find-and-replace and line operations**

```bash
# Replace first occurrence on each line
sed 's/old/new/' file.txt

# Replace ALL occurrences on each line (global)
sed 's/old/new/g' file.txt

# Edit file in-place
sed -i 's/old/new/g' file.txt

# Delete lines matching a pattern
sed '/^#/d' file.conf              # remove comment lines

# Delete empty lines
sed '/^$/d' file.txt

# Print only lines 5-10
sed -n '5,10p' file.txt

# Insert text before line 3
sed '3i\new line of text' file.txt

# Append text after line 3
sed '3a\new line of text' file.txt

# Replace only on lines matching a pattern
sed '/error/s/old/new/g' file.txt
```

**When to use which:**

| Tool  | Best for                                       | Key strength                                |
| ----- | ---------------------------------------------- | ------------------------------------------- |
| `cut` | Extracting fixed columns from delimited data   | Simple, fast, single-purpose                |
| `awk` | Field-based processing, conditions, arithmetic | Mini programming language for columnar data |
| `sed` | Search-and-replace, line insertions/deletions  | In-place text transformations               |

---

## System Administration

### Q14: Explain the Linux Filesystem Hierarchy (FHS).

**Answer:**

The Filesystem Hierarchy Standard defines the directory structure on Linux:

| Path     | Purpose                                                        |
| :------- | :------------------------------------------------------------- |
| `/`      | Root of the entire filesystem                                  |
| `/bin`   | Essential user binaries (`ls`, `cp`, `cat`)                    |
| `/sbin`  | System binaries (`fdisk`, `iptables`, `reboot`)                |
| `/etc`   | System-wide configuration files                                |
| `/home`  | User home directories                                          |
| `/root`  | Home directory for the root user                               |
| `/var`   | Variable data: logs (`/var/log`), spool, mail, temp runtime    |
| `/tmp`   | Temporary files (cleared on reboot by default)                 |
| `/usr`   | Secondary hierarchy: user programs, libraries, docs            |
| `/opt`   | Optional/third-party software packages                         |
| `/dev`   | Device files (`/dev/sda`, `/dev/null`, `/dev/tty`)             |
| `/proc`  | Virtual filesystem exposing kernel/process info (not on disk)  |
| `/sys`   | Virtual filesystem for kernel objects (devices, drivers, etc.) |
| `/boot`  | Bootloader files, kernel (`vmlinuz`), `initramfs`, GRUB config |
| `/mnt`   | Temporary mount points (manual mounts)                         |
| `/media` | Auto-mount points for removable media (USB, CD)                |
| `/lib`   | Shared libraries needed by `/bin` and `/sbin`                  |
| `/run`   | Runtime data since last boot (PIDs, sockets, lock files)       |

---

### Q15: How do you manage users and groups?

**Answer:**

**User management:**

```bash
useradd devuser                     # Create user
useradd -m -s /bin/bash devuser     # Create with home dir and shell
passwd devuser                      # Set password
usermod -aG wheel devuser           # Add user to supplementary group
userdel -r devuser                  # Delete user and home directory
id devuser                          # Show UID, GID, groups
```

**Group management:**

```bash
groupadd devops                     # Create group
groupdel devops                     # Delete group
gpasswd -a devuser devops           # Add user to group
gpasswd -d devuser devops           # Remove user from group
groups devuser                      # List groups for a user
```

**Key files:**

| File           | Content                               |
| :------------- | :------------------------------------ |
| `/etc/passwd`  | User accounts (name, UID, GID, shell) |
| `/etc/shadow`  | Encrypted passwords and aging info    |
| `/etc/group`   | Group definitions and memberships     |
| `/etc/gshadow` | Group passwords (rarely used)         |

---

### Q16: What are SUID, SGID, and the Sticky Bit?

**Answer:**

These are special permission bits beyond the standard rwx.

**SUID (Set User ID) -- `4xxx`:**
When set on an executable, it runs with the **file owner's** privileges, not the caller's.

```bash
chmod u+s /usr/bin/passwd           # or chmod 4755
ls -l /usr/bin/passwd
# -rwsr-xr-x. 1 root root ... /usr/bin/passwd
```

The `s` in the user-execute position means SUID is set. This is why normal users can change their own password (the binary runs as root).

**SGID (Set Group ID) -- `2xxx`:**
On a file: runs with the file's group privileges.
On a directory: new files created inside inherit the directory's group, not the creator's primary group.

```bash
chmod g+s /shared/project           # or chmod 2775
```

**Sticky Bit -- `1xxx`:**
On a directory: only the file owner (or root) can delete/rename files, even if others have write permission to the directory.

```bash
chmod +t /tmp                       # or chmod 1777
ls -ld /tmp
# drwxrwxrwt. ... /tmp
```

The `t` in the others-execute position indicates the sticky bit. This is why users cannot delete each other's files in `/tmp`.

---

### Q17: How does package management work on RHEL (yum/dnf)?

**Answer:**

`dnf` (RHEL 8+) replaced `yum` (RHEL 7). Both manage RPM packages with automatic dependency resolution.

```bash
dnf install httpd                   # Install a package
dnf remove httpd                    # Remove a package
dnf update                          # Update all packages
dnf update httpd                    # Update a specific package
dnf search nginx                    # Search repos for a package
dnf info httpd                      # Show package details
dnf list installed                  # List all installed packages
dnf provides /etc/httpd/conf/httpd.conf   # Find which package owns a file
dnf history                         # Show transaction history
dnf history undo 15                 # Undo a specific transaction
dnf clean all                       # Clear cached metadata and packages
```

**Repository configuration:**
Repo files live in `/etc/yum.repos.d/*.repo`. Each file defines `baseurl`, `gpgcheck`, and `enabled` flags.

```bash
dnf repolist                        # List enabled repos
dnf repolist all                    # List all repos (enabled + disabled)
dnf config-manager --add-repo <url> # Add a new repo
```

---

### Q18: How do you schedule tasks with `cron` and `at`?

**Answer:**

**Cron (recurring tasks):**

Edit the user crontab with `crontab -e`. Format:

```
MIN  HOUR  DOM  MON  DOW  COMMAND
 *    *     *    *    *    /path/to/script.sh
```

| Field | Range         |
| :---- | :------------ |
| MIN   | 0-59          |
| HOUR  | 0-23          |
| DOM   | 1-31          |
| MON   | 1-12          |
| DOW   | 0-7 (0,7=Sun) |

Examples:

```bash
0 2 * * * /opt/backup.sh            # Every day at 2:00 AM
*/5 * * * * /opt/health-check.sh    # Every 5 minutes
0 0 * * 0 /opt/weekly-report.sh     # Every Sunday at midnight
30 8 1 * * /opt/monthly-task.sh     # 1st of every month at 8:30 AM
```

```bash
crontab -l                          # List current user's cron jobs
crontab -e                          # Edit crontab
crontab -r                          # Remove all cron jobs
```

System-wide cron directories: `/etc/cron.daily/`, `/etc/cron.hourly/`, `/etc/cron.weekly/`, `/etc/cron.monthly/`

**at (one-time tasks):**

```bash
at now + 30 minutes                 # Run once, 30 minutes from now
at 02:00 AM tomorrow                # Run once at 2 AM tomorrow
atq                                 # List pending at jobs
atrm 3                              # Remove job number 3
```

---

### Q19: How do you manage processes?

**Answer:**

```bash
ps aux                              # List all running processes (BSD style)
ps -ef                              # List all processes (System V style)
ps aux | grep nginx                 # Find a specific process
top                                 # Real-time process monitor
htop                                # Improved interactive process viewer (install separately)
```

**Background and foreground:**

```bash
./long-task.sh &                    # Start a job in the background
jobs                                # List background jobs
fg %1                               # Bring job 1 to foreground
bg %1                               # Resume a stopped job in background
Ctrl+Z                              # Suspend (stop) current foreground process
```

**Priority (nice values):**

Nice values range from -20 (highest priority) to 19 (lowest). Default is 0.

```bash
nice -n 10 ./cpu-heavy-task.sh      # Start with lower priority
renice -n 5 -p 1234                 # Change priority of running process (PID 1234)
```

**Zombie and orphan processes:**

- **Zombie:** A process that has finished but its parent has not yet read its exit status (via `wait()`). Shows as `Z` in `ps`.
- **Orphan:** A process whose parent has terminated. Adopted by PID 1 (`systemd`).

---

### Q20: What are `systemctl` commands you should know?

**Answer:**

```bash
systemctl start httpd               # Start a service
systemctl stop httpd                # Stop a service
systemctl restart httpd             # Stop then start
systemctl reload httpd              # Reload config without restarting
systemctl status httpd              # Show service status, recent logs, PID

systemctl enable httpd              # Start on boot
systemctl disable httpd             # Do not start on boot
systemctl is-enabled httpd          # Check if enabled
systemctl is-active httpd           # Check if currently running

systemctl mask httpd                # Completely prevent starting (even manually)
systemctl unmask httpd              # Undo mask

systemctl list-units --type=service          # List loaded services
systemctl list-units --type=service --all    # Include inactive
systemctl list-unit-files --type=service     # List install state of all services

systemctl daemon-reload             # Reload unit files after editing them

systemctl get-default               # Show default boot target
systemctl set-default multi-user.target      # Boot to CLI (no GUI)
systemctl set-default graphical.target       # Boot to GUI
```

**Unit file locations:**

| Path                       | Priority                  |
| :------------------------- | :------------------------ |
| `/etc/systemd/system/`     | Highest (admin overrides) |
| `/run/systemd/system/`     | Runtime units             |
| `/usr/lib/systemd/system/` | Vendor/package defaults   |

---

### Q21: How do you use `journalctl` to query logs?

**Answer:**

`journalctl` reads logs from `systemd`'s journal (binary log storage).

```bash
journalctl                          # Show all logs (oldest first)
journalctl -e                       # Jump to end of logs
journalctl -f                       # Follow logs in real time (like tail -f)
journalctl -u httpd                 # Logs for a specific service
journalctl -u httpd --since "1 hour ago"
journalctl -u httpd --since "2026-02-01" --until "2026-02-02"
journalctl -p err                   # Only error-level and above
journalctl -p warning -u sshd       # Warnings+ for sshd
journalctl -b                       # Logs from current boot
journalctl -b -1                    # Logs from previous boot
journalctl --disk-usage             # Show how much disk the journal uses
journalctl --vacuum-size=500M       # Shrink journal to 500M
journalctl -k                       # Kernel messages only (like dmesg)
journalctl -o json-pretty -u httpd  # Output in JSON format
```

Persistent journaling (survives reboots) requires `/var/log/journal/` to exist. On RHEL, create it with `mkdir -p /var/log/journal && systemd-tmpfiles --create --prefix /var/log/journal`.

---

### Q22: How do you mount filesystems and use `/etc/fstab`?

**Answer:**

**Manual mount:**

```bash
mount /dev/sdb1 /mnt/data           # Mount a partition
mount -t xfs /dev/sdb1 /mnt/data    # Specify filesystem type
umount /mnt/data                    # Unmount
mount -o ro /dev/sdb1 /mnt/data     # Mount read-only
lsblk                               # List block devices and mount points
blkid                                # Show UUIDs and filesystem types
```

**Permanent mounts with `/etc/fstab`:**

Each line in `/etc/fstab` defines a mount:

```
<device>        <mount-point>  <type>  <options>       <dump> <pass>
UUID=abc-123    /mnt/data      xfs     defaults        0      2
/dev/sdb1       /mnt/backup    ext4    defaults,noexec 0      2
```

| Field   | Meaning                                             |
| :------ | :-------------------------------------------------- |
| device  | Block device or UUID (UUID preferred for stability) |
| mount   | Target directory                                    |
| type    | Filesystem (`xfs`, `ext4`, `nfs`, `swap`)           |
| options | `defaults`, `ro`, `noexec`, `nosuid`, `nofail`      |
| dump    | 0 = skip backup, 1 = include in dump                |
| pass    | fsck order (0 = skip, 1 = root, 2 = other)          |

```bash
mount -a                            # Mount everything in fstab (test after editing)
```

A bad `/etc/fstab` entry can prevent the system from booting. Always test with `mount -a` before rebooting.

---

### Q23: How do you check disk usage?

**Answer:**

**`df` -- Disk Free (filesystem-level):**

```bash
df -h                               # Human-readable sizes for all mounted filesystems
df -hT                              # Include filesystem type
df -i                               # Show inode usage instead of block usage
```

**`du` -- Disk Usage (directory-level):**

```bash
du -sh /var/log                     # Total size of a directory
du -sh /var/log/*                   # Size of each item inside
du -sh /home/* | sort -rh | head    # Top largest home directories
du -ah /var/log | sort -rh | head -20   # Top 20 largest files/dirs
```

**Quick disk hog hunting:**

```bash
# Find the largest files on the system
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```

---

### Q24: How do you use `tar` and compression tools?

**Answer:**

`tar` bundles files into an archive. Combine with compression tools for smaller output.

```bash
# Create a gzip-compressed archive
tar -czvf backup.tar.gz /etc/nginx/

# Create a bzip2-compressed archive (slower, smaller)
tar -cjvf backup.tar.bz2 /var/log/

# Create an xz-compressed archive (slowest, smallest)
tar -cJvf backup.tar.xz /opt/app/

# Extract a gzip archive
tar -xzvf backup.tar.gz

# Extract to a specific directory
tar -xzvf backup.tar.gz -C /tmp/restore/

# List contents without extracting
tar -tzvf backup.tar.gz
```

| Flag | Meaning           |
| :--- | :---------------- |
| `-c` | Create archive    |
| `-x` | Extract archive   |
| `-t` | List contents     |
| `-v` | Verbose output    |
| `-f` | Specify filename  |
| `-z` | gzip compression  |
| `-j` | bzip2 compression |
| `-J` | xz compression    |

**Standalone compression:**

```bash
gzip file.log                       # Compress (replaces original with file.log.gz)
gunzip file.log.gz                  # Decompress
zip archive.zip file1 file2         # Create zip archive
unzip archive.zip                   # Extract zip
```

---

### Q25: How do you transfer files between systems (`rsync`, `scp`)?

**Answer:**

**`scp` (Secure Copy):**

```bash
scp file.txt user@remote:/tmp/                  # Copy local to remote
scp user@remote:/var/log/app.log ./              # Copy remote to local
scp -r /opt/configs/ user@remote:/backup/        # Copy directory recursively
scp -P 2222 file.txt user@remote:/tmp/           # Custom SSH port
```

**`rsync` (Remote Sync):**

`rsync` is preferred over `scp` for large transfers because it only copies changed data (delta transfer).

```bash
rsync -avz /opt/app/ user@remote:/opt/app/       # Sync local to remote
rsync -avz user@remote:/var/log/ ./logs/          # Sync remote to local
rsync -avz --delete /src/ /dest/                  # Mirror (delete extra files in dest)
rsync -avz --exclude='*.log' /src/ /dest/         # Exclude patterns
rsync -avzP /large-file.iso user@remote:/tmp/     # Show progress, resumable
```

| Flag        | Meaning                                    |
| :---------- | :----------------------------------------- |
| `-a`        | Archive mode (preserves permissions, etc.) |
| `-v`        | Verbose                                    |
| `-z`        | Compress during transfer                   |
| `-P`        | Show progress + allow resume               |
| `--delete`  | Delete files in dest not in source         |
| `--dry-run` | Simulate without making changes            |

---

### Q26: What is the `/proc` filesystem?

**Answer:**

`/proc` is a virtual filesystem that exposes kernel and process information as regular files. Nothing in `/proc` exists on disk; it is generated dynamically by the kernel.

**Per-process info (`/proc/<PID>/`):**

```bash
ls /proc/1/                         # Info about PID 1 (systemd)
cat /proc/1/cmdline                 # Command that started the process
cat /proc/1/status                  # Process status (state, memory, UIDs)
ls -l /proc/1/fd/                   # Open file descriptors
cat /proc/1/environ                 # Environment variables
```

**System-wide info:**

| File                | Content                                   |
| :------------------ | :---------------------------------------- |
| `/proc/cpuinfo`     | CPU model, cores, speed                   |
| `/proc/meminfo`     | Total/free/available memory, swap         |
| `/proc/uptime`      | System uptime in seconds                  |
| `/proc/loadavg`     | Load average (1, 5, 15 min)               |
| `/proc/filesystems` | Supported filesystem types                |
| `/proc/mounts`      | Currently mounted filesystems             |
| `/proc/sys/`        | Tuneable kernel parameters (via `sysctl`) |

**Tuning kernel parameters at runtime:**

```bash
cat /proc/sys/net/ipv4/ip_forward               # Check IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward          # Enable (non-persistent)
sysctl -w net.ipv4.ip_forward=1                  # Same, using sysctl
sysctl -p                                        # Reload /etc/sysctl.conf (persistent)
```

---

### Q27: How do you use `find` to search for files?

**Answer:**

```bash
# Find by name
find /var/log -name "*.log"

# Find by name (case-insensitive)
find /etc -iname "*.conf"

# Find by type (f=file, d=directory, l=symlink)
find /opt -type d -name "config"

# Find by size
find / -type f -size +100M          # Files larger than 100MB
find /tmp -type f -size -1k         # Files smaller than 1KB

# Find by modification time
find /var/log -mtime -7             # Modified in the last 7 days
find /tmp -mtime +30                # Modified more than 30 days ago

# Find by permissions
find / -perm -4000 -type f          # Files with SUID bit set

# Find by owner
find /home -user devuser

# Find and execute a command
find /tmp -name "*.tmp" -delete                      # Delete matching files
find /var/log -name "*.log" -exec gzip {} \;         # Compress each result
find /opt -name "*.sh" -exec chmod +x {} \;          # Make scripts executable

# Find empty files/directories
find /var/log -empty
```

**`locate` (faster but uses a database):**

```bash
locate httpd.conf                   # Instant search from cached DB
updatedb                            # Rebuild the locate database (runs nightly by default)
```

---

### Q28: What is I/O redirection and piping?

**Answer:**

Every process has three standard streams:

| FD  | Name   | Default  |
| :-- | :----- | :------- |
| 0   | stdin  | Keyboard |
| 1   | stdout | Terminal |
| 2   | stderr | Terminal |

**Redirection operators:**

```bash
command > file.txt                  # Redirect stdout to file (overwrite)
command >> file.txt                 # Redirect stdout to file (append)
command 2> error.log                # Redirect stderr to file
command 2>&1                        # Redirect stderr to stdout
command > all.log 2>&1              # Redirect both stdout and stderr to file
command &> all.log                  # Shorthand for above (bash)
command < input.txt                 # Read stdin from file
```

**Piping (`|`):**
Sends the stdout of one command as stdin to the next.

```bash
ps aux | grep nginx | grep -v grep
cat /var/log/messages | sort | uniq -c | sort -rn | head
dmesg | tail -20
```

**`/dev/null` (the black hole):**

```bash
command > /dev/null 2>&1            # Discard all output
```

---

### Q29: How do you use `grep` and basic regular expressions?

**Answer:**

```bash
grep "error" /var/log/messages              # Search for a string in a file
grep -i "error" /var/log/messages           # Case-insensitive search
grep -r "TODO" /opt/app/                    # Recursive search in directory
grep -n "error" file.log                    # Show line numbers
grep -c "error" file.log                    # Count matching lines
grep -v "debug" file.log                    # Invert match (non-matching lines)
grep -l "error" /var/log/*.log              # List filenames with matches only
grep -w "root" /etc/passwd                  # Match whole word only
grep -A 3 "error" file.log                 # Show 3 lines after match
grep -B 2 "error" file.log                 # Show 2 lines before match
grep -C 2 "error" file.log                 # Show 2 lines before and after
```

**Common regex patterns (`grep -E` or `egrep`):**

```bash
grep -E "^root" /etc/passwd                 # Lines starting with "root"
grep -E "bash$" /etc/passwd                 # Lines ending with "bash"
grep -E "error|warning|critical" file.log   # OR pattern
grep -E "[0-9]{1,3}\.[0-9]{1,3}" file.log  # Pattern with repetition
```

---

### Q30: What are environment variables? How do you set them?

**Answer:**

Environment variables store configuration values accessible to the shell and child processes.

```bash
echo $HOME                          # Print a variable
echo $PATH                          # Print the executable search path
env                                 # List all environment variables
printenv USER                       # Print a specific variable

export MY_VAR="hello"               # Set for current shell + child processes
MY_VAR="hello"                      # Set for current shell only (not exported)
unset MY_VAR                        # Remove a variable
```

**Making variables persistent:**

| Scope         | File                                    |
| :------------ | :-------------------------------------- |
| Single user   | `~/.bashrc` or `~/.bash_profile`        |
| All users     | `/etc/profile` or `/etc/profile.d/*.sh` |
| systemd units | `Environment=` in the unit file         |

**`$PATH` explained:**
A colon-separated list of directories the shell searches for executables.

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

export PATH="$PATH:/opt/myapp/bin"  # Append a directory to PATH
```

---

### Q31: What is the difference between `.bashrc` and `.bash_profile`?

**Answer:**

| File            | When sourced                                        |
| :-------------- | :-------------------------------------------------- |
| `.bash_profile` | **Login** shells only (SSH, `su -`, console login)  |
| `.bashrc`       | **Interactive non-login** shells (new terminal tab) |

In practice on RHEL, `.bash_profile` sources `.bashrc`:

```bash
# ~/.bash_profile
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

So most users put aliases, functions, and exports in `~/.bashrc` and it works in both cases.

**System-wide equivalents:** `/etc/profile` (login) and `/etc/bashrc` (non-login).

---

## Networking and Security

### Q32: What networking commands should you know?

**Answer:**

**`ip` (replaces `ifconfig` on modern systems):**

```bash
ip addr show                        # Show all interfaces and IPs
ip addr add 192.168.1.10/24 dev eth0    # Add an IP address
ip link set eth0 up                 # Bring interface up
ip link set eth0 down               # Bring interface down
ip route show                       # Show routing table
ip route add default via 192.168.1.1    # Add default gateway
ip neigh show                       # Show ARP table (replaces arp)
```

**`ss` (replaces `netstat`):**

```bash
ss -tlnp                            # TCP listening ports with process names
ss -ulnp                            # UDP listening ports
ss -tunapl                          # All TCP/UDP connections with processes
ss -s                               # Socket statistics summary
```

**DNS utilities:**

```bash
dig google.com                      # DNS lookup (detailed)
dig google.com +short               # Just the IP
nslookup google.com                 # Interactive DNS lookup
host google.com                     # Simple DNS lookup
```

**Connectivity testing:**

```bash
ping -c 4 8.8.8.8                   # Send 4 ICMP packets
traceroute google.com               # Trace the route to a host
mtr google.com                      # Combines ping + traceroute (real-time)
curl -I https://example.com         # Fetch HTTP headers only
curl -v https://example.com         # Verbose HTTP request
wget https://example.com/file.tar.gz    # Download a file
```

**RHEL network configuration (NetworkManager):**

```bash
nmcli device status                 # List network devices
nmcli connection show               # List connections
nmcli connection modify eth0 ipv4.addresses 192.168.1.10/24
nmcli connection up eth0            # Apply changes
```

---

### Q33: How does DNS resolution work on Linux?

**Answer:**

When a program resolves a hostname, the system follows this order:

1. **`/etc/nsswitch.conf`** -- Defines the lookup order (usually `files dns`).
2. **`/etc/hosts`** -- Local static hostname-to-IP mappings. Checked first.
3. **`/etc/resolv.conf`** -- Lists DNS server IPs to query if `/etc/hosts` has no match.

```bash
# /etc/hosts
127.0.0.1   localhost
192.168.1.50  myapp.internal

# /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
search example.com
```

On RHEL 8+, `NetworkManager` manages `/etc/resolv.conf`. Direct edits may be overwritten. Use `nmcli` to set DNS:

```bash
nmcli connection modify eth0 ipv4.dns "8.8.8.8 8.8.4.4"
nmcli connection up eth0
```

---

### Q34: How does `firewalld` work (RHEL firewall)?

**Answer:**

`firewalld` is the default firewall management tool on RHEL 7+. It uses **zones** to define trust levels.

```bash
systemctl status firewalld          # Check firewall status
firewall-cmd --state                # Quick check (running or not)

# Zones
firewall-cmd --get-default-zone     # Usually "public"
firewall-cmd --get-active-zones     # Show active zones and interfaces
firewall-cmd --list-all             # List all rules in default zone

# Allow a service
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent

# Allow a port
firewall-cmd --add-port=8080/tcp --permanent

# Remove a rule
firewall-cmd --remove-service=http --permanent
firewall-cmd --remove-port=8080/tcp --permanent

# Apply changes
firewall-cmd --reload

# List allowed services/ports
firewall-cmd --list-services
firewall-cmd --list-ports
```

The `--permanent` flag writes the rule to disk. Without it, the rule is temporary (lost on reload/reboot). Always pair `--permanent` with `--reload`.

---

### Q35: What is SELinux?

**Answer:**

SELinux (Security-Enhanced Linux) is a mandatory access control (MAC) system built into the RHEL kernel. It adds a layer of security beyond standard file permissions.

**Modes:**

| Mode         | Behavior                                                  |
| :----------- | :-------------------------------------------------------- |
| `enforcing`  | Policies are enforced. Violations are blocked and logged. |
| `permissive` | Policies are not enforced but violations are logged.      |
| `disabled`   | SELinux is completely off.                                |

```bash
getenforce                          # Show current mode
setenforce 0                        # Switch to permissive (temporary)
setenforce 1                        # Switch to enforcing (temporary)
# Permanent change: edit /etc/selinux/config and set SELINUX=enforcing
```

**Contexts (labels):**
Every file, process, and port has an SELinux context: `user:role:type:level`.

```bash
ls -Z /var/www/html/                # Show file contexts
ps -eZ | grep httpd                 # Show process contexts
```

**Common troubleshooting:**

```bash
# A service cannot read a file it should have access to
restorecon -Rv /var/www/html/       # Restore default contexts for the path
setsebool -P httpd_can_network_connect on   # Enable a boolean
getsebool -a | grep httpd           # List SELinux booleans for httpd
ausearch -m avc -ts recent          # Search for recent SELinux denials
sealert -a /var/log/audit/audit.log # Human-readable denial analysis
```

---

### Q36: What is the difference between `su` and `sudo`?

**Answer:**

| Feature      | `su`                                    | `sudo`                                      |
| :----------- | :-------------------------------------- | :------------------------------------------ |
| **Action**   | Switches to another user's shell        | Runs a single command as another user       |
| **Password** | Requires the **target user's** password | Requires **your own** password              |
| **Audit**    | Minimal logging                         | Every command is logged (`/var/log/secure`) |
| **Scope**    | Full shell access as that user          | Granular: specific commands can be allowed  |

```bash
su - root                           # Switch to root (needs root password)
su - devuser                        # Switch to devuser (needs devuser password)
sudo systemctl restart httpd        # Run one command as root (needs your password)
sudo -u postgres psql               # Run as a specific user
sudo -i                             # Open a root shell (like su -)
```

**`/etc/sudoers` (edit with `visudo`):**

```bash
devuser ALL=(ALL) ALL               # User can run any command as any user
%wheel  ALL=(ALL) ALL               # Members of wheel group can sudo
devuser ALL=(ALL) NOPASSWD: ALL     # No password prompt (use with caution)
devuser ALL=(ALL) /usr/bin/systemctl restart httpd  # Only allow specific command
```

---

### Q37: What are `chown` and `chgrp`?

**Answer:**

```bash
chown user file.txt                 # Change file owner
chown user:group file.txt           # Change owner and group
chown -R user:group /opt/app/       # Recursive (all files/subdirs)
chgrp devops file.txt               # Change group only
chgrp -R devops /shared/project/    # Recursive group change
```

New files inherit the user's primary group by default. Use SGID on a directory (Q16) to force group inheritance.

---

## CLI Tools and Utilities

### Q38: How do you use `xargs`?

**Answer:**

`xargs` reads items from stdin and passes them as arguments to a command. It solves the "argument list too long" problem and enables parallel execution.

```bash
# Delete all .tmp files found by find
find /tmp -name "*.tmp" | xargs rm -f

# Handle filenames with spaces (null-delimited)
find /tmp -name "*.tmp" -print0 | xargs -0 rm -f

# Run with a maximum number of arguments per command
echo "a b c d e f" | xargs -n 2 echo
# Output:
# a b
# c d
# e f

# Run in parallel (4 processes)
find /images -name "*.png" | xargs -P 4 -I {} convert {} {}.jpg

# Interactive (ask before each execution)
find /tmp -name "*.log" | xargs -p rm
```

---

### Q39: What does `tee` do?

**Answer:**

`tee` reads from stdin and writes to both stdout **and** one or more files. It lets you see output on screen while saving it to a file.

```bash
# Save command output to file while also printing to terminal
df -h | tee disk_report.txt

# Append instead of overwrite
df -h | tee -a disk_report.txt

# Write to multiple files
dmesg | tee file1.txt file2.txt

# Common use: write to a root-owned file with sudo
echo "192.168.1.10 myhost" | sudo tee -a /etc/hosts
# Note: sudo echo ... > /etc/hosts DOES NOT WORK because the redirect
# is handled by the current (non-root) shell.
```

---

### Q40: What does `watch` do?

**Answer:**

`watch` runs a command repeatedly and displays the output, refreshing at a set interval. Useful for monitoring.

```bash
watch df -h                         # Refresh disk usage every 2 seconds (default)
watch -n 5 free -m                  # Refresh memory usage every 5 seconds
watch -d ss -tlnp                   # Highlight differences between updates
watch "ps aux | grep nginx"         # Watch a piped command (use quotes)
```

---

### Q41: What is `nohup` and how is it different from `&`?

**Answer:**

- **`&`** runs a process in the background of the current shell. If the shell (or SSH session) closes, the process receives `SIGHUP` and typically dies.
- **`nohup`** makes the process immune to `SIGHUP`. Output goes to `nohup.out` by default.

```bash
./long-task.sh &                    # Background, but dies if you log out
nohup ./long-task.sh &              # Background, survives logout
nohup ./long-task.sh > /var/log/task.log 2>&1 &  # Custom log file
```

**`disown`** can also detach an already-running background job from the shell:

```bash
./long-task.sh &
disown %1                           # Detach job 1 from the shell
```

For long-running tasks on servers, prefer `systemd` services or `tmux`/`screen` sessions.

---

### Q42: How do you use `lsof`?

**Answer:**

`lsof` (List Open Files) shows files opened by processes. On Linux, everything is a file (sockets, pipes, devices), so this tool is very versatile.

```bash
lsof -i :80                         # What process is using port 80?
lsof -i :8080 -i :443               # Check multiple ports
lsof -u devuser                     # Files opened by a specific user
lsof -p 1234                        # Files opened by PID 1234
lsof /var/log/messages               # Which process has this file open?
lsof +D /var/log/                    # All open files in a directory
lsof -i TCP -s TCP:LISTEN           # All TCP listening sockets
```

Common use case: "Why can't I unmount this filesystem?"

```bash
lsof +D /mnt/data                   # Find processes using files on /mnt/data
```

---

### Q43: What is `systemd` timer and how does it differ from cron?

**Answer:**

`systemd` timers are the modern alternative to cron for scheduling tasks.

**Advantages over cron:**

- Better logging (via `journalctl`)
- Dependency management (wait for network, other services)
- Runs missed executions (if the system was off)
- Per-unit resource limits (CPU, memory via cgroups)

A timer requires two unit files:

**`/etc/systemd/system/backup.service`:**

```ini
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

**`/etc/systemd/system/backup.timer`:**

```ini
[Unit]
Description=Run Backup Daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl enable --now backup.timer  # Enable and start the timer
systemctl list-timers --all          # List all timers and next run times
journalctl -u backup.service        # Check logs for the service
```

---

### Q44: What is `chroot`?

**Answer:**

`chroot` changes the apparent root directory for a process and its children. The process sees a different `/` and cannot access files outside it.

```bash
chroot /mnt/sysimage /bin/bash      # Common in rescue mode to access installed system
```

**Use cases:**

- **System rescue:** Boot from live media, mount the real root filesystem, then `chroot` into it to fix bootloader, reset passwords, or repair packages.
- **Build isolation:** Create minimal environments for building software.
- **Legacy containment:** Simple process isolation (predates containers).

`chroot` is **not** a security sandbox. Root inside a chroot can break out. Containers (namespaces + cgroups) replaced chroot for isolation.

---

### Q45: What is the difference between `stat`, `file`, and `type`?

**Answer:**

These are three different inspection commands:

**`stat` -- Detailed file metadata:**

```bash
stat /etc/passwd
# Outputs: size, inode, permissions, timestamps (access, modify, change), owner, etc.
```

**`file` -- Identify file type by content (not extension):**

```bash
file /bin/ls
# /bin/ls: ELF 64-bit LSB executable, x86-64

file image.png
# image.png: PNG image data, 800 x 600

file script.sh
# script.sh: Bash script, ASCII text executable
```

**`type` -- Classify a shell command:**

```bash
type ls
# ls is aliased to 'ls --color=auto'

type cd
# cd is a shell builtin

type python3
# python3 is /usr/bin/python3
```

---

### Q46: How do you check and manage running services and open ports?

**Answer:**

This combines several tools covered above into a practical workflow.

```bash
# 1. Check if a service is running
systemctl is-active httpd

# 2. Check what port it listens on
ss -tlnp | grep httpd

# 3. Check if the firewall allows that port
firewall-cmd --list-ports
firewall-cmd --list-services

# 4. Test connectivity from another machine
curl -v http://server-ip:80

# 5. Check if something else is using the port
lsof -i :80
ss -tlnp | grep :80
```

---

### Q47: What is the difference between a process and a thread?

**Answer:**

| Aspect         | Process                                    | Thread                                               |
| :------------- | :----------------------------------------- | :--------------------------------------------------- |
| **Definition** | Independent program in execution           | Lightweight unit of execution within a process       |
| **Memory**     | Has its own memory space                   | Shares memory with other threads in the same process |
| **Creation**   | Heavier (fork)                             | Lighter (pthread_create)                             |
| **Crash**      | Does not affect other processes            | Can crash the entire process                         |
| **IPC**        | Needs explicit mechanisms (pipes, sockets) | Can communicate via shared memory directly           |
| **View**       | `ps aux`                                   | `ps -eLf` (shows threads as LWPs)                    |

```bash
ps -eLf | head                      # List processes with thread count (NLWP column)
```

---

### Q48: What is the difference between a soft reboot and a hard reboot?

**Answer:**

- **Soft reboot:** The OS shuts down services gracefully, syncs filesystems, unmounts, and then reboots. Data integrity is preserved.

```bash
reboot                              # Standard graceful reboot
shutdown -r now                     # Same
systemctl reboot                    # Same via systemctl
shutdown -r +5 "Rebooting in 5 min" # Schedule with warning message
```

- **Hard reboot:** Forces an immediate reboot without graceful shutdown. Risk of data corruption and filesystem damage.

```bash
echo b > /proc/sysrq-trigger       # Immediate kernel reboot (emergency only)
```

**Other shutdown commands:**

```bash
shutdown -h now                     # Halt the system
poweroff                            # Power off
systemctl poweroff                  # Same via systemctl
shutdown -c                         # Cancel a scheduled shutdown
```

---
