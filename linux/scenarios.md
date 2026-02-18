# Linux -- Scenario-Based Questions

Real-world troubleshooting and problem-solving exercises.

---

### Scenario 1: "Connection Refused" Error

**Situation:** You try to curl a web server on port 80 and get `curl: (7) Failed to connect to localhost port 80: Connection refused`.

**How would you troubleshoot this?**

**Answer:**

1.  **Check if the service is running:**

    ```bash
    systemctl status httpd
    # or
    ps -ef | grep httpd
    ```

2.  **Check if it's listening on the correct port:**

    ```bash
    ss -tlnp | grep :80
    # or
    netstat -tlnp | grep :80
    ```

    - If it's not listening, check config files (`/etc/httpd/conf/httpd.conf`).

3.  **Check Firewall (iptables/firewalld):**

    ```bash
    firewall-cmd --list-all
    iptables -L -n
    ```

    - Even if the service is running, the firewall might be blocking the port.

4.  **Check SELinux (if on RHEL/CentOS):**
    ```bash
    sestatus
    # Check audit logs for denials
    grep "denied" /var/log/audit/audit.log
    ```

---

### Scenario 2: Disk Space is Full, but `df -h` shows space available

**Situation:** A user complains they can't write a file due to "No space left on device", but `df -h` shows 50% free space.

**How would you troubleshoot this?**

**Answer:**

There are two common causes:

1.  **Inode Exhaustion:**
    - The filesystem has run out of index nodes (inodes), usually caused by millions of tiny files.
    - **Check:** `df -i`
    - **Fix:** Delete unnecessary small files or directories.

2.  **Deleted Files Held Open:**
    - A large file was deleted (unlinked), but a running process still has a file descriptor open to it. The space won't be freed until the process closes the file or restarts.
    - **Check:** `lsof | grep deleted`
    - **Fix:** Restart the process holding the file (e.g., `systemctl restart rsyslog`), or kill the PID.

---

### Scenario 3: High Load Average

**Situation:** The system is sluggish, and `uptime` shows a load average of 25.0 on a 4-core machine.

**How would you identify the cause?**

**Answer:**

1.  **Run `top` or `htop`:**
    - Check **%CPU**: Is a specific process eating 100% CPU? (User space issue)
    - Check **%wa** (iowait): Is the CPU waiting for disk/network I/O? (System/Hardware issue)

2.  **If High CPU:**
    - Identify the process (e.g., a runaway script or java app).
    - `strace -p <PID>` to see what system calls it's making.
    - `renice` it or `kill` it.

3.  **If High I/O Wait (%wa > 20%):**
    - Use `iotop` to find which process is doing heavy disk I/O.
    - Use `iostat -x 1` to check disk utilization/saturation.
    - Check for disk errors in `/var/log/messages` or `dmesg`.

---

### Scenario 4: Forgot Root Password (RHEL 7/8/9)

**Situation:** You are locked out of a server and need to reset the root password.

**How would you do this?**

**Answer:**

1.  **Reboot** the server.
2.  At the GRUB menu, press **'e'** to edit the kernel boot parameters.
3.  Find the line starting with `linux16` or `linux` (depending on version).
4.  Append `rd.break` to the end of the line.
5.  Press **Ctrl+X** to boot.
6.  The system boots into emergency mode with a read-only root filesytem.
7.  **Remount root as read-write:**
    ```bash
    mount -o remount,rw /sysroot
    ```
8.  **Chroot into the system:**
    ```bash
    chroot /sysroot
    ```
9.  **Reset password:**
    ```bash
    passwd root
    ```
10. **Enable SELinux relabeling (Critical on RHEL):**
    ```bash
    touch /.autorelabel
    ```
11. **Exit and Reboot:**
    ```bash
    exit
    exit
    ```

---

### Scenario 5: SSH is slow to connect

**Situation:** It takes 10-20 seconds to get a password prompt when SSHing into a server. Once logged in, it's fast.

**How would you troubleshoot this?**

**Answer:**

This is almost always a DNS reverse lookup timeout.

1.  **Debug client-side:**

    ```bash
    ssh -vvv user@host
    ```

    - Look for hangs at "debug1: expecting SSH2_MSG_KEX_DH_GEX_GROUP".

2.  **Fix on Server:**
    - Edit `/etc/ssh/sshd_config`.
    - Set `UseDNS no`.
    - Restart sshd: `systemctl restart sshd`.

3.  **Alternative:** Fix the DNS configuration in `/etc/resolv.conf` so the server can resolve the client's IP, but disabling `UseDNS` is the standard fix for internal servers.

---

### Scenario 6: Cannot delete a file (even as root)

**Situation:** You try to delete a file as root: `rm -f file.txt`, but get "Operation not permitted". Permissions are `777`.

**How would you troubleshoot this?**

**Answer:**

The file likely has the **immutable bit** set.

1.  **Check attributes:**

    ```bash
    lsattr file.txt
    # Output: ----i----------- file.txt
    ```

2.  **Remove immutable bit:**

    ```bash
    chattr -i file.txt
    ```

3.  **Now delete it:**
    ```bash
    rm file.txt
    ```

_Note: This is often used by security hardening tools or malware to prevent file modification._

---

### Scenario 7: What is a Zombie Process?

**Situation:** `top` shows 5 zombie processes.

**Answer:**

- **Definition:** A process that has completed execution (exit status 0 or error) but still has an entry in the process table.
- **Cause:** The parent process failed to read the child's exit status (failed to call `wait()`).
- **Impact:** Zombies don't consume CPU or RAM, but they consume a PID. If the system runs out of PIDs, no new processes can start.
- **Fix:**
  - You **cannot** kill a zombie directly (it's already dead - `kill -9` won't work).
  - You must kill the **Parent Process** (`PPID`). This forces the `init` process (PID 1) to adopt the zombies and clean them up automatically.

---

### Scenario 8: Filesystem is 100% full

**Situation:** A production server's `/var` partition is 100% full. Services are failing to write logs.

**How would you free space immediately?**

**Answer:**

```bash
# 1. Find the largest files
du -ahx /var | sort -rh | head -20

# 2. Common culprits
du -sh /var/log/*          # old logs
du -sh /var/cache/yum/*    # package cache
du -sh /var/tmp/*          # temp files
```

**Quick fixes:**

| Action                        | Command                                    |
| ----------------------------- | ------------------------------------------ |
| Clear old logs                | `journalctl --vacuum-size=100M`            |
| Compress large logs           | `gzip /var/log/messages-*`                 |
| Remove package cache          | `yum clean all`                            |
| Truncate a running log (safe) | `> /var/log/large-file.log`                |
| Find and remove old files     | `find /var/tmp -type f -mtime +30 -delete` |

**Do not use `rm` on a log file that a process has open** -- use truncate (`>`) instead. Deleting an open file does not free space until the process releases it.

**Long-term fix:** Set up log rotation in `/etc/logrotate.d/` and monitor disk usage with alerts.

---

### Scenario 9: A service fails to start

**Situation:** After a reboot, `httpd` does not start. `systemctl status httpd` shows "failed".

**How would you troubleshoot this?**

**Answer:**

```bash
# 1. Check the status output for clues
systemctl status httpd -l

# 2. Read the full journal for the service
journalctl -u httpd --no-pager -n 50

# 3. Check the config syntax
httpd -t
# or for nginx:
nginx -t
```

**Common causes and fixes:**

| Cause                            | How to confirm                           | Fix                                                             |
| -------------------------------- | ---------------------------------------- | --------------------------------------------------------------- | ----------------------------------------- |
| Config syntax error              | `httpd -t` shows the error               | Fix the config file                                             |
| Port already in use              | `ss -tlnp                                | grep :80` shows another process                                 | Stop the other process or change the port |
| Missing SSL certificate          | Journal mentions "certificate not found" | Fix the path in ssl.conf                                        |
| SELinux denial                   | `grep denied /var/log/audit/audit.log`   | `setsebool` or `semanage` to allow                              |
| Permission issue on DocumentRoot | `ls -lZ /var/www/html`                   | Fix ownership with `chown` or SELinux context with `restorecon` |
| Dependency service not running   | Journal says "waiting for network"       | Check `Requires=` and `After=` in the unit file                 |

---

### Scenario 10: A cron job is not running

**Situation:** You set up a cron job but it never executes.

**How would you troubleshoot this?**

**Answer:**

```bash
# 1. Verify the crontab entry
crontab -l                       # current user
crontab -l -u username           # specific user
cat /etc/crontab                 # system crontab

# 2. Check if crond is running
systemctl status crond

# 3. Check cron logs
grep CRON /var/log/cron
```

**Common causes:**

| Cause                         | Fix                                                                  |
| ----------------------------- | -------------------------------------------------------------------- |
| Wrong cron syntax             | Use `crontab.guru` to validate the schedule                          |
| Script not executable         | `chmod +x /path/to/script.sh`                                        |
| Missing full path to commands | Cron has a minimal `PATH`. Use `/usr/bin/python3` not `python3`      |
| Environment variables missing | Set them in the crontab or source a profile at the top of the script |
| Output not captured           | Redirect output: `* * * * * /script.sh >> /var/log/myjob.log 2>&1`   |
| Wrong user                    | Ensure the crontab belongs to the right user (`crontab -u user -e`)  |
| SELinux blocking              | Check `/var/log/audit/audit.log`                                     |

---

### Scenario 11: OOM Killer terminated a process

**Situation:** A Java application was running fine, then suddenly disappeared. No crash logs in the application directory.

**How would you confirm and fix this?**

**Answer:**

```bash
# 1. Check if OOM Killer terminated it
dmesg | grep -i "oom\|killed"
# or
journalctl -k | grep -i oom

# 2. Check which process was killed
grep "Killed process" /var/log/messages
```

**What is OOM Killer?**
When the system runs out of memory, the kernel's Out-Of-Memory (OOM) Killer selects and terminates a process to free RAM. It picks the process with the highest "oom_score".

**Fixes:**

| Action                     | How                                          |
| -------------------------- | -------------------------------------------- | ----- |
| Add more RAM / Swap        | Physical upgrade or `mkswap` + `swapon`      |
| Limit application memory   | Set JVM heap: `-Xmx512m`                     |
| Protect critical processes | `echo -1000 > /proc/<PID>/oom_score_adj`     |
| Use systemd limits         | `MemoryMax=1G` in the unit file              |
| Find the memory leak       | `top` sorted by `%MEM`, `ps aux --sort=-%mem | head` |

---

### Scenario 12: SELinux is blocking an application

**Situation:** You deployed a web app that reads files from `/opt/myapp/data/`. It works when SELinux is set to Permissive but fails in Enforcing mode.

**How would you fix this without disabling SELinux?**

**Answer:**

```bash
# 1. Confirm SELinux is the issue
grep "denied" /var/log/audit/audit.log | tail -10
# or use the audit2why tool
ausearch -m avc -ts recent | audit2why

# 2. Check the current file context
ls -lZ /opt/myapp/data/

# 3. Set the correct SELinux context
semanage fcontext -a -t httpd_sys_content_t "/opt/myapp/data(/.*)?"
restorecon -Rv /opt/myapp/data/

# 4. If a non-standard port is needed
semanage port -a -t http_port_t -p tcp 8443
```

**Key SELinux commands:**

| Command                                     | Purpose                                              |
| ------------------------------------------- | ---------------------------------------------------- |
| `getenforce`                                | Check current mode (Enforcing/Permissive/Disabled)   |
| `setenforce 0`                              | Temporarily set Permissive (resets on reboot)        |
| `sestatus`                                  | Detailed SELinux status                              |
| `restorecon -Rv /path`                      | Reset file contexts to policy defaults               |
| `audit2allow -a`                            | Generate policy module from denials (use cautiously) |
| `setsebool -P httpd_can_network_connect on` | Enable a boolean permanently                         |

---

### Scenario 13: Server cannot reach external network

**Situation:** A server can ping its gateway but cannot reach the internet (`ping 8.8.8.8` times out).

**How would you troubleshoot this?**

**Answer:**

```bash
# 1. Check IP and interface status
ip a
ip link show

# 2. Check the default route
ip r
# Should show: default via <gateway-ip> dev <interface>

# 3. Ping the gateway
ping -c 3 <gateway-ip>

# 4. Ping an external IP (bypasses DNS)
ping -c 3 8.8.8.8

# 5. If ping to IP works but DNS does not
cat /etc/resolv.conf
nslookup google.com
dig google.com
```

**Decision tree:**

| Symptom                        | Likely cause                       | Fix                                                                    |
| ------------------------------ | ---------------------------------- | ---------------------------------------------------------------------- |
| No IP address                  | Interface not configured           | `nmcli con up <connection>` or check `/etc/sysconfig/network-scripts/` |
| Cannot ping gateway            | Wrong subnet or cable issue        | Check IP/mask, check physical connection                               |
| Can ping gateway, not internet | Routing or upstream firewall issue | Check `ip route`, contact network team                                 |
| Can ping IP, not hostname      | DNS misconfiguration               | Fix `/etc/resolv.conf` or `nmcli con mod <conn> ipv4.dns "8.8.8.8"`    |

---

### Scenario 14: Permission Denied on an Executable Script

**Situation:** You wrote a bash script at `/opt/scripts/deploy.sh`. Running `./deploy.sh` gives `Permission denied`, but `cat deploy.sh` works fine.

**How would you fix this?**

**Answer:**

```bash
# 1. Check current permissions
ls -l /opt/scripts/deploy.sh
# -rw-r--r--. 1 devuser devops ... deploy.sh   (no execute bit)

# 2. Add execute permission
chmod +x /opt/scripts/deploy.sh

# 3. Alternatively, run it through the interpreter directly
bash /opt/scripts/deploy.sh
```

**If `chmod +x` itself fails:**

| Cause                     | How to confirm                         | Fix                                          |
| ------------------------- | -------------------------------------- | -------------------------------------------- |
| Not the file owner        | `ls -l` shows a different user         | `sudo chown devuser deploy.sh` or use `sudo` |
| Filesystem mounted noexec | `mount \| grep /opt` shows `noexec`    | Remount: `mount -o remount,exec /opt`        |
| SELinux denial            | `grep denied /var/log/audit/audit.log` | `restorecon -v /opt/scripts/deploy.sh`       |
| Immutable attribute       | `lsattr deploy.sh` shows `i` flag      | `chattr -i deploy.sh`                        |

---

### Scenario 15: Extending a Logical Volume That Is Running Out of Space

**Situation:** The `/var` partition sits on LVM and is 95% full. The volume group has free space. Extend it online without downtime.

**How would you do this?**

**Answer:**

```bash
# 1. Check current state
df -hT /var                          # Confirm it's LVM-backed
lvs                                  # List logical volumes
vgs                                  # Check free space in the VG

# 2. Extend the logical volume by 10G
lvextend -L +10G /dev/myvg/var_lv

# 3. Grow the filesystem to fill the new space
# For XFS (RHEL default):
xfs_growfs /var

# For ext4:
resize2fs /dev/myvg/var_lv

# 4. Verify
df -h /var
```

**If the VG has no free space, add a new disk:**

```bash
pvcreate /dev/sdc                    # Initialize new disk as PV
vgextend myvg /dev/sdc               # Add PV to existing VG
lvextend -L +10G /dev/myvg/var_lv    # Now extend the LV
xfs_growfs /var
```

Key point: XFS can only grow, not shrink. ext4 can do both (but shrinking requires unmounting first).

---

### Scenario 16: Log Rotation Not Working

**Situation:** `/var/log/myapp/app.log` keeps growing and is now 15 GB. Logrotate is configured but the file never rotates.

**How would you troubleshoot this?**

**Answer:**

```bash
# 1. Check the logrotate config
cat /etc/logrotate.d/myapp

# 2. Do a dry run to see what would happen
logrotate -d /etc/logrotate.d/myapp

# 3. Force rotation manually
logrotate -f /etc/logrotate.d/myapp

# 4. Check logrotate status file
cat /var/lib/logrotate/logrotate.status | grep myapp
```

**Common causes:**

| Cause                             | Fix                                                      |
| --------------------------------- | -------------------------------------------------------- |
| Config not in `/etc/logrotate.d/` | Move the config file there                               |
| Wrong path in config              | Fix the path to match the actual log file                |
| `copytruncate` missing            | If the app does not reopen log files, add `copytruncate` |
| Missing `missingok` directive     | Add it so logrotate does not error on missing files      |
| Logrotate status file corrupt     | Delete `/var/lib/logrotate/logrotate.status` and re-run  |
| SELinux blocking                  | Check audit log and fix context                          |

**Example working config:**

```
/var/log/myapp/app.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

---

### Scenario 17: NFS Mount Hangs or Goes Stale

**Situation:** An NFS-mounted directory (`/mnt/shared`) shows `Stale file handle` errors. Any `ls` on the mount hangs.

**How would you recover?**

**Answer:**

```bash
# 1. Check NFS mount status
mount | grep nfs
df -hT                               # This may also hang if mount is stale

# 2. Force unmount the stale mount
umount -f /mnt/shared                # Force unmount
# If that fails:
umount -l /mnt/shared                # Lazy unmount (detach immediately, clean up later)

# 3. Verify the NFS server is reachable
ping nfs-server
showmount -e nfs-server              # List exports from the server

# 4. Re-mount
mount -t nfs nfs-server:/export/shared /mnt/shared

# 5. For persistent mounts, ensure /etc/fstab has proper options
```

**Recommended fstab options for NFS:**

```
nfs-server:/export/shared  /mnt/shared  nfs  defaults,soft,timeo=10,retrans=3,nofail  0  0
```

| Option    | Purpose                                                        |
| --------- | -------------------------------------------------------------- |
| `soft`    | Return error instead of hanging indefinitely on server failure |
| `timeo`   | Timeout in deciseconds before retrying                         |
| `retrans` | Number of retries before returning an error                    |
| `nofail`  | Do not block boot if the NFS server is unavailable             |

---

### Scenario 18: System Is Thrashing on Swap

**Situation:** A server is extremely slow. `free -h` shows RAM is nearly full, and swap usage is high. `top` shows constant swap in/out (si/so columns).

**How would you resolve this?**

**Answer:**

```bash
# 1. Identify what is consuming memory
ps aux --sort=-%mem | head -15
top -o %MEM

# 2. Check swap usage per process
for pid in $(ls /proc/ | grep -E '^[0-9]+$'); do
    awk '/VmSwap/{print "'$pid'", $2, $3}' /proc/$pid/status 2>/dev/null
done | sort -k2 -rn | head -10

# 3. Check the swappiness value
cat /proc/sys/vm/swappiness
```

**Fixes:**

| Action                               | Command/Method                                            |
| ------------------------------------ | --------------------------------------------------------- |
| Kill the offending process           | `kill <PID>` or restart the service                       |
| Reduce swappiness                    | `sysctl vm.swappiness=10` (persist in `/etc/sysctl.conf`) |
| Add more RAM                         | Physical upgrade or increase VM allocation                |
| Set memory limits on services        | `MemoryMax=2G` in the systemd unit file                   |
| Clear swap (force pages back to RAM) | `swapoff -a && swapon -a` (only if RAM has room)          |

Swappiness of 10 means the kernel prefers to evict filesystem cache over swapping application pages. Default 60 is too aggressive for most servers.

---

### Scenario 19: Filesystem Mounted as Read-Only

**Situation:** A server suddenly starts returning "Read-only file system" errors when you try to create or modify files, but the system is still running.

**How would you fix this?**

**Answer:**

```bash
# 1. Confirm the mount is read-only
mount | grep " / "
# Look for "ro" in the options

# 2. Check dmesg for filesystem errors
dmesg | tail -50
# Look for: "EXT4-fs error" or "XFS ... Corruption" messages
```

**What happened:** The kernel detected filesystem corruption or disk I/O errors and remounted the filesystem as read-only to prevent further damage.

```bash
# 3. Try remounting as read-write
mount -o remount,rw /

# 4. If that fails, check disk health
smartctl -a /dev/sda                  # SMART data (requires smartmontools)
fsck /dev/sda1                        # Filesystem check (requires unmount or single-user/rescue mode)
```

**If persistent:**

- The disk may be failing. Check SMART data and `dmesg` for I/O errors.
- Boot into rescue mode and run `fsck` on the affected partition.
- Replace the disk if hardware failure is confirmed.
- Restore from backup if corruption is severe.

---

### Scenario 20: User Account is Locked Out

**Situation:** A user reports they cannot log in. SSH says "Permission denied" even with the correct password.

**How would you troubleshoot?**

**Answer:**

```bash
# 1. Check if the account is locked
passwd -S username
# "LK" or "L" means locked
# "PS" means password is set (normal)

# 2. Check if the account expired
chage -l username
# Look at "Account expires" and "Password expires"

# 3. Check /etc/shadow for locked indicator
grep username /etc/shadow
# A "!" or "!!" at the start of the password hash means locked

# 4. Check SSH logs for specifics
grep username /var/log/secure | tail -20

# 5. Check if pam_faillock locked them out (too many failed attempts)
faillock --user username
```

**Fixes:**

| Cause                      | Fix                                               |
| -------------------------- | ------------------------------------------------- |
| Account locked             | `passwd -u username` or `usermod -U username`     |
| Too many failed logins     | `faillock --user username --reset`                |
| Password expired           | `chage -d 0 username` (force reset on next login) |
| Account expired            | `chage -E -1 username` (remove expiration)        |
| Shell set to /sbin/nologin | `usermod -s /bin/bash username`                   |
| Not in AllowUsers (sshd)   | Add to `AllowUsers` in `/etc/ssh/sshd_config`     |

---

### Scenario 21: File Descriptor Exhaustion

**Situation:** An application logs "Too many open files" errors and starts rejecting connections.

**How would you troubleshoot and fix this?**

**Answer:**

```bash
# 1. Check the current limit for the process
cat /proc/<PID>/limits | grep "Max open files"

# 2. Count how many file descriptors the process has open
ls /proc/<PID>/fd | wc -l
# or
lsof -p <PID> | wc -l

# 3. Check system-wide limits
cat /proc/sys/fs/file-nr
# Format: <allocated> <free> <max>
# Example: 5120  0  262144

# 4. Check what files/sockets are being held
lsof -p <PID> | head -30
```

**Fixes:**

**Temporary (for running process):**

```bash
prlimit --pid <PID> --nofile=65535:65535
```

**Permanent (per service via systemd):**

```ini
# In the unit file [Service] section
LimitNOFILE=65535
```

**Permanent (system-wide):**

```bash
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535

# /etc/sysctl.conf
fs.file-max = 262144
```

Then reload: `sysctl -p` and restart the service.

If the process is leaking file descriptors (count keeps growing), that is a bug in the application and needs a code fix.

---

### Scenario 22: System Clock is Drifting

**Situation:** Logs from this server show timestamps several minutes off compared to other servers. Kerberos/Active Directory authentication is failing because of time skew.

**How would you fix this?**

**Answer:**

```bash
# 1. Check current time and timezone
timedatectl

# 2. Check if NTP sync is active
timedatectl | grep "NTP"
# Should show: NTP enabled: yes, NTP synchronized: yes

# 3. If chronyd (RHEL default NTP client) is not running
systemctl status chronyd
systemctl enable --now chronyd

# 4. Force an immediate sync
chronyc makestep

# 5. Check NTP sources and their status
chronyc sources -v
chronyc tracking
```

**If chronyd is running but not syncing:**

```bash
# Check that NTP servers are configured
cat /etc/chrony.conf
# Should have lines like: server time.google.com iburst

# Check firewall allows NTP (UDP 123)
firewall-cmd --list-services | grep ntp
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload
```

---

### Scenario 23: Port Conflict After Package Upgrade

**Situation:** After upgrading a package, a service fails to start because another process is already using its port.

**How would you resolve this?**

**Answer:**

```bash
# 1. Find what is using the port
ss -tlnp | grep :8080
# Output: LISTEN  0  128  *:8080  *:*  users:(("java",pid=4521,fd=12))

# 2. Get details about the conflicting process
ps -p 4521 -o pid,user,cmd

# 3. Options:
```

| Option                                       | Command                                             |
| -------------------------------------------- | --------------------------------------------------- |
| Stop the conflicting process                 | `kill 4521` or `systemctl stop conflicting-service` |
| Change the new service's port                | Edit its config and update `firewall-cmd` rules     |
| Check if it is the same service (duplicate)  | `systemctl list-units \| grep <service>`            |
| Look for leftover processes from old install | `ps aux \| grep <process-name>`                     |

A common cause is a service that was running from a manual install (e.g., from `/opt/`) while the package manager installs its own copy to `/usr/`.

---

### Scenario 24: Systemd Service Starts Too Early

**Situation:** Your application service starts before the database it depends on is ready. The app crashes on boot, but a manual `systemctl restart myapp` after 10 seconds works fine.

**How would you fix this?**

**Answer:**

Edit the service unit file:

```bash
systemctl edit myapp.service         # Creates an override file
```

Add dependency and ordering directives:

```ini
[Unit]
Requires=postgresql.service
After=postgresql.service
After=network-online.target
Wants=network-online.target
```

| Directive   | Purpose                                                      |
| ----------- | ------------------------------------------------------------ |
| `After=`    | Start myapp only after the listed units have started         |
| `Requires=` | If the dependency fails, myapp is also stopped               |
| `Wants=`    | Weaker dependency. myapp starts even if the dependency fails |

If the database takes time to initialize after its service is "active":

```ini
[Service]
ExecStartPre=/bin/sleep 5
# or better: use a readiness check script
ExecStartPre=/opt/scripts/wait-for-db.sh
Restart=on-failure
RestartSec=5
```

Reload after editing:

```bash
systemctl daemon-reload
systemctl restart myapp
```

---

### Scenario 25: Recovering a Deleted File That Is Still Open

**Situation:** Someone accidentally deleted a large, critical log file, but the process that was writing to it is still running. The data still exists in memory.

**How would you recover it?**

**Answer:**

```bash
# 1. Find the process that still has the file open
lsof | grep deleted | grep "filename"
# Example output:
# rsyslogd 1234 root 5w REG 8,1 52428800 131073 /var/log/important.log (deleted)

# 2. The file descriptor is still accessible via /proc
cat /proc/1234/fd/5 > /var/log/important.log.recovered

# 3. Verify the recovered file
wc -l /var/log/important.log.recovered
tail /var/log/important.log.recovered
```

Why this works: `rm` only removes the directory entry (the name). The actual data blocks are not freed until the last file descriptor referencing the inode is closed. Since the process still has it open, the data is intact in `/proc/<PID>/fd/<FD>`.

After recovery, restart the service so it creates a fresh file at the original path.

---

### Scenario 26: Kernel Upgrade Breaks Boot

**Situation:** After a `dnf update` that included a kernel upgrade, the server fails to boot. It drops into a GRUB rescue prompt or kernel panic.

**How would you fix this?**

**Answer:**

**Option 1: Boot the previous kernel from GRUB:**

1. At the GRUB menu, select **"Advanced options"** or a previous kernel entry.
2. The system boots with the old, working kernel.
3. Investigate why the new kernel fails (missing drivers, incompatible module).

**Option 2: Set the default kernel to the old one:**

```bash
# After booting the old kernel
grub2-editenv list                           # Show current default
grubby --info=ALL                            # List all installed kernels
grubby --set-default /boot/vmlinuz-<old-version>
# or by index:
grub2-set-default 1                          # Set the second entry as default
grub2-mkconfig -o /boot/grub2/grub.cfg       # Regenerate GRUB config
```

**Option 3: Boot from rescue media if GRUB is broken:**

```bash
# 1. Boot from RHEL ISO or rescue USB
# 2. Mount the root filesystem
mount /dev/sda2 /mnt/sysimage
mount /dev/sda1 /mnt/sysimage/boot
# 3. Chroot
chroot /mnt/sysimage
# 4. Reinstall GRUB
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg
# 5. Exit and reboot
```

---

### Scenario 27: DNS Resolution Works for Some Domains But Not Others

**Situation:** `dig google.com` returns results, but `dig internal.company.com` times out. Both used to work.

**How would you troubleshoot?**

**Answer:**

```bash
# 1. Check configured DNS servers
cat /etc/resolv.conf
nmcli device show | grep DNS

# 2. Query specific DNS servers directly
dig internal.company.com @8.8.8.8          # Against public DNS (should fail for internal)
dig internal.company.com @10.0.0.53        # Against internal DNS

# 3. Check search domain
grep search /etc/resolv.conf
# Should include: search company.com

# 4. Check DNS server order
# /etc/resolv.conf lists nameservers in priority order
# If the internal DNS is missing or comes after a public one that cannot resolve internal names,
# internal resolution will fail.
```

**Common causes:**

| Cause                                   | Fix                                                       |
| --------------------------------------- | --------------------------------------------------------- |
| Internal DNS server missing from config | Add via `nmcli con mod eth0 ipv4.dns "10.0.0.53 8.8.8.8"` |
| DNS server order wrong                  | Internal DNS should come first                            |
| Search domain missing                   | `nmcli con mod eth0 ipv4.dns-search "company.com"`        |
| Internal DNS server is down             | Check connectivity to port 53: `dig @10.0.0.53 +short`    |
| Firewall blocking DNS                   | Check port 53/udp and 53/tcp                              |

---

### Scenario 28: Rsync Backup Fails Midway on Large Transfer

**Situation:** A nightly rsync backup of 500 GB fails partway through with `Connection reset by peer` or `broken pipe`.

**How would you make it reliable?**

**Answer:**

```bash
# 1. Use partial and progress flags so rsync can resume
rsync -avzP --partial --partial-dir=.rsync-partial \
  /data/ user@backup-server:/backup/data/

# 2. Wrap in a retry loop
#!/bin/bash
MAX_RETRIES=5
COUNT=0
while [ $COUNT -lt $MAX_RETRIES ]; do
    rsync -avzP --partial --timeout=60 /data/ user@backup:/backup/data/ && break
    COUNT=$((COUNT + 1))
    echo "Retry $COUNT/$MAX_RETRIES..."
    sleep 30
done

# 3. Keep the SSH connection alive
# In ~/.ssh/config or /etc/ssh/ssh_config:
# Host backup-server
#     ServerAliveInterval 60
#     ServerAliveCountMax 3
```

**Common causes of mid-transfer failure:**

| Cause                     | Fix                                            |
| ------------------------- | ---------------------------------------------- |
| SSH timeout               | `ServerAliveInterval 60` in SSH config         |
| Network instability       | Use `--partial` + retry loop                   |
| Server-side disk full     | Check `df -h` on the backup server             |
| Max SSH sessions exceeded | Increase `MaxSessions` in remote `sshd_config` |
| Bandwidth saturation      | `rsync --bwlimit=50000` (limit to 50 MB/s)     |

---

### Scenario 29: `sudo` Does Not Work for a User

**Situation:** A user runs `sudo systemctl restart httpd` and gets `user is not in the sudoers file. This incident will be reported.`

**How would you fix this?**

**Answer:**

```bash
# 1. Log in as root (or another sudo-capable user)

# 2. Check if the user is in the wheel group (RHEL convention)
id username
groups username

# 3. Add the user to wheel
usermod -aG wheel username

# 4. Verify wheel group has sudo access
grep wheel /etc/sudoers
# Should contain: %wheel  ALL=(ALL)  ALL

# 5. If the line is commented out, edit with visudo
visudo
# Uncomment: %wheel  ALL=(ALL)  ALL
```

The user must log out and back in for group changes to take effect. Verify with `id` after re-login.

**If you need more granular access (not full sudo):**

```bash
# In visudo, add specific command access:
username ALL=(ALL) /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd
```

---

### Scenario 30: High I/O Wait Caused by Journald

**Situation:** A server has high `%wa` in `top`, and `iotop` shows `systemd-journald` doing heavy writes.

**How would you fix this?**

**Answer:**

```bash
# 1. Check journal size
journalctl --disk-usage

# 2. Shrink the journal immediately
journalctl --vacuum-size=200M
journalctl --vacuum-time=7d

# 3. Set permanent limits
# Edit /etc/systemd/journald.conf:
# [Journal]
# SystemMaxUse=500M
# SystemMaxFileSize=50M
# MaxRetentionSec=2week
# RateLimitIntervalSec=30s
# RateLimitBurst=1000

# 4. Restart journald
systemctl restart systemd-journald
```

**If a specific service is flooding the journal:**

```bash
# Find which service is writing the most
journalctl --since "1 hour ago" | awk '{print $5}' | sort | uniq -c | sort -rn | head

# Fix the noisy service or suppress its verbose output
# In the unit file:
# [Service]
# StandardOutput=null
# StandardError=null
```

**Additional checks:**

- Make sure the disk can handle the I/O (SSD vs HDD).
- If persistent journal (`/var/log/journal/`) is on a slow partition, consider moving it or using volatile storage (`Storage=volatile` in `journald.conf`).

---
