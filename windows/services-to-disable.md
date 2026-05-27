# Services to Disable on a Domain Controller

## See What's Running

```powershell
Get-Service | Where-Object {$_.Status -eq "Running"} |
  Select Name, DisplayName, StartType |
  Sort-Object DisplayName

# Cross-reference with what's listening
netstat -ano | findstr "LISTENING"
```

## Print Spooler (Disable First)

PrintNightmare. No printers on a DC.

```powershell
Stop-Service -Name Spooler -Force
Set-Service -Name Spooler -StartupType Disabled
```

## Remote Registry

Red Team favorite — allows remote registry edits from anywhere.

```powershell
Stop-Service -Name RemoteRegistry -Force
Set-Service -Name RemoteRegistry -StartupType Disabled
```

## Fax

```powershell
Stop-Service -Name Fax -Force
Set-Service -Name Fax -StartupType Disabled
```

## IIS

Only after confirming ADFS isn't dependent on it. Check first:

```powershell
Get-Service adfssrv
Get-WindowsFeature Web-Server
```

If ADFS is healthy and IIS is just present from a build template:

```powershell
Stop-Service -Name W3SVC -Force
Set-Service -Name W3SVC -StartupType Disabled
Stop-Service -Name IISADMIN -Force
Set-Service -Name IISADMIN -StartupType Disabled
```

If ADFS breaks after stopping IIS, re-enable it. In some configurations they're
coupled.

## Bluetooth

```powershell
Stop-Service -Name bthserv -Force
Set-Service -Name bthserv -StartupType Disabled
```

## Xbox (yes really, sometimes on Server)

```powershell
Get-Service | Where-Object {$_.Name -like "xbox*"} | Stop-Service -Force
Get-Service | Where-Object {$_.Name -like "xbox*"} | Set-Service -StartupType Disabled
```

## WMI Performance Adapter

Lower priority but worth it.

```powershell
Stop-Service -Name wmiApSrv -Force
Set-Service -Name wmiApSrv -StartupType Disabled
```

## PowerShell v2

Old version that bypasses script block logging in newer engines.

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName MicrosoftWindowsPowerShellV2Root
```

## SMB Hardening (Not a Service, But Related)

```powershell
# Disable SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force

# Verify
Get-SmbServerConfiguration | Select EnableSMB1Protocol

# Enforce SMB signing
Set-SmbServerConfiguration -RequireSecuritySignature $true -Force
```

## Decision Framework Before Disabling Anything

Three questions:

1. Does anything in the network topology depend on this? Keycloak needs LDAP/LDAPS.
   ADFS needs 443. Don't break dependencies.
2. Will an employee fail their periodic access check because of this? Five points
   per failure.
3. Is this DC-specific or should I push it fleet-wide via GPO? Print Spooler
   should be disabled on every machine, not just the DC.

Print Spooler and Remote Registry are the safest immediate wins. Document them
with screenshots and move to the next thing.
