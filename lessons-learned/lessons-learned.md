# Lessons Learned

## Troubleshooting
- **Check the physical layer first.** The "Proxmox outage" was an unpowered switch.
- **Establish scope before fixing.** One unreachable host was actually the whole lab offline.
- **Timeout vs. refused tells you which layer to look at.** Timeout = nothing answered.
- **Pinging your own IP proves nothing about the network.** Test the gateway.
- **Check your own inputs.** A failed login was the wrong username, not a failed password reset.
- **Old settings can mislead.** A stale static IP made the network look healthy when DHCP hadn't been tested.
- **"It worked after X" is not a root cause.** Record what was observed and mark causes unconfirmed when they are.

## Design & security
- **The firewall must be the only path.** Removed a cable that let traffic bypass it.
- **When requirements change, the design changes.** The blanket RFC1918 WAN block was replaced by one explicit VPN rule once remote access was needed.
- **Interface address ≠ interface subnet.** A rule scoped to the firewall's own IP would have silently bypassed segmentation. Peer review caught it.
- **Know the VPN mode.** Peer-to-peer is site-to-site; remote access is for individual users.
- **A test only counts if it can fail.** Unplugged the direct cable before testing the VPN.
- **Least privilege works quietly.** A blocked port (HTTP to the switch) showed up as a timeout, exactly as designed.

## Tooling
- **Git repo-level config overrides global config.** A placeholder email broke commit attribution; found with `git config --show-origin`.
- **Run commands from the right folder.** The prompt shows where you are.
- **A folder named `private/` is ignored at any depth.** Moving it inside `screenshots/` kept the images out of the commit.

## Linux administration (rhel01)
- **Configured ≠ applied.** Proxmox showed VLAN-aware enabled while the change was still pending. Check the running state (`/sys/class/net/vmbr0/bridge/vlan_filtering`), not the GUI.
- **A passing syntax check is not proof of effect.** `sshd -t` passed on an unchanged config; `sshd -T` shows what is actually enforced.
- **Client vs. server config:** `ssh_config` (outbound) vs. `sshd_config` (inbound).
- **Generate keys on the client.** The private key must never live on the server.
- **Know which machine you're on.** The prompt (`PS C:\>` vs `user@host:~$`) tells you.
- **SELinux denials are fixed with labels, not by disabling SELinux.** `semanage fcontext` + `restorecon` is permanent; `chcon` is not.
- **Test fstab with `mount -a` before rebooting.** A bad entry can drop the server into emergency mode.
- **Validate after reboot.** The reboot test exposed a volatile journal that would have hidden pre-reboot evidence during an incident.
- **Don't claim evidence you don't have.** No AVC was logged during the original fix, so the denial was reproduced as a controlled test.

## Monitoring (splunk01)
- **Check what you downloaded before you use it.** File size and type (76 KiB `text/html`, `.msi` vs `.deb`) revealed the wrong file before it caused a failure.
- **Monitor from outside the target.** A check running on the server it watches can't report that server being down.
- **Grant access, not root.** ACLs gave the forwarder read-only access to specific logs; logrotate hooks keep it working after rotation.
- **Defaults aren't capacity planning.** Index max size defaulted larger than the disk.
- **Ctrl+C cancels; Ctrl+Z only pauses.** A paused install step can look finished. Verify the end state (`systemctl is-enabled`).
- **Named terminal tabs prevent running commands on the wrong host.**