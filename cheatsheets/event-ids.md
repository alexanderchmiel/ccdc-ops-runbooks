# Cheatsheet — Critical Event IDs

## Run on Every Rotation

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='Security'
  StartTime=(Get-Date).AddMinutes(-30)
  Id=@(4720,4724,4726,4728,4732,4756,4625,4648,4672)
} | Select TimeCreated, Id | Format-Table
```

## Priority

```
4728  Added to Domain Admins        CRITICAL — IR now
4726  User account deleted          CRITICAL — IR now
4720  New user created              HIGH
4724  Password reset attempted      HIGH
4732  Added to local Admins         HIGH
4756  Added to Universal group      HIGH
4648  Explicit credential logon     HIGH — lateral movement
4672  Special privileges assigned   HIGH — escalation
4725  User account disabled         HIGH — Red Team lockout?
4625  Failed logon                  MEDIUM — 5+ = brute force
4740  Account locked out            MEDIUM
4722  Account enabled               MEDIUM
4723  Password change attempted     MEDIUM
```

## Logon Types (Event 4624)

```
2     Console
3     Network (SMB)
4     Batch (scheduled task)
5     Service
7     Unlock
8     NetworkCleartext
9     NewCredentials (RunAs)
10    RDP
11    Cached
```

Type 3 from an unexpected source = lateral movement.

## PowerShell Log

Log: `Microsoft-Windows-PowerShell/Operational`

```
4103  Module logging
4104  Script block (the actual script)
4105  Script start
4106  Script stop
```

Encoded commands in 4104 = Red Team.
