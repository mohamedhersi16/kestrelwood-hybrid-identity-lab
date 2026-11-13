# Kestrelwood Hybrid Identity Lab

A home lab where I built an on-premises Active Directory domain for a fictional healthcare company, Kestrelwood Health, and synced it to Microsoft Entra ID. I practiced onboarding, department moves and offboarding in PowerShell, and confirmed each change reached the cloud.

## Lab Setup

| Item | Details |
|---|---|
| Host | Windows 11 Pro with Hyper-V |
| Domain controller | DC01, Windows Server 2022 |
| Client | CLIENT01, Windows 11, joined to the domain |
| Domain | kestrelwood.internal |
| Cloud | Microsoft Entra ID (free tenant) |
| Sync tool | Microsoft Entra Connect Sync, Express Settings, Password Hash Sync |

**Structure:** Departments are organized into OUs (IT, HR, Clinical Staff, Billing, Executives, Disabled Users). Access is controlled with security groups (SG-IT, SG-HR, SG-Clinical, SG-Billing, SG-Executives).

---

## Part 1: Active Directory

**Joined a client PC to the domain**
![Client domain join](ad-01-client-domain-join.png)

**Applied a Group Policy screen lock**
![Screen lock working](ad-02-screen-lock-working.png)

**Reset a user's password**
![Password reset](ad-03-password-reset-account-enabled.png)

**Fixed users set to "password never expires"** (see Problems and Fixes)
![Error](ad-04-error-password-never-expires-conflict.png)
![Audit](ad-05-audit-password-never-expires.png)
![Fixed](ad-06-audit-fixed-all-users.png)

**Joiner:** created a new user
![Joiner audit](ad-07-joiner-final-audit.png)

**Mover:** moved a user from Billing to HR and updated the department, title, OU and groups
![Department and title](ad-08-mover-department-title-after.png)
![Groups swapped](ad-09-mover-groups-swapped.png)
![Mover audit](ad-10-mover-final-audit.png)

**Leaver:** offboarded a user
![Before](ad-11-leaver-before-elena.png)
![Disabled](ad-12-leaver-disabled.png)
![Groups removed](ad-13-leaver-groups-removed.png)
![Moved to Disabled Users](ad-14-leaver-moved-to-disabled.png)
![Leaver audit](ad-15-leaver-final-audit.png)

---

## Part 2: Hybrid Sync to Entra ID

**Prepared user sign-in names for the cloud** (see Problems and Fixes)
![UPN updated](00-azure-upn-updated.png)

**Created a cloud admin account with MFA and installed Entra Connect**
![Sync status](01-entra-connect-sync-status.png)
![Sync runs](02-sync-service-operations.png)
![Global admin role](03-syncadmin-global-admin.png)
![MFA sign-in](04-syncadmin-signin-mfa.png)

**Confirmed users synced.** Synced users show "Yes"; cloud-only accounts show "No".
![All users synced](05-all-users-sync-status.png)

### Joiner: new hire synced to the cloud
```powershell
Get-ADUser -Filter "SamAccountName -eq 'agrant'"   # confirm the username is free
$pw = Read-Host "Temp password" -AsSecureString
New-ADUser -Name "Aisha Grant" -GivenName "Aisha" -Surname "Grant" -SamAccountName "agrant" `
  -UserPrincipalName "agrant@<tenant>.onmicrosoft.com" -Path $ou -Department "Billing" `
  -Title "Billing Specialist" -AccountPassword $pw -ChangePasswordAtLogon $true -Enabled $true
Add-ADGroupMember -Identity "SG-Billing" -Members agrant
Start-ADSyncSyncCycle -PolicyType Delta
```
![Aisha Grant](06-joiner-aisha-grant.png)

### Mover: Clinical Staff to Billing
```powershell
$user = Get-ADUser -Filter 'Name -eq "Samuel Okafor"' -Properties MemberOf
Set-ADUser $user -Department "Billing" -Title "Medical Coding Specialist"
Remove-ADGroupMember -Identity "SG-Clinical" -Members $user -Confirm:$false
Add-ADGroupMember -Identity "SG-Billing" -Members $user
Move-ADObject -Identity $user -TargetPath $ou
Start-ADSyncSyncCycle -PolicyType Delta
```
Old access is removed before new access is added, to prevent privilege creep.
![Samuel Okafor](07-mover-samuel-okafor.png)
![SG-Billing members](08-sg-billing-members.png)

### Leaver: offboarding synced to the cloud
```powershell
$user = Get-ADUser -Filter 'Name -eq "Marcus Hill"' -Properties MemberOf
$disabledOU = (Get-ADOrganizationalUnit -Filter 'Name -eq "Disabled Users"').DistinguishedName
Disable-ADAccount $user
$rand = ConvertTo-SecureString ([guid]::NewGuid().ToString() + "Aa1!") -AsPlainText -Force
Set-ADAccountPassword $user -Reset -NewPassword $rand
$user.MemberOf | ForEach-Object { Remove-ADGroupMember -Identity $_ -Members $user -Confirm:$false }
Set-ADUser $user -Description "Leaver - disabled 2026-11-12 per HR ticket"
Move-ADObject -Identity $user -TargetPath $disabledOU
Start-ADSyncSyncCycle -PolicyType Delta
```
The account was disabled in the cloud, and the user was removed from SG-HR.
![Marcus Hill disabled](09-leaver-marcus-hill.png)
![SG-HR members](10-sg-hr-members-after-leaver.png)
![Audit log](11-marcus-audit-log.png)

---

## Problems and Fixes

| Problem | Cause | Fix |
|---|---|---|
| Could not force a password change at next sign-in | The user was set to "password never expires," which blocks it | Audited all users for this setting and turned it off |
| The domain `kestrelwood.internal` could not be used for cloud sign-in | `.internal` domains can't be verified in Entra ID | Changed user sign-in names to the `onmicrosoft.com` domain before syncing |
| `Start-ADSyncSyncCycle` was "not recognized" | The sync module wasn't loaded in the PowerShell session | `Import-Module "C:\Program Files\Microsoft Azure AD Sync\Bin\ADSync\ADSync.psd1"` |
| New hire was missing her department group | The group was left out at creation | Added her to SG-Billing and checked the group's member list |

## What I'd Do Differently in Production

- Install Entra Connect on its own server, not on the domain controller.
- Use a company-owned, verified domain for sign-in.
- Choose Custom setup to control which OUs sync.
- Revoke cloud sessions during offboarding so active sign-ins end right away.
