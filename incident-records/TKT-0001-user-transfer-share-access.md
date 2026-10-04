# TKT-0001 — Transferred User Cannot Access Finance Share

| Field | Value |
|---|---|
| Ticket | TKT-0001 |
| Type | Service request / access issue |
| Priority | P3 — single user, workaround not available |
| Requester | Jamie Cole (`jcole`) — transferred Sales → Finance |
| Assigned to | Corey Armstrong |
| Opened | 2026-10-03 20:10 (local) |
| Resolved | 2026-10-03 20:30 (local) |
| Time to resolve | 20 minutes |
| Status | Resolved |

> **Lab scenario.** The incomplete transfer was introduced deliberately to simulate a common help-desk process gap. All investigation, fix, and validation steps were performed live.

---

## 1. Reported issue

User transferred from Sales to Finance. When logging in on ws01, the S: (Departments) drive shows only the **Sales** folder. Browsing directly to `\\dc01.corp.internal\Departments\Finance` returns **Access denied**.

## 2. Environment

| Component | Detail |
|---|---|
| Domain | `corp.internal` (dc01, 192.168.20.10, Windows Server 2022) |
| Client | ws01 (Windows 11 Enterprise, VLAN 30) |
| Share | `\\dc01.corp.internal\Departments` — access-based enumeration enabled |
| Access model | AGDLP: user → `GG_<Dept>` (global) → `DL_Share_<Dept>_RW` (domain local) → NTFS Modify |
| Drive mapping | GPO `U_DriveMap_Departments` → S: |

## 3. Reproduction

1. Logged in to ws01 as `CORP\jcole`.
2. S: displayed Sales only (ABE hides folders the user cannot access).
3. Direct UNC path to the Finance folder → **Access denied**. *(Screenshot 46)*

## 4. Investigation

| Check | Command / tool | Finding |
|---|---|---|
| Account attributes | `Get-ADUser jcole -Properties Department, Title` | Department = Finance, Title = Financial Analyst, OU = Finance ✅ |
| Group membership | `Get-ADPrincipalGroupMembership jcole` | **Domain Users, GG_Sales** ❌ — not in GG_Finance |
| Folder permissions | `icacls C:\Shares\Departments\Finance` | Only `DL_Share_Finance_RW` (Modify), Domain Admins, SYSTEM |
| Group nesting | `Get-ADGroupMember DL_Share_Finance_RW` | Contains `GG_Finance` only |
| Effective access | Folder → Security → Advanced → Effective Access (jcole) | All permissions denied *(Screenshot 47)* |

**Conclusion:** the account *looked* like a Finance user (OU, Department, Title), but access is granted by **group membership**, and the user was still only in `GG_Sales`.

## 5. Root cause

The transfer was processed by moving the user object and updating attributes, but the **department security group was not changed**. Moving an OU or editing the Department field grants no permissions in this design.

## 6. Resolution

```powershell
Add-ADGroupMember -Identity GG_Finance -Members jcole
Remove-ADGroupMember -Identity GG_Sales -Members jcole -Confirm:$false
```

Sales access was removed as part of the fix to apply least privilege and prevent privilege creep from the transfer.

## 7. Validation

| Check | Before | After |
|---|---|---|
| `Get-ADPrincipalGroupMembership jcole` | Domain Users, GG_Sales | Domain Users, **GG_Finance** |
| User logged off / on (new Kerberos ticket) | — | Done |
| `whoami /groups \| findstr GG_` on ws01 | GG_Sales | **GG_Finance** |
| S: drive contents | Sales | **Finance** only |
| Write test | Access denied | `transfer-test.txt` created ✅ *(Screenshot 48)* |

## 8. Follow-up

- [x] Created **KB-0001 — Department Transfer: Access Checklist**
- [x] dc01 Security log forwarded to Splunk (`index=windows`); alert "AD security group membership change" (4728/4729/4732/4733/4756/4757) created and tested (screenshots 49–51)
- [ ] Consider a transfer script that swaps department groups in one step to remove the manual gap

## 9. Lessons learned

- In AD, **OU placement and attributes are not access** — group membership is.
- Always compare the **user's groups** against the **resource's ACL** before changing anything.
- Group changes require the user to **log off and back on** (or purge Kerberos tickets) because group SIDs are captured in the logon token.
- A transfer is both an **add and a remove**; skipping the remove leaves excess access.