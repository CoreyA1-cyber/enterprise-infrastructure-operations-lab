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