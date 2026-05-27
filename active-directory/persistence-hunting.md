# Persistence Hunting on a Domain Controller

Red Team plants persistence in places admins rarely look. Things that survive
reboots, run automatically, or grant access without credentials.

Hunt before you harden. Otherwise you harden around a backdoor.

## Quick Persistence Check (5 minutes)

Four commands. Screenshot everything.

```powershell
# 1. Scheduled tasks outside Microsoft namespace
Get-ScheduledTask |
  Where-Object {$_.TaskPath -notlike "\Microsoft\*"} |
  Select TaskName, TaskPath

# 2. Services from suspicious paths
Get-WmiObject Win32_Service |
  Where-Object {$_.PathName -notlike "*system32*" -and
                $_.PathName -notlike "*Program Files*"} |
  Select Name, PathName

# 3. Privileged group membership
Get-ADGroupMember "Domain Admins" | Select Name, SamAccountName

# 4. WMI subscriptions
Get-WMIObject -Namespace root\subscription -Class __EventFilter
```

If any of these come back with unexpected results, drop everything and investigate.

## Scheduled Tasks

```powershell
# All non-Microsoft tasks with what they execute
Get-ScheduledTask |
  Where-Object {$_.TaskPath -notlike "\Microsoft\*"} |
  ForEach-Object {
    [PSCustomObject]@{
      Name    = $_.TaskName
      Path    = $_.TaskPath
      Execute = $_.Actions.Execute
      Args    = $_.Actions.Arguments
      User    = $_.Principal.UserId
    }
  } | Format-List
```

Red flags:
- Tasks running as SYSTEM that you didn't create
- Execute path in `%temp%`, `C:\Users\Public`, or `C:\Windows\Temp`
- PowerShell with `-EncodedCommand` arguments
- Tasks created in the last few days

```powershell
# Tasks created recently
Get-ScheduledTask |
  Where-Object {$_.Date -gt (Get-Date).AddDays(-7)} |
  Select TaskName, Date
```

## Registry Autorun Keys

```powershell
$keys = @(
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce",
    "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run",
    "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce",
    "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
)
foreach ($k in $keys) {
    Write-Host "`n=== $k ===" -ForegroundColor Yellow
    Get-ItemProperty -Path $k -ErrorAction SilentlyContinue
}
```

### Winlogon hijacking

This one bites. Shell and Userinit should be exactly:

- `Shell = explorer.exe`
- `Userinit = C:\Windows\system32\userinit.exe,`

Anything appended is a backdoor. Fix:

```powershell
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" `
  -Name "Shell" -Value "explorer.exe"
Set-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" `
  -Name "Userinit" -Value "C:\Windows\system32\userinit.exe,"
```

## Malicious Services

```powershell
# Services running from non-standard paths
Get-WmiObject Win32_Service |
  Where-Object {$_.PathName -notlike "*system32*" -and
                $_.PathName -notlike "*Program Files*" -and
                $_.PathName -notlike "*Microsoft*"} |
  Select Name, DisplayName, PathName, State

# Specifically check temp and user directories
Get-WmiObject Win32_Service |
  Where-Object {$_.PathName -like "*temp*" -or
                $_.PathName -like "*appdata*" -or
                $_.PathName -like "*public*"} |
  Select Name, PathName
```

A service running from `C:\Users\` is almost always malicious. Stop and disable:

```powershell
Stop-Service -Name "SuspiciousService" -Force
Set-Service -Name "SuspiciousService" -StartupType Disabled
```

## WMI Subscriptions

This is the stealthiest persistence method. Hides entirely in the WMI database
and survives reboots.

```powershell
Get-WMIObject -Namespace root\subscription -Class __EventFilter |
  Select Name, Query

Get-WMIObject -Namespace root\subscription -Class __EventConsumer |
  Select Name, CommandLineTemplate, ScriptText

Get-WMIObject -Namespace root\subscription -Class __FilterToConsumerBinding |
  Select Filter, Consumer
```

Any results here are almost certainly Red Team. Remove in this order — binding,
consumer, filter:

```powershell
Get-WMIObject -Namespace root\subscription -Class __FilterToConsumerBinding | Remove-WMIObject
Get-WMIObject -Namespace root\subscription -Class __EventConsumer | Remove-WMIObject
Get-WMIObject -Namespace root\subscription -Class __EventFilter | Remove-WMIObject
```

## Recently Created Users

```powershell
# AD accounts created in the last 24 hours
Get-ADUser -Filter * -Properties Created, LastLogonDate |
  Where-Object {$_.Created -gt (Get-Date).AddDays(-1)} |
  Select Name, SamAccountName, Created, Enabled

# Local accounts on the DC itself
Get-LocalUser | Select Name, Enabled, LastLogon, PasswordLastSet

# Accounts with $ suffix (commonly used to hide accounts)
Get-LocalUser | Where-Object {$_.Name -like "*$"}
```

## Startup Folders

```powershell
Get-ChildItem "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup" |
  Select Name, LastWriteTime

Get-ChildItem "C:\Users\*\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup" |
  Select Name, LastWriteTime, Directory
```

## Recent File Activity in System Directories

```powershell
# System32 files modified in last 24 hours
Get-ChildItem C:\Windows\System32 -File |
  Where-Object {$_.LastWriteTime -gt (Get-Date).AddHours(-24)} |
  Select Name, LastWriteTime | Sort LastWriteTime -Descending

# Executables in temp directories
Get-ChildItem C:\Windows\Temp, C:\Temp, "$env:TEMP" `
  -Include *.exe, *.ps1, *.bat, *.vbs, *.dll `
  -Recurse -ErrorAction SilentlyContinue |
  Select FullName, LastWriteTime
```

## On Every 30-Minute Rotation

Run the quick check at the top of this file. Persistence can be replanted
the moment you turn your back, especially after you've cleaned it once.
