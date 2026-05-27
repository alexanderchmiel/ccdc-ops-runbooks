# POS Hardening After Redeployment

When a POS gets compromised and redeployed, Red Team is already scanning by the
time you have RDP. You have minutes before re-compromise. Order matters.

## Don't Join the Domain Yet

Lock down the firewall first. Joining an unhardened machine to AD gives Red Team
a foothold the moment it connects.

## Change Local Passwords First

```powershell
net user Administrator NewStr0ngP@ss2026!

Get-LocalUser | Select Name, Enabled, LastLogon

# Disable anything you don't recognize
Disable-LocalUser -Name "suspiciousaccount"

# Check local admins
Get-LocalGroupMember -Group "Administrators"
```

## Firewall Lockdown

```powershell
# Default block inbound
Set-NetFirewallProfile -Profile Domain,Private,Public `
  -Enabled True -DefaultInboundAction Block

# Allow RDP only from DC IP
New-NetFirewallRule -DisplayName "Allow RDP from DC" `
  -Direction Inbound -Protocol TCP -LocalPort 3389 `
  -RemoteAddress 10.100.6.64 -Action Allow

# Allow WinRM only from DC
New-NetFirewallRule -DisplayName "Allow WinRM from DC" `
  -Direction Inbound -Protocol TCP -LocalPort 5985,5986 `
  -RemoteAddress 10.100.6.64 -Action Allow

# Block lateral outbound
New-NetFirewallRule -DisplayName "Block RDP Outbound" `
  -Direction Outbound -Protocol TCP -RemotePort 3389 -Action Block

New-NetFirewallRule -DisplayName "Block WinRM Outbound" `
  -Direction Outbound -Protocol TCP -RemotePort 5985,5986 -Action Block

# Block SMB outbound to anything except DC
New-NetFirewallRule -DisplayName "Block SMB Lateral" `
  -Direction Outbound -Protocol TCP -RemotePort 445 -Action Block

New-NetFirewallRule -DisplayName "Allow SMB to DC" `
  -Direction Outbound -Protocol TCP -RemotePort 445 `
  -RemoteAddress 10.100.6.64 -Action Allow

# No internet
New-NetFirewallRule -DisplayName "Block Internet Outbound" `
  -Direction Outbound -RemoteAddress Internet -Action Block
```

## Disable Unnecessary Services

```powershell
Stop-Service Spooler -Force
Set-Service Spooler -StartupType Disabled

Stop-Service RemoteRegistry -Force
Set-Service RemoteRegistry -StartupType Disabled

Stop-Service Fax -Force
Set-Service Fax -StartupType Disabled

# IIS — POS isn't a web server
Stop-Service W3SVC -Force -ErrorAction SilentlyContinue
Set-Service W3SVC -StartupType Disabled -ErrorAction SilentlyContinue

# Windows Search
Stop-Service WSearch -Force -ErrorAction SilentlyContinue
Set-Service WSearch -StartupType Disabled -ErrorAction SilentlyContinue

# Old PowerShell
Disable-WindowsOptionalFeature -Online -FeatureName MicrosoftWindowsPowerShellV2Root
```

## What's Listening?

```powershell
netstat -ano | findstr LISTENING

# Map PID to process
Get-Process | Select Id, ProcessName | Sort Id

# More targeted
Get-NetTCPConnection -State Listen |
  Where-Object {$_.LocalPort -eq <PORT>} |
  ForEach-Object {
    $proc = Get-Process -Id $_.OwningProcess
    [PSCustomObject]@{Port=$_.LocalPort; PID=$_.OwningProcess; Process=$proc.Name}
  }
```

Anything unexpected? Investigate before joining the domain.

## Teleport Compromise Scenarios

Teleport is the remote access gateway. If Red Team owns Teleport, they own
every machine it touches.

### Detect compromised Teleport

```powershell
Get-Service teleport -ErrorAction SilentlyContinue

# Check Teleport config
Get-Content 'C:\ProgramData\Teleport\teleport.yaml' -ErrorAction SilentlyContinue

# Unexpected outbound connections from Teleport process
$tp = (Get-Process teleport -EA SilentlyContinue).Id
Get-NetTCPConnection -State Established |
  Where-Object {$_.OwningProcess -eq $tp} |
  Select LocalAddress, LocalPort, RemoteAddress, RemotePort
```

### Teleport spawning cmd.exe or PowerShell

That's an active session. Investigate:

```powershell
Get-WmiObject Win32_Process |
  Where-Object {$_.ParentProcessId -eq $tp} |
  Select Name, ProcessId, CommandLine
```

### Kill an active Red Team session

```powershell
query session
query user
logoff <SESSION_ID>

# Block the source IP
New-NetFirewallRule -DisplayName "Block RT Session $(Get-Date -Format HHmm)" `
  -Direction Inbound -RemoteAddress <SOURCE_IP> -Action Block
```

Screenshot first. That's IR evidence.

### Rotate Teleport tokens (from Teleport auth server)

```bash
tctl tokens ls
tctl tokens rm <TOKEN_ID>
tctl tokens add --type=node --ttl=1h
```

## Domain Join (After Hardening)

```powershell
Add-Computer -DomainName "chefops.local" `
  -Credential (Get-Credential) `
  -OUPath "OU=POS_Terminals,DC=chefops,DC=local" `
  -Restart
```

Specify the OU path so the POS lands in POS_Terminals and gets the restrictive
GPO applied immediately.

After reboot:

```powershell
gpresult /r /scope computer
gpupdate /force
```

## Ongoing Monitoring

Every 30 minutes:

```powershell
# Active sessions
query user

# New listeners?
netstat -ano | findstr LISTENING

# Recent logons
Get-WinEvent -FilterHashtable @{
  LogName='Security'; Id=@(4624,4625,4648)
  StartTime=(Get-Date).AddMinutes(-30)
} | Select TimeCreated, Id, Message | Format-List
```

Signs of re-compromise:
- New process from an unexpected parent
- New listening port
- Outbound connection to unknown IP
- New local admin account
- Teleport spawning cmd or PowerShell
