# KB-0001 — Department Transfer: Access Checklist

| Field | Value |
|---|---|
| Applies to | `corp.internal` user accounts, Departments file share |
| Audience | Help desk / systems administrators |
| Related | [TKT-0001](../incident-records/TKT-0001-user-transfer-share-access.md) |
| Last updated | 2026-10-03 |

## Purpose

Make sure a user who changes departments gets access to the new department's resources, **and loses access to the old ones**, in one pass.

## Background — how access works here (AGDLP)

```
User account → GG_<Dept> (global group) → DL_Share_<Dept>_RW (domain local) → NTFS Modify on \Departments\<Dept>
```

- **Group membership is what grants access.** Moving the user's OU or editing the Department field does **not** change permissions.
- The share uses **access-based enumeration**, so users only see folders they can open.

## Procedure

Replace `<user>`, `<OldDept>`, `<NewDept>`.

**1. Record the current state (before)**
```powershell
Get-ADUser <user> -Properties Department, Title | Select-Object Name, Department, Title, DistinguishedName
Get-ADPrincipalGroupMembership <user> | Select-Object Name
```

**2. Update the account record**
```powershell
Move-ADObject -Identity (Get-ADUser <user>).DistinguishedName -TargetPath "OU=<NewDept>,OU=Users,OU=Corp,DC=corp,DC=internal"
Set-ADUser <user> -Department "<NewDept>" -Title "<New title>"
```

**3. Swap department groups (the step that grants access)**
```powershell
Add-ADGroupMember    -Identity GG_<NewDept> -Members <user>
Remove-ADGroupMember -Identity GG_<OldDept> -Members <user> -Confirm:$false
```

**4. Confirm (after)**
```powershell
Get-ADPrincipalGroupMembership <user> | Select-Object Name
```
Expected: `Domain Users`, `GG_<NewDept>` — and **not** `GG_<OldDept>`.

**5. User action**
Ask the user to **sign out and sign back in**. Group membership is read into the Kerberos ticket at logon; an existing session keeps the old groups.

**6. Validate with the user**
```powershell
whoami /groups | findstr /i "GG_"
```
- S: shows the **new** department folder only
- User can create a file in the new folder
- Old department folder is no longer visible

## Common mistakes

| Symptom | Likely cause | Fix |
|---|---|---|
| New folder missing, Access denied | Group not changed (only OU/attributes) | Step 3 |
| Still denied after group change | User hasn't logged off/on | Step 5 |
| User sees both old and new folders | Old group not removed (privilege creep) | Remove `GG_<OldDept>` |
| S: drive missing entirely | GPO not applied | `gpupdate /force`, sign out/in, check `gpresult /r` |

## Troubleshooting tools

- `Get-ADPrincipalGroupMembership <user>` — what groups the user is in
- `icacls <folder>` — which groups the folder trusts
- Folder → Properties → Security → Advanced → **Effective Access** — what the user can actually do
- `gpresult /r /scope user` — which GPOs applied