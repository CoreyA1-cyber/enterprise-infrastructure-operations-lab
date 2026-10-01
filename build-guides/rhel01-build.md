# Build Guide: rhel01 (RHEL 10 Application/SSH Server)

**Role:** Web application and SSH server on the Servers VLAN
**Address:** 192.168.20.20/24 · Gateway 192.168.20.1 · DNS 192.168.20.1
**Platform:** Proxmox VM 120 · 2 vCPU (type host) · 4 GB RAM · 40 GB OS disk + 10 GB data disk

---

## 1. VM creation (Proxmox)
| Setting | Value | Why |
|---|---|---|
| CPU type | host | RHEL 10 requires x86-64-v3; the default QEMU CPU model will not boot it |
| BIOS / Machine | OVMF (UEFI) / q35 | Modern firmware |
| Disks | scsi0 40 GB (OS), scsi1 10 GB (data) | Data disk kept separate for LVM practice |
| Network | vmbr0, VLAN tag 20, VirtIO; Proxmox NIC firewall off | Network policy enforced at pfSense; host policy by firewalld |
| QEMU agent | Enabled | Hypervisor visibility of guest IP and clean shutdown |

**Prerequisite:** vmbr0 must be VLAN-aware *and the change applied*. See change record row 21.

After install: detach ISO and set scsi0 first in boot order, or the VM will boot the installer again.

## 2. Installation
- Installation source: RHEL 10.2 DVD ISO, SHA-256 verified on download
- Network configured first (static IPv4), then registered with Red Hat
- Destination: 40 GB disk only (data disk left untouched)
- Software: Server (no GUI)
- Root account locked; admin user created as member of `wheel`

## 3. Network verification
```bash
nmcli con show
nmcli -f ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns con show ens18
```
`ipv4.method: manual` confirms the static configuration persists across reboots.

## 4. Packages and updates
```bash
sudo subscription-manager status
sudo dnf update -y
sudo dnf install -y qemu-guest-agent httpd policycoreutils-python-utils
sudo systemctl enable --now qemu-guest-agent
```

## 5. Users, groups, and sudo delegation
```bash
sudo groupadd webadmins
sudo useradd -m -G webadmins opsuser1
sudo passwd opsuser1
sudo visudo -f /etc/sudoers.d/webadmins
```
`/etc/sudoers.d/webadmins`:
%webadmins ALL=(root) /usr/bin/systemctl start httpd, /usr/bin/systemctl stop httpd, /usr/bin/systemctl restart httpd, /usr/bin/systemctl status httpd

Least privilege: web admins can manage httpd only. `visudo` validates syntax before saving.
Verify: `sudo -l -U opsuser1`

## 6. SSH key authentication and hardening
On the admin workstation (PowerShell), **not** on the server:
```powershell
ssh-keygen -t ed25519 -C "admin-workstation"
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <admin-user>@192.168.20.20 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```
The private key never leaves the workstation; only the public key is placed on the server.

After confirming key login works (keep an existing session open):
```bash
sudo tee /etc/ssh/sshd_config.d/01-hardening.conf <<'EOF'
PasswordAuthentication no
PermitRootLogin no
EOF
sudo sshd -t && sudo systemctl reload sshd
sudo sshd -T | grep -Ei '^(passwordauthentication|permitrootlogin)'
```
- Server config is `sshd_config.d`, not `ssh_config.d` (client).
- Named `01-` because sshd uses the first value found; this takes precedence over vendor defaults.
- `sshd -t` checks syntax only; `sshd -T` shows the effective configuration.

## 7. Web service and host firewall
```bash
sudo systemctl enable --now httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --remove-service=cockpit --remove-service=dhcpv6-client
sudo firewall-cmd --reload
sudo firewall-cmd --list-services        # http ssh
sudo systemctl disable --now cockpit.socket
```
`enable --now` = start now and at boot. `--permanent` + `--reload` = survives reboot and is active now.

## 8. LVM storage and persistent mounts
```bash
sudo pvcreate /dev/sdb
sudo vgcreate vg_data /dev/sdb
sudo lvcreate -n lv_web -L 5G vg_data
sudo lvcreate -n lv_backup -L 3G vg_data
sudo mkfs.xfs /dev/vg_data/lv_web
sudo mkfs.xfs /dev/vg_data/lv_backup
sudo mkdir -p /srv/web /backup
sudo blkid /dev/vg_data/lv_web /dev/vg_data/lv_backup
```
`/etc/fstab` entries (by UUID; device names can change, UUIDs do not):
UUID=<lv_web-uuid> /srv/web xfs defaults 0 0
UUID=<lv_backup-uuid> /backup xfs defaults 0 0

```bash
sudo systemctl daemon-reload
sudo mount -a          # validates fstab before a reboot can fail on it
findmnt /srv/web
findmnt /backup
```
~2 GB intentionally left free in vg_data for online extension (`lvextend -r`).

## 9. Serving content from LVM + SELinux
```bash
echo "<h1>rhel01 - Enterprise Infrastructure Operations Lab</h1>" | sudo tee /srv/web/index.html
sudo tee /etc/httpd/conf.d/lab-site.conf <<'EOF'
DocumentRoot "/srv/web"
<Directory "/srv/web">
    Require all granted
</Directory>
EOF
sudo apachectl configtest && sudo systemctl restart httpd
sudo semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
sudo restorecon -Rv /srv/web
curl http://localhost
```
SELinux stays **Enforcing**. A permanent labeling rule (`semanage` + `restorecon`) is used instead of `chcon` (temporary) or disabling SELinux. Denial demonstrated in `validation/rhel01-selinux-validation.md`.

## 10. Persistent journal
```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/persistent.conf <<'EOF'
[Journal]
Storage=persistent
EOF
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo restorecon -Rv /var/log/journal
sudo systemctl restart systemd-journald
sudo journalctl --flush
```

## 11. Scheduled backup (systemd timer)
See `runbooks/rhel01-backup-restore.md` for the script, units, and restore procedure.

## 12. Validation
- Network and segmentation: `validation/rhel01-network-validation.md`
- SELinux: `validation/rhel01-selinux-validation.md`
- Reboot persistence: `validation/rhel01-reboot-validation.md`