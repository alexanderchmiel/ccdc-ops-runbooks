# Full DC Hardening

Everything beyond the first 30 minutes. Work through these in order.

## Privileged Group Audit

```powershell
$groups = @(
    "Domain Admins", "Enterprise Admins", "Schema Admins",
    "Backup Operators", "Account Operators", "Server Operators",
    "Group Policy Creator Owners", "DNSAdmins"
)

foreach ($g in $groups) {
    Write-Host "`n=== $g ===" -ForegroundColor Red
    Get-ADGroupMember -Identity $g | Select Name, SamAccountName
}
```

DNSAdmins is the one people forget. A member of DNSAdmins can execute arbitrary
DLLs as SYSTEM on the DC. That group should be essentially empty.

## Kerberos Attack Surface

### AS-REP Roasting

```powershell
# Find vulnerable accounts
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} -Properties DoesNotRequirePreAuth |
  Select Name, SamAccountName

# Fix all
Get-ADUser -Filter {DoesNotRequirePreAuth -eq $true} |
  Set-ADUser -DoesNotRequirePreAuth $false
```

### Kerberoasting

```powershell
# User accounts with SPNs are suspicious (service accounts are expected)
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName |
  Select Name, SamAccountName, ServicePrincipalName

# Force AES on accounts that legitimately have SPNs
Set-ADUser -Identity 'svcaccount' -KerberosEncryptionType AES256,AES128
```

## KRBTGT Password (Golden Ticket Prevention)

```powershell
Get-ADUser krbtgt -Properties PasswordLastSet | Select Name, PasswordLastSet
```

If old, reset TWICE with 10 minutes between. Two resets invalidate Golden Tickets.

```powershell
# Reset 1
Set-ADAccountPassword -Identity krbtgt `
  -NewPassword (ConvertTo-SecureString "$(New-Guid)$(New-Guid)" -AsPlainText -Force)

# Wait 10 minutes

# Reset 2
Set-ADAccountPassword -Identity krbtgt `
  -NewPassword (ConvertTo-SecureString "$(New-Guid)$(New-Guid)" -AsPlainText -Force)
```

Brief auth disruption during this — coordinate with team.

## DCSync ACL Check

```powershell
$domainDN = (Get-ADDomain).DistinguishedName
$acl = Get-ACL "AD:\$domainDN"
$acl.Access | Where-Object {
  $_.ObjectType -eq "1131f6aa-9c07-11d1-f79f-00c04fc2dcd2" -or
  $_.ObjectType -eq "1131f6ab-9c07-11d1-f79f-00c04fc2dcd2"
} | Select IdentityReference, ObjectType
```

Only DCs and Domain Admins should appear. Any other account with these GUIDs can
dump every password hash in the domain.

## AdminSDHolder Check

```powershell
$acl = Get-ACL "AD:\CN=AdminSDHolder,CN=System,DC=chefops,DC=local"
$acl.Access | Where-Object {
  $_.IdentityReference -notlike "*Domain Admins*" -and
  $_.IdentityReference -notlike "*Enterprise Admins*" -and
  $_.IdentityReference -notlike "*SYSTEM*" -and
  $_.IdentityReference -notlike "*Administrators*"
} | Select IdentityReference, ActiveDirectoryRights
```

Anything unexpected here has persistent admin rights reapplied every 60 minutes.

## Password Policy

```powershell
Set-ADDefaultDomainPasswordPolicy -Identity "chefops.local" `
  -LockoutThreshold 5 `
  -LockoutDuration "00:15:00" `
  -LockoutObservationWindow "00:15:00" `
  -MinPasswordLength 14 `
  -PasswordHistoryCount 12 `
  -ComplexityEnabled $true
```

Lockout of 3 is too aggressive — Red Team can lock employees out intentionally.
5 is a fair balance for competition.

## Fine-Grained Policy for Service Accounts

```powershell
New-ADFineGrainedPasswordPolicy -Name "ServiceAccount_PSO" `
  -Precedence 10 `
  -MinPasswordLength 25 `
  -ComplexityEnabled $true `
  -PasswordHistoryCount 24 `
  -MaxPasswordAge "0" `
  -LockoutThreshold 3 `
  -LockoutDuration "00:30:00"

Add-ADFineGrainedPasswordPolicySubject -Identity "ServiceAccount_PSO" `
  -Subjects "OU=ChefOps_ServiceAccounts,DC=chefops,DC=local"
```

## Services to Disable on a DC

```powershell
# Print Spooler — PrintNightmare. Do this first.
Stop-Service -Name Spooler -Force
Set-Service -Name Spooler -StartupType Disabled

# Remote Registry
Stop-Service -Name RemoteRegistry -Force
Set-Service -Name RemoteRegistry -StartupType Disabled

# Fax
Stop-Service -Name Fax -Force
Set-Service -Name Fax -StartupType Disabled
```

IIS only if you've confirmed ADFS doesn't depend on it. Check `Get-Service adfssrv`
is healthy first.

## SMB Hardening

```powershell
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force
```

Disable PowerShell v2 while you're at it:

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName MicrosoftWindowsPowerShellV2Root
```
