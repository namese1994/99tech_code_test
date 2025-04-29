
## 1 · Introduction & Context

- **Infrastructure:** A single Ubuntu 24.04 VM (64 GB SSD), running **only NGINX** as a load balancer for upstream services.  
- **Alert:** Monitoring tools reported **disk usage consistently at ≈99%** over an undetermined period (ranging from minutes to hours or even days).  
- **Risk:** Once disk usage hits 100%, any file-writing operations (NGINX logs, systemd-journald entries, PID files, TLS session tickets…) may fail, causing service disruption or host crashes.  
- **Report Objectives:**  
  1. Present a **systematic troubleshooting approach** to immediately create a safe "breathing room" and isolate root causes.  
  2. Categorize causes based on **three time-based scenarios**:  
     - **Spike** (sudden surge)  - **Creep** (gradual leak)  - **Plateau** (sustained at 99%).  
  3. Provide a **Recovery Plan** and **Preventive Actions**, demonstrating a Senior DevOps mindset: "quick mitigation – prevention of recurrence – proactive monitoring."


## 2 · Troubleshooting Strategy

### 2.1 Immediate Objective — **"Free up ≥5% Disk Space"**

> **Since the VM is *persistently* at 99% disk usage, any new write operations are dangerous.**  
> The first step is to quickly free ~3–4 GB (≈5%) to avoid hitting 100% and allow safe investigation.

| Step | Action | Example Command | Explanation / Risk |
|------|--------|-----------------|--------------------|
| 0-a | Check both **space & inodes** | `df -h && df -i` | If inodes are 100% full ⇒ small files need cleaning, not just large files. |
| 0-b | Identify *largest* files/directories | `du -ahx / | sort -rh | head -20` | Limit view to root filesystem only (`-x`). |
| 0-c | **Truncate** the largest log (typically NGINX or journald) | `sudo truncate -s 0 /var/log/nginx/access.log` | Keeps file descriptor open → safer than deletion, avoids immediate service reload. |
| 0-d | Vacuum journald (if systemd logs >1GB) | `sudo journalctl --vacuum-size=500M` | Quickly reduces space without affecting NGINX. |

> **Expected Result:** `df -h` shows ≤94% usage, and inode usage ≤90%.  
> Once breathing room is created, proceed to root cause analysis.

---

### 2.2 Classify by Monitoring Graph Pattern

| Type | %Disk Graph Behavior | Main Hypothesis | Priority Check Step |
|------|----------------------|-----------------|---------------------|
| **Spike** | Sharp surge to 99% | Large file created recently | `du -ahx / | sort -rh | head` |
| **Creep** | Steady increase 0.5–1% per hour | Log/cache leak | Monitor rapidly growing folders (`ncdu` or `du --time`) |
| **Plateau** | Flat line at 99% | Deleted-but-open files, journald auto-cleanup stuck, reserved blocks, inode exhaustion | `lsof +L1`, `df -i`, `tune2fs -l | grep Reserved` |

---

## 3 · Root Cause Analysis Related to NGINX

---

### 3.1 NGINX Log Files Growing Abnormally (Spike or Creep)

**Symptoms:**  
- Disk % closely follows traffic patterns; each traffic peak results in disk usage increasing slightly.  
- No immediate errors, but latency rises due to I/O wait.

**How to Check:**  
1. **Measure total log directory size:**
   ```bash
   du -sh /var/log/nginx/
   ```
   > If it exceeds 60 GB on a 64 GB SSD, it’s almost certainly the culprit.
2. **Check if logrotate is active:**
   - If a dry-run says "skipping rotation" because the file is too young ➟ logrotate schedule/size settings are inappropriate.

**Quick and Safe Mitigation:**  
- **Truncate and retain the latest 20,000 lines** to preserve recent logs for RCA:
   ```bash
   tail -n 20000 /var/log/nginx/access.log > /tmp/access_tail.log
   sudo truncate -s 0 /var/log/nginx/access.log
   ```
- **Move oversized logs temporarily to EBS/S3** if complete retention is required.

**Prevention:**  
- Configure logrotate to run **daily** and by **size (100 MB)**, keeping 7 compressed rotated logs.

---

### 3.2 Logrotate Failure / Deleted-but-Open Files (Plateau at 99%)

**Symptoms:**  
- `du -sh /var/log/nginx/` shows only a few GB, but `df -h` remains at 99%.  
- Scheduled logrotate runs, but disk usage does not decrease.

**How to Check:**  
1. **List deleted but open files:**
   ```bash
   sudo lsof +L1 | grep nginx
   ```
   > You’ll find lines like `access.log (deleted)` with several GB sizes.
2. **Count open file descriptors on master process:**
   ```bash
   sudo ls -l /proc/$(pidof nginx | cut -d' ' -f1)/fd | wc -l
   ```
   > A very high FD count (>1024) suggests stuck log rotation.

**Quick Mitigation:**  
- **Reload NGINX** safely without downtime:
   ```bash
   systemctl reload nginx
   ```
- **If reload insufficient**, perform a **conditional restart** (only after creating ≥5% free space):
   ```bash
   systemctl restart nginx
   ```

**Prevention:**  
- Add a `postrotate` script in logrotate to `systemctl reload nginx || true`.  
- Set a cronjob every 5 minutes to check `lsof +L1 | grep nginx` and send Slack alerts if any deleted file exceeds 100 MB.

---

### 3.3 Upload Temp Directory Overflow (/var/lib/nginx/body) (Spike)

**Symptoms:**  
- Disk % spikes precisely when uploading features are released, or during POST spam attacks.  
- Users complain of "502 upload error"; `access.log` contains `client_body_temp` failures.

**How to Check:**  
1. **Measure upload temp directory size:**
   ```bash
   du -sh /var/lib/nginx/body/
   ```
2. **Count files and check timestamps:**
   ```bash
   find /var/lib/nginx/body -type f | wc -l
   stat --format='%y :%s' /var/lib/nginx/body/* | head
   ```
   > If thousands of files appear within minutes ➟ upload flood detected.

**Quick Mitigation:**  
- **Delete files older than 30 minutes** (safe for active sessions):
   ```bash
   sudo find /var/lib/nginx/body -type f -mmin +30 -delete
   ```

**Prevention:**  
- Set appropriate `client_max_body_size` (e.g., 20 MB).  
- Implement a cron cleanup (e.g., using `tmpwatch`).  
- Deploy a WAF rule for "request-body anomaly" detection.

---

### 3.4 Unlimited Proxy / FastCGI Cache Growth (Creep)

**Symptoms:**  
- Disk usage increases steadily by 0.5–1% per hour, even during low traffic.  
- `cache_status=MISS` appears abnormally high as the cache fills up.

**How to Check:**  
1. **Measure cache directory size:**
   ```bash
   du -sh /var/cache/nginx/
   ```
2. **Check cache path configuration:**
   ```bash
   grep -n proxy_cache_path /etc/nginx/nginx.conf
   ```
   > If the `max_size` parameter is **missing**, the cache will eventually consume all disk space.

**Quick Mitigation:**  
- **Purge older cache files** to quickly reduce disk usage:
   ```bash
   sudo find /var/cache/nginx -type f -mtime +1 -delete
   ```
- **Delete all cache and reload** if backend can handle it:
   ```bash
   sudo rm -rf /var/cache/nginx/*
   systemctl reload nginx
   ```

**Prevention:**  
- Add `max_size=1g inactive=30m` to `proxy_cache_path`.  
- Set up a Grafana panel for "cache_dir_usage %" and alert if it exceeds 70%.

---

### 3.5 DDoS/Bot Flood Overwhelming Logs (Spike - Sudden Surge)

**Symptoms:**  
- NGINX CPU usage and Requests Per Second (RPS) spike dramatically; disk usage climbs from ~80% to 99% within minutes.  
- `access.log` floods with repetitive IPs and empty User-Agents.

**How to Check:**  
1. **Identify high-load IPs:**
   ```bash
   awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
   ```
2. **Monitor access-log growth rate:**
   ```bash
   watch -n2 'stat -c %s /var/log/nginx/access.log'
   ```
3. **Check concurrent connections:**
   ```bash
   ss -Htan '( sport = :80 or sport = :443 )' | awk '{print $5}' | cut -d':' -f1 | sort | uniq -c | sort -nr | head
   ```

**Emergency Mitigation:**

1. **Rotate or truncate logs immediately to free space:**
   ```bash
   # Save last 100,000 lines for investigation
   tail -n 100000 /var/log/nginx/access.log > /tmp/access_tail.log
   sudo truncate -s 0 /var/log/nginx/access.log
   sudo truncate -s 0 /var/log/nginx/error.log
   ```
   > **Goal:** Instantly free several GBs without service disruption.

2. **Compress and move old logs** (if preservation is needed, and after freeing ≥10% space):
   ```bash
   tar -czf /mnt/diagnostic/ddos_logs_$(date +%F).tar.gz /var/log/nginx/*.log.*
   sudo rm -f /var/log/nginx/*.log.*
   ```
   > **Goal:** Free additional space while preserving forensic evidence.

**Prevention:**  
- Configure NGINX request rate limiting:
   ```nginx
   limit_req_zone $binary_remote_addr zone=one:10m rate=5r/s;
   limit_req zone=one burst=10 nodelay;
   ```
- Deploy WAF solutions (Cloudflare/AWS WAF); set Prometheus alerts if RPS exceeds 3× baseline within 5 minutes.

---

### 3.6 Core Dumps Caused by NGINX Crash (Spike - Isolated)

**Symptoms:**  
- Service interruption; syslog records "segmentation fault (core dumped)".  
- Disk usage suddenly increases by a few GB at the time of crash, then stays constant at 99%.

**How to Check:**  
1. **List core dumps:**
   ```bash
   ls -lh /var/crash/ /var/lib/systemd/coredump/
   ```
2. **Identify the crashed process:**
   ```bash
   coredumpctl list | head
   ```

**Emergency Mitigation:**  
- **Move dumps to S3 or another storage location for later analysis**,  
- **Or delete immediately** (if analysis isn't required):
   ```bash
   sudo rm -rf /var/crash/*
   ```

**Prevention:**  
- Stick to stable NGINX versions; thoroughly test third-party modules in a staging environment before production use.

---

## 4 · Root Cause Analysis Related to Ubuntu

---

### 4.1 systemd-journald Disk Consumption (Creep or Plateau)

**Symptoms:**  
- `du -sh /var/log/journal/` or `journalctl --disk-usage` reports large disk space usage.  
- Disk % slowly increases but remains at ~99% due to journald’s built-in auto-rotation, which may have an overly long retention period (30–60 days).

**How to Check:**  
1. **Measure journald disk usage:**
   ```bash
   journalctl --disk-usage
   ```
2. **Check journald retention settings:**
   ```bash
   grep -n ^SystemMaxUse /etc/systemd/journald.conf
   ```

**Emergency Mitigation:**  
- **Shrink logs to a safe size quickly:**
   ```bash
   sudo journalctl --vacuum-size=500M
   ```
- **Or delete logs older than 3 days:**
   ```bash
   sudo journalctl --vacuum-time=3d
   ```

**Prevention:**  
- Configure `SystemMaxUse=500M` and `RuntimeMaxUse=200M`.  
- Alert via Prometheus if `node_journal_log_bytes` exceeds 70% of its configured limit.

---

### 4.2 Kernel/OS Core Dumps (Spike - Isolated)

**Symptoms:**  
- Large files (1–5 GB) appear in `/var/crash/`; disk usage spikes to 99% after an OS-level crash.  
- `dmesg` logs show OOPS or kernel panic events.

**How to Check:**  
1. **List crash dump files:**
   ```bash
   ls -lh /var/crash/
   ```
2. **Check crash statistics:**
   ```bash
   sudo apport-cli list --crashes
   ```

**Emergency Mitigation:**  
- **Move crash dumps to diagnostic storage:**
   ```bash
   mv /var/crash/* /mnt/diagnostic/
   ```
- **Or delete dumps after collecting needed information:**
   ```bash
   sudo rm -rf /var/crash/*
   ```

**Prevention:**  
- Disable automatic core dump generation if not required:
   - Edit `/etc/default/apport` → set `enabled=0`.
- Actively monitor hardware (SMART monitoring, memory tests) to reduce chances of kernel panic events.


---

## 5 · Comprehensive Preventive Actions

After restoring the system and completing root cause analysis, it is crucial to implement a full set of preventive measures, based on the principle:

> **Proactive Monitoring – Automated Cleanup – Defensive Limits – Clear Operational Procedures**

---

### 5.1 Set Up Early Warning Monitoring & Alerts

- **Monitor disk and inode usage** using Prometheus + Grafana, or Cloud Monitoring tools.
- Configure alert thresholds at safe levels:
  - Disk usage > 80% ⇒ medium severity alert.
  - Disk usage > 90% or inode usage > 90% ⇒ critical severity alert.

---

### 5.2 Properly Configure Log and Cache Lifecycles

- **Enforce disk quotas on cache and log directories** both by size and retention time.
- Example: Rotate logs **daily and by size**, limit cache **maximum size and expiration time**.

---

### 5.3 Schedule Periodic Auto-Cleanup Tasks

- Implement cron jobs or systemd-timers to automatically clean up directories prone to disk exhaustion:
  - `/var/log/`
  - `/var/lib/nginx/body/`
  - `/var/cache/nginx/`
  - `/tmp/`
- Tools like `tmpwatch`, `tmpreaper`, or custom shell scripts can be utilized.

---

### 5.4 Protect Services from Abnormal Growth or Attacks

- Configure **request rate limiting** and **maximum body size** in NGINX: Avoid resource exhaustion during traffic spikes or upload attacks.
- Integrate WAF (Web Application Firewall) or Cloudflare protection if services are exposed to the internet.
- Set dynamic alerts based on RPS (Requests Per Second) baselines to detect DDoS anomalies early.
