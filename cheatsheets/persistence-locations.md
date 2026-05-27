# Cheatsheet — Persistence Hunt Locations

## The 4 Critical Checks (5 minutes)

```powershell
# 1. Scheduled tasks
Get-ScheduledTask | Where {$_.TaskPath -notlike "\Microsoft\*"} | Select TaskName, TaskPath

# 2. Services from suspicious paths
Get-WmiObject Win32_Service | Where {$_.PathName -notlike "*system32*" -and $_.PathName -notlike "*Program Files*"} | Select Name, PathName

# 3. Privileged groups
Get-ADGroupMember "Domain Admins" | Select Name, SamAccountName

# 4. WMI subscriptions
Get-WMIObject -Namespace root\subscription -Class __EventFilter
```

## All Persistence Locations

```
Scheduled Tasks         Get-ScheduledTask
Services                Get-WmiObject Win32_Service
Registry Run keys       HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
                        HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
                        HKCU:\... (same paths)
Winlogon                HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
                        (Shell, Userinit)
WMI Subscriptions       root\subscription namespace
Startup folders         C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup
                        C:\Users\*\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
Image File Execution    HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options
AdminSDHolder           CN=AdminSDHolder,CN=System,DC=...
DCSync ACLs             ACL on domain root with replication GUIDs
KRBTGT password age     Get-ADUser krbtgt -Properties PasswordLastSet
Rogue GPOs              Get-GPO -All | Sort ModificationTime -Descending
Local accounts          Get-LocalUser
Local admins            Get-LocalGroupMember Administrators
```

## Winlogon Should Be Exactly

```
Shell    = explorer.exe
Userinit = C:\Windows\system32\userinit.exe,
```

Anything appended = backdoor.

## DCSync Permission GUIDs

```
1131f6aa-9c07-11d1-f79f-00c04fc2dcd2   Replicating Directory Changes
1131f6ab-9c07-11d1-f79f-00c04fc2dcd2   Replicating Directory Changes All
```

Only DCs and Domain Admins should have these on the domain root.

## Red Flags

- Scheduled task running PowerShell with `-EncodedCommand`
- Service running from `C:\Users\`, `C:\Temp\`, or `%AppData%`
- Account in Domain Admins you don't recognize
- Any WMI subscription
- Recent file activity in `C:\Windows\System32`
- KRBTGT password older than the competition started
