# Runbook: rhel01 Backup and Restore

## Scope
Backs up web content and key configuration: `/srv/web`, `/etc/httpd/conf.d`, `/etc/ssh/sshd_config.d`, `/etc/sudoers.d`, `/etc/fstab`.

## Schedule
Daily at 02:00 via `lab-backup.timer` (`Persistent=true` runs a missed backup after downtime). 7-day retention.
Destination: `/backup` (logical volume `vg_data/lv_backup`).

## Components
`/usr/local/bin/lab-backup.sh`
```bash
#!/bin/bash
# Backs up web content and key configuration from rhel01.
set -euo pipefail
DEST=/backup
STAMP=$(date +%Y%m%d-%H%M%S)
ARCHIVE="$DEST/rhel01-$STAMP.tar.gz"
tar -czf "$ARCHIVE" /srv/web /etc/httpd/conf.d /etc/ssh/sshd_config.d /etc/sudoers.d /etc/fstab
find "$DEST" -name 'rhel01-*.tar.gz' -mtime +7 -delete
echo "Backup complete: $ARCHIVE"
```
`/etc/systemd/system/lab-backup.service`
```ini
[Unit]
Description=rhel01 configuration and web content backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/lab-backup.sh
```
`/etc/systemd/system/lab-backup.timer`
```ini
[Unit]
Description=Run rhel01 backup daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

## Check backup health
```bash
systemctl list-timers lab-backup.timer
journalctl -u lab-backup.service --since "2 days ago" --no-pager
ls -lh /backup
```

## Run a backup manually
```bash
sudo systemctl start lab-backup.service
journalctl -u lab-backup.service -n 5 --no-pager
```

## Restore a single file
```bash
LATEST=$(ls -t /backup/rhel01-*.tar.gz | head -1)
sudo tar -tzf "$LATEST" | grep <filename>       # confirm it exists in the archive
sudo tar -xzf "$LATEST" -C / <path/without/leading/slash>
sudo restorecon -Rv <restored-path>             # restore correct SELinux labels
```
Example (web page): `sudo tar -xzf "$LATEST" -C / srv/web/index.html && sudo restorecon -Rv /srv/web`

## Post-restore validation
```bash
curl http://localhost
ls -Z /srv/web
```

## Tested
<date>: deleted /srv/web/index.html, confirmed page missing, restored from latest archive, relabeled, confirmed page served. Output: see validation notes.

## Known limitation
Backups are stored on the same server. They protect against accidental deletion and bad changes, **not** loss of the server. Production would copy archives off-host (3-2-1 rule).