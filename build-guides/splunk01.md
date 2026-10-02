# Build Guide: splunk01 (Central Logging and Monitoring)

**Role:** Splunk Enterprise indexer/search head; synthetic service monitoring
**Address:** 192.168.20.30/24 · Gateway 192.168.20.1 · DNS 192.168.20.1
**Platform:** Proxmox VM 130 · 4 vCPU (host) · 8 GB RAM · 80 GB disk · Ubuntu 26.04.1 LTS

## Design decisions
| Decision | Choice | Why |
|---|---|---|
| OS | Ubuntu 26.04.1 LTS | Ubuntu 26.04.1 LTS. Not yet on Splunk's published supported-OS list at install time; validated working in this lab (indexing, forwarding, alerting).|
| Placement | Servers VLAN | Forwarders on the same segment send logs without crossing the firewall |
| License | Splunk Enterprise trial | Includes alerting (not available on the Free license) |
| Monitoring location | Checks run from splunk01, not on the target | An external check still reports when the target host itself fails |

## 1. VM and OS
- Proxmox: q35, OVMF, VirtIO SCSI single, QEMU agent, NIC vmbr0 tag 20, Proxmox NIC firewall off
- Installer: static IPv4, OpenSSH server, LVM. The default root LV used only part of the disk; it was expanded to the full volume during install.
- ISO detached and disk set first in boot order after install

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
sudo timedatectl set-timezone America/New_York
```
Time sync matters: Splunk orders events by timestamp, and incident metrics (time to detect/restore) depend on accurate clocks across hosts.

## 2. SSH hardening
Key-based login from the admin workstation; `/etc/ssh/sshd_config.d/01-hardening.conf` with `PasswordAuthentication no` and `PermitRootLogin no`, verified with `sudo sshd -T`.

## 3. Host firewall (ufw)
```bash
sudo ufw allow from 192.168.99.0/24 to any port 22 proto tcp     # SSH: VPN admins
sudo ufw allow from 192.168.99.0/24 to any port 8000 proto tcp   # Splunk Web: VPN admins
sudo ufw allow from 192.168.20.0/24 to any port 9997 proto tcp   # forwarders: Servers VLAN
sudo ufw allow from 192.168.30.0/24 to any port 9997 proto tcp   # forwarders: Users VLAN
sudo ufw enable
```
pfSense: TCP 8000 added to the `MGMT_PORTS` alias so VPN admins can reach Splunk Web.

## 4. Splunk Enterprise
```bash
cd /tmp
wget -O splunk-<version>-linux-amd64.deb "<download URL>"
sudo dpkg -i /tmp/splunk-*-linux-amd64.deb
sudo /opt/splunk/bin/splunk enable boot-start -systemd-managed 1 -user splunk --accept-license
sudo systemctl start Splunkd
sudo ss -tlnp | grep -E '8000|8089|9997'
```
Splunk runs as the unprivileged `splunk` user under systemd (`Splunkd.service`).

**Configuration (Splunk Web):**
- Receiving enabled on TCP 9997
- Indexes: `linux`, `network`, `windows`, each with max size 20 GB (the default would exceed the 80 GB disk)

## 5. Universal Forwarder on rhel01
```bash
sudo dnf install -y /tmp/splunkforwarder-*.rpm
sudo /opt/splunkforwarder/bin/splunk enable boot-start -systemd-managed 1 -user splunkfwd --accept-license
```
`/opt/splunkforwarder/etc/system/local/outputs.conf`
```ini
[tcpout]
defaultGroup = splunk01

[tcpout:splunk01]
server = 192.168.20.30:9997
```
`/opt/splunkforwarder/etc/system/local/inputs.conf`
```ini
[monitor:///var/log/secure]
index = linux
sourcetype = linux_secure

[monitor:///var/log/messages]
index = linux
sourcetype = syslog

[monitor:///var/log/httpd/access_log]
index = linux
sourcetype = access_combined

[monitor:///var/log/httpd/error_log]
index = linux
sourcetype = apache_error
```

**Log access without root (least privilege):** the forwarder runs as `splunkfwd`, but these logs are root-only. Read access is granted per file with ACLs instead of running the agent as root:
```bash
sudo setfacl -m u:splunkfwd:r /var/log/secure /var/log/messages
sudo setfacl -m u:splunkfwd:rx /var/log/httpd
sudo setfacl -m u:splunkfwd:r /var/log/httpd/access_log /var/log/httpd/error_log
sudo -u splunkfwd head -2 /var/log/secure      # verify
```
Log rotation creates new files without the ACL, so a `setfacl` line was added to the `postrotate` block of `/etc/logrotate.d/rsyslog` and `/etc/logrotate.d/httpd`.
**Known limitation:** package updates may replace these vendor logrotate files. Re-check after updates.

## 6. Synthetic HTTP check
`/usr/local/bin/http-check.sh`
```bash
#!/bin/bash
# Synthetic HTTP check: records status code and response time for rhel01.
TARGET="http://192.168.20.20/"
TS=$(date '+%Y-%m-%dT%H:%M:%S%z')
RESULT=$(curl -s -o /dev/null -m 5 -w '%{http_code} %{time_total}' "$TARGET")
CODE=${RESULT%% *}
TIME=${RESULT##* }
if   [ "$CODE" = "200" ]; then STATE=UP
elif [ "$CODE" = "000" ]; then STATE=DOWN
else                           STATE=DEGRADED
fi
echo "$TS target=rhel01 url=$TARGET status=$CODE response_time=$TIME state=$STATE" >> /var/log/svccheck/rhel01-http.log
```
Runs every minute via `http-check.timer` (`OnCalendar=minutely`). Output uses `key=value` pairs so Splunk extracts fields automatically.

Splunk input (`/opt/splunk/etc/system/local/inputs.conf`):
```ini
[monitor:///var/log/svccheck/rhel01-http.log]
index = linux
sourcetype = svc_check
```

## 7. Alert
| Setting | Value |
|---|---|
| Name | rhel01 HTTP service down |
| Search | `index=linux sourcetype=svc_check target=rhel01 state!=UP` |
| Schedule | Every minute (`* * * * *`), time range last 2 minutes |
| Trigger | Number of results > 0, once |
| Throttle | 10 minutes |
| Action | Add to Triggered Alerts, severity High |

Worst-case detection time is about 2 minutes (1-minute check interval + 1-minute alert schedule).
Email notification is not configured (no mail server in the lab); Triggered Alerts provides the timestamped record.

## 8. Validation
See `validation/splunk01-validation.md`.