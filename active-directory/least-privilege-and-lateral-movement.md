# Least Privilege and Stopping Lateral Movement

Three ways escalation happens in AD:

1. Excessive group membership
2. Misconfigured ACLs on AD objects
3. Kerberos abuse (Kerberoasting, AS-REP roasting)

Address all three.

## OU Structure First

Permissions and GPOs hang off OUs. Build the structure before locking anything down.

```powershell
New-ADOrganizationalUnit -Name "ChefOps_Users" -Path "DC=chefops,DC=local"
New-ADOrganizationalUnit -Name "ChefOps_Computers" -Path "DC=chefops,DC=local"
New-ADOrganizationalUnit -Name "ChefOps_ServiceAccounts" -Path "DC=chefops,DC=local"
New-ADOrganizationalUnit -Name "ChefOps_Admins" -Path "DC=chefops,DC=local"

New-ADOrganizationalUnit -Name "Regular_Employees" `
  -Path "OU=ChefOps_Users,DC=chefops,DC=local"
New-ADOrganizationalUnit -Name "IT_Staff" `
  -Path "OU=ChefOps_Users,DC=chefops,DC=local"
New-ADOrganizationalUnit -Name "Kiosk_Accounts" `
  -Path "OU=ChefOps_Users,DC=chefops,DC=local"

# Move users into the right place
Move-ADObject -Identity "CN=jsmith,CN=Users,DC=chefops,DC=local" `
  -TargetPath "OU=Regular_Employees,OU=ChefOps_Users,DC=chefops,DC=local"
```

## Separate Admin Accounts for IT Staff

IT staff get two accounts. Daily work account has nothing. Admin account is the
only one in Domain Admins.

```powershell
New-ADUser -Name "jsmith-adm" `
  -SamAccountName "jsmith-adm" `
  -UserPrincipalName "jsmith-adm@chefops.local" `
  -Path "OU=ChefOps_Admins,DC=chefops,DC=local" `
  -AccountPassword (ConvertTo-SecureString "ComplexP@ss123!" -AsPlainText -Force) `
  -Enabled $true

Add-ADGroupMember -Identity "Domain Admins" -Members "jsmith-adm"
```

If Red Team compromises `jsmith` they get nothing useful. The `-adm` account is
only used when explicitly needed.

## GPO Restrictions

### Restrict Local Admin via Restricted Groups GPO

`Computer Configuration > Windows Settings > Security Settings > Restricted Groups`

Add Administrators group and define exactly who should be in it. Now even if
Red Team creates a local admin on a workstation, the GPO removes it on next refresh.

### Deny Network Logon for Regular Users

`Computer Configuration > Windows Settings > Security Settings > Local Policies > User Rights Assignment`

- "Deny access to this computer from the network" — add regular user groups
- "Deny log on through Remote Desktop Services" — add regular user groups

Regular employees should never RDP into servers. Only admin accounts.

### Disable cmd and PowerShell for Regular Employees

`User Configuration > Administrative Templates > System`

- "Prevent access to the command prompt" — Enabled
- "Don't run specified Windows applications" — enable and add `powershell.exe`,
  `cmd.exe`, `wscript.exe`, `cscript.exe`, `mshta.exe`, `regedit.exe`

A POS or kiosk employee has zero legitimate reason to open cmd.

## Kerberos Hardening

### Find AS-REP roastable accounts

```powershell
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Properties DoesNotRequirePreAuth |
  Select Name, SamAccountName

# Fix all
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} |
  Set-ADUser -DoesNotRequirePreAuth $false
```

### Find Kerberoastable user accounts

```powershell
# Service accounts with SPNs are expected. User accounts with SPNs are not.
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName |
  Select Name, SamAccountName, ServicePrincipalName

# Remove SPN from a user account that shouldn't have one
Set-ADUser -Identity "username" `
  -ServicePrincipalNames @{Remove="HTTP/something.chefops.local"}
```

### Force AES encryption on service accounts

```powershell
Set-ADUser -Identity "svcaccount" -KerberosEncryptionType AES256,AES128
```

RC4 is what makes Kerberoasting fast. AES makes it impractical.

## ACL Audit on Sensitive Objects

This is where DCSync attacks come from — an account with replication rights
that shouldn't have them.

```powershell
$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-ACL "AD:\$domainDN"
$acl.Access | Where-Object {
  $_.ObjectType -eq "1131f6aa-9c07-11d1-f79f-00c04fc2dcd2" -or
  $_.ObjectType -eq "1131f6ab-9c07-11d1-f79f-00c04fc2dcd2"
} | Select IdentityReference, ObjectType
```

Those GUIDs are the DCSync permission GUIDs. Only DCs and Domain Admins should appear.

## Mental Model

Four layers. If Red Team gets one employee's credentials, they should hit a wall
at every layer:

```
Layer 1 — Account Hygiene
  No unnecessary group memberships
  Separate admin accounts for IT staff

Layer 2 — OU Structure + GPOs
  Restrict what users can run
  Restrict where they can log in from

Layer 3 — Kerberos Hardening
  No SPNs on user accounts
  AS-REP roasting disabled
  AES on service accounts

Layer 4 — ACL Auditing
  No rogue DCSync rights
  AdminSDHolder clean
```
