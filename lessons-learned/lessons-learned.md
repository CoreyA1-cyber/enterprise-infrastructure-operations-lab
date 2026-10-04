# Lessons Learned

Consolidated from the build, the incidents, and the changes. Each entry is what broke, why, and the takeaway — the things that shaped how the lab was built and operated.

## Troubleshoot layer by layer, bottom up

- **Day-1 outage: the switch was unpowered.** Everything downstream looked broken; the cause was no power at the access layer. Lesson: start at physical/power before touching config.
- **VLAN 30 DNS failure.** The client had an IP but "No Internet." Packet captures showed DNS queries arriving on the USERS interface but never leaving toward the servers interface — pfSense was dropping them. Working the path hop by hop isolated a firewall rule, not a DNS or client problem.
- **A stale static IP on the admin PC** blocked connectivity after a network change. Checking addressing first would have found it faster.

## Firewall rules: order, scope, and "apply" all matter

- **pfSense rules don't take effect until Applied.** The AD pass rules were saved but inactive; traffic only matched once they were applied/reordered above the block-to-servers rule. First match wins.
- **Custom block rules don't log by default.** The DNS drop showed nothing in the firewall log because the block rule had logging off. Turned logging on so future drops are visible. (Only the default-deny rule logs out of the box.)
- **OpenVPN was set to Peer-to-Peer, not Remote Access,** and a rule had source `*`. Scope every allow to the specific source, destination, and ports it needs.

## Identity & access come from group membership, not attributes

- **TKT-0001:** a transferred user's OU and Department were updated, but the security group was never changed — so they had no access. In AD, the group grants access; the OU and attributes don't.
- **Group changes need a re-logon.** Group SIDs are captured in the Kerberos ticket at login, so changes don't apply until the user signs out and back in.
- **A transfer is an add *and* a remove.** Leaving the old group attached causes privilege creep.

## Tools create objects even when they error

- **`New-ADUser` creates the account even if the password fails policy** — leaving it disabled with no password. The fix was to reset and enable the existing object, not delete and recreate.
- **Domain join failed with "Access denied" until PowerShell was run as Administrator.** The error didn't name elevation as the cause; checking the session's privilege level did.

## Containment can cut off the administrator

- **INC-0002:** blocking the brute-force source by IP also blocked the admin workstation, because they shared the same VPN address — locking SSH out. Recovered via the Proxmox console. Lesson: block at a boundary the admin doesn't share, keep console access as a fallback, and scope blocks to `/32` not whole subnets.

## Verify configuration, don't assume it saved

- **The dc01 Splunk forwarder had no `outputs.conf`** — the installer never wrote it, so nothing forwarded despite the service running. Checked the file, wrote it by hand, and confirmed an established connection on 9997.
- **A stray `997/tcp allow-from-anywhere` ufw rule** (a typo of 9997) was found during a firewall audit and removed. Re-check rules with `ufw status numbered` after any change.
- **Service names are case-sensitive.** The health check silently skipped `splunkforwarder`; the real unit is `SplunkForwarder`. A silent skip hid a missing check until it was verified.

## Virtualization & install gotchas

- **Proxmox VLAN-aware bridge must be applied,** not just set — an unapplied change left VMs with no VLAN tagging.
- **Windows 11 requires TPM + Secure Boot keys** on the VM; dc01 (Server 2022) didn't. Match VM firmware to the guest's requirements.

---

**Overall takeaway:** the difference between "it works" and "I can prove it works and fix it when it doesn't" is documentation and validation. Every incident here was faster to resolve because the baseline, the change history, and the evidence were already written down.