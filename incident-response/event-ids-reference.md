# Windows Security Event IDs — Quick Reference

The events that matter most when hunting active Red Team activity. Run this
on a 30-minute rotation.

## Pull Critical Events

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    StartTime = (Get-Date).AddMinutes(-30)
    Id        = @(4720,4722,4723,4724,4725,4726,4728,4732,4740,4756,4625,4648,4672,4776)
} | Select TimeCreated, Id, Message | Format-List
```

## Severity Reference

| Event ID | Meaning | Severity | Action |
|----------|---------|----------|--------|
| 4720 | New user account created | HIGH | Verify you created it. If not, disable and IR |
| 4722 | User account enabled | MEDIUM | Check if you enabled it |
| 4723 | Password change attempted | MEDIUM | Watch for brute force pattern |
| 4724 | Password reset attempted | HIGH | Should only be from admins |
| 4725 | User account disabled | HIGH | Is Red Team locking accounts? |
| 4726 | User account deleted | CRITICAL | IR immediately |
| 4728 | Added to Domain Admins | CRITICAL | IR immediately, do not wait |
| 4732 | Added to local Administrators | HIGH | Investigate source machine |
| 4740 | Account locked out | MEDIUM | Brute force indicator |
| 4756 | Added to Universal group | HIGH | Check the group and the new member |
| 4625 | Failed logon | MEDIUM | 5+ in succession = brute force |
| 4648 | Explicit credential logon | HIGH | Lateral movement indicator |
| 4672 | Special privileges assigned | HIGH | Privilege escalation indicator |
| 4776 | NTLM credential validation | LOW–MEDIUM | Watch for anomalous sources |
| 4624 | Successful logon | LOW | Watch logon types — type 3 = network, type 10 = RDP |

## Logon Type Reference (Event 4624)

| Type | Meaning |
|------|---------|
| 2 | Interactive (console) |
| 3 | Network (SMB, file share) |
| 4 | Batch (scheduled task) |
| 5 | Service |
| 7 | Unlock |
| 8 | NetworkCleartext (RDP w/ stored creds) |
| 9 | NewCredentials (RunAs) |
| 10 | RemoteInteractive (RDP) |
| 11 | CachedInteractive |

Type 3 from an unexpected source IP is the lateral movement smell test.

## PowerShell Event IDs

Different log: `Microsoft-Windows-PowerShell/Operational`

| Event ID | Meaning |
|----------|---------|
| 4103 | Module logging — function/cmdlet calls |
| 4104 | Script block logging — actual script content |
| 4105 | Script start |
| 4106 | Script stop |

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-PowerShell/Operational' `
  -MaxEvents 50 | Where-Object {$_.Id -eq 4104} |
  Select TimeCreated, Message | Format-List
```

If you see encoded commands or obfuscated PowerShell in 4104 events, that's
Red Team activity.

## Specific Hunt Queries

### New computer object in last 30 minutes (gamingpc scenario)

```powershell
Get-ADComputer -Filter * -Properties Created |
  Where-Object {$_.Created -gt (Get-Date).AddMinutes(-30)} |
  Select Name, Created, DistinguishedName
```

### Anyone added to Domain Admins recently

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'; Id=4728
    StartTime=(Get-Date).AddHours(-2)
} | Select TimeCreated, Message | Format-List
```

### Brute force pattern

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'; Id=4625
    StartTime=(Get-Date).AddMinutes(-30)
} | Group-Object {$_.Properties[19].Value} |
  Where-Object {$_.Count -gt 5} |
  Select Name, Count
```

5+ failed logons from the same source IP in 30 minutes is brute force.
Block immediately.

## Monitoring Script

Drop this in a PowerShell window and let it run:

```powershell
function Watch-SecurityEvents {
    while ($true) {
        Clear-Host
        Write-Host "[$(Get-Date -Format 'HH:mm:ss')] Checking..." -ForegroundColor Cyan

        $events = Get-WinEvent -FilterHashtable @{
            LogName   = 'Security'
            StartTime = (Get-Date).AddMinutes(-5)
            Id        = @(4720,4724,4726,4728,4732,4756)
        } -ErrorAction SilentlyContinue

        if ($events) {
            Write-Host "ALERTS:" -ForegroundColor Red
            $events | Select TimeCreated, Id | Format-Table
        } else {
            Write-Host "No alerts" -ForegroundColor Green
        }

        Start-Sleep -Seconds 60
    }
}

Watch-SecurityEvents
```
