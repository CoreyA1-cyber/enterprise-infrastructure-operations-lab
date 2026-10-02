# Enterprise Infrastructure Operations Lab

## Objective
My objective with this project is to create a small enterprise network where I can get hands-on technical skills to display knowledge and capabality of setting up and maintaining infrastructure which I can translate these skills into a live operation for career aspirations.
## Environment
| Component | Platform |
|---|---|
| Firewall | pfSense CE 2.7.2 (physical) |
| Switch | TP-Link Omada 8-port PoE+ |
| Hypervisor | Proxmox VE on Dell OptiPlex |
| Linux server | RHEL 10 |
| Directory services | Windows Server 2022 (AD DS, DNS) |
| Client | Windows (domain-joined) |
| Monitoring | Splunk |

## Documentation
- [IP Plan](architecture/ip-plan.md)
- [Asset Inventory](architecture/asset-inventory.md)
- [Failure Scenario](architecture/failure-scenario.md)
- [rhel01 Build Guide](build-guides/rhel01-build.md)
- [Backup & Restore Runbook](runbooks/rhel01-backup-restore.md)
- [splunk01 Build Guide](build-guides/splunk01-build.md)
- [Monitoring Change Record](change-records/2026-10-01-monitoring-deployment.md)
- [Equipment Layout](architecture/equipment-layout.md)

## Status
| Phase | Status |
|---|---|
| Network foundation & remote access | ✅ Complete |
| RHEL administration (rhel01) | ✅ Complete |
| Monitoring (splunk01) | ✅ Complete |
| NOC incident | ✅ Complete |
| Change management | ⏳ Planned |
| AD support incident | ⏳ Planned |
| Security & automation | ⏳ Planned |
