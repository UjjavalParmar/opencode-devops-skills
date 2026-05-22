---
name: linux-debug
description: Linux host triage — systemd unit failures, journalctl, high CPU/MEM/IO, disk full, OOM, file-descriptor limits, zombie processes, cgroup/limits issues. NOT for container internals (container-debug) or in-cluster networking (network-debug).
---

# Triage order (read-only)
1. `uptime` ; `w` ; `dmesg -T | tail -50`
2. `systemctl --failed`
3. `journalctl -p err -b --since "1 hour ago" --no-pager`
4. `top -b -n1 -o %CPU | head -20` ; `top -b -n1 -o %MEM | head -20`
5. `df -hT` ; `df -i` ; `du -xhd1 / 2>/dev/null | sort -h | tail`
6. `free -h` ; `vmstat 1 5`
7. `ss -tlnp` ; `ss -s`
8. `iostat -xz 1 3` (sysstat)

# Symptom → command
- **OOM killer fired** → `dmesg -T | grep -i 'killed process'` ; `journalctl -k --since "1 hour ago" | grep -i oom`
- **Disk full** → `df -h` then `du -xhd1 <mount> | sort -h`; check `lsof | grep deleted` for held inodes
- **Unit crashing** → `systemctl status <U>` ; `journalctl -u <U> -e --no-pager`
- **High load, low CPU** → IO wait: `iostat -xz 1` ; `iotop -boP -n3`
- **FD exhaustion** → `lsof -p <PID> | wc -l` ; `cat /proc/<PID>/limits`
- **Port in use** → `ss -tlnp | grep :<PORT>` ; `fuser -n tcp <PORT>`
- **Zombie** → `ps -eo stat,pid,ppid,cmd | awk '$1 ~ /Z/'`

# Mutations
- Restart unit: `systemctl restart <U>` — rollback: `systemctl stop <U>` or revert config + reload
- Kill: `kill -TERM <PID>` then `-KILL` only if unresponsive
- Truncate log (only if disk-full emergency): `: > /var/log/<F>` — irreversible; prefer `logrotate -f`
