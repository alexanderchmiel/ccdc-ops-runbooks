# Domain Controller — First 30 Minutes

The DC is the authentication backbone. If it goes down, the domain goes down.
Don't change anything until you know what you have.

## Orient (Minutes 0–5)

```powershell
hostname
whoami
ipconfig /all
Get-ADDomain
Get-ADForest
Get-ADDomainController -Filter *
```

Write down the IP, hostname, domain name, and PDC Emulator. You'll need them.

## Audit Users (Minutes 5–15)

```powershell
# Domain Admins — every name should be familiar
Get-ADGroupMember "Domain Admins" | Select Name, SamAccountName

# Stale accounts and accounts that never logged in
Get-ADUser -Filter * -Properties LastLogonDate, Enabled |
  Select Name, SamAccountName, Enabled, LastLogonDate |
  Sort LastLogonDate
```

### Fix ms-DS-MachineAccountQuota immediately

Default is 10, which means any domain user can join up to 10 machines. This is
how the "gamingpc" rogue device attack works.

```powershell
# Check
Get-ADObject -Identity ((Get-ADDomain).DistinguishedName) `
  -Properties ms-DS-MachineAccountQuota

# Fix
Set-ADDomain -Identity chefops.local `
  -Replace @{"ms-DS-MachineAccountQuota"="0"}
```

### Rogue computer check

```powershell
Get-ADComputer -Filter * -Properties Created |
  Select Name, Created | Sort Created -Descending
```

## GPO Posture (Minutes 15–25)

Enable PowerShell logging via GPO. Three settings, all need to be on:

- Computer Configuration > Admin Templates > Windows Components > Windows PowerShell
- Module Logging
- Script Block Logging
- Transcription

Verify with `gpresult /r /scope computer`. We lost points at quals for configuring
this but not proving it was enforced on multiple hosts — the gpresult output is
your evidence.

## ADFS Health (Minutes 25–35)

```powershell
Get-Service adfssrv
Get-AdfsSslCertificate
Get-AdfsProperties | Select FederationServiceName
```

Open `https://sts.chefops.local/adfs/ls/idpinitiatedsignon` in a browser and
screenshot the page. That's your scored evidence.

## Audit Policy Baseline

```powershell
auditpol /get /category:*

# Set anything showing "No Auditing"
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Account Management" /success:enable /failure:enable
auditpol /set /subcategory:"DS Access" /success:enable /failure:enable
auditpol /set /subcategory:"Policy Change" /success:enable /failure:enable
```

## Mindset

Business continuity beats perfect security. Before any aggressive change ask:
"Will the periodic employee access check fail because of this?" Five points
per failed check adds up over a day.

Black Team said it explicitly during quals: don't disable RDP in a cloud
environment. Same principle applies to any port you depend on for your own
management access.
