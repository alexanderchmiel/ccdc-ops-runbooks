# Windows Firewall Configuration

## Audit Current State

```powershell
# Are profiles enabled?
Get-NetFirewallProfile | Select Name, Enabled, DefaultInboundAction, DefaultOutboundAction

# All inbound allow rules
Get-NetFirewallRule -Direction Inbound -Action Allow -Enabled True |
  Select DisplayName, Profile | Sort DisplayName

# Find ANY/ANY rules (Red Team backdoors often look like this)
Get-NetFirewallRule -Direction Inbound -Action Allow -Enabled True |
  Where-Object {$_.Profile -eq "Any"} |
  Select DisplayName, Profile

# What ports are listening
netstat -ano | findstr LISTENING
```

## DC Required Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 53 | TCP/UDP | DNS |
| 88 | TCP/UDP | Kerberos |
| 135 | TCP | RPC endpoint mapper |
| 389 | TCP/UDP | LDAP |
| 445 | TCP | SMB (GPO via SYSVOL) |
| 464 | TCP/UDP | Kerberos password changes |
| 636 | TCP | LDAPS |
| 3268 | TCP | Global Catalog |
| 3269 | TCP | Global Catalog SSL |
| 3389 | TCP | RDP (keep open in cloud env) |
| 5985/5986 | TCP | WinRM |
| 443 | TCP | ADFS if applicable |
| 49152-65535 | TCP | RPC dynamic range |

## Block Specific IPs

```powershell
# Block one IP inbound
New-NetFirewallRule -DisplayName "Block Suspicious IP" `
  -Direction Inbound `
  -RemoteAddress 192.168.1.50 `
  -Action Block

# Block whole subnet
New-NetFirewallRule -DisplayName "Block Subnet" `
  -Direction Outbound `
  -RemoteAddress 192.168.10.0/24 `
  -Action Block

# Block multiple IPs at once
New-NetFirewallRule -DisplayName "Block Multiple IPs" `
  -Direction Inbound `
  -RemoteAddress @("192.168.1.50", "192.168.1.51", "10.0.0.25") `
  -Action Block

# Block IP on specific port
New-NetFirewallRule -DisplayName "Block LDAP from Host" `
  -Direction Inbound `
  -Protocol TCP -LocalPort 389 `
  -RemoteAddress 192.168.10.15 `
  -Action Block
```

## During Active Incident

When you catch a Red Team IP in logs, timestamp the rule name. It becomes
evidence for the IR report.

```powershell
New-NetFirewallRule -DisplayName "Block RedTeam - $(Get-Date -Format 'HHmm')" `
  -Direction Inbound `
  -RemoteAddress <SUSPICIOUS_IP> `
  -Action Block `
  -Profile Any
```

Screenshot immediately. That's your evidence of remediation.

## Block Known-Bad Ports

```powershell
# Telnet
New-NetFirewallRule -DisplayName "Block Telnet" -Direction Inbound `
  -Protocol TCP -LocalPort 23 -Action Block

# FTP plain text
New-NetFirewallRule -DisplayName "Block FTP" -Direction Inbound `
  -Protocol TCP -LocalPort 21 -Action Block

# SMBv1 on Public profile only (keep on Domain for SYSVOL)
New-NetFirewallRule -DisplayName "Block SMB Public" -Direction Inbound `
  -Protocol TCP -LocalPort 445 -Action Block -Profile Public

# Disable SMBv1 entirely
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```

## Removing and Disabling Rules

```powershell
# List all block rules
Get-NetFirewallRule -Action Block | Select DisplayName, Direction, Enabled

# Remove a rule
Remove-NetFirewallRule -DisplayName "Block Suspicious IP"

# Disable without deleting
Set-NetFirewallRule -DisplayName "Block Suspicious IP" -Enabled False
```

## Important Notes

Before blocking any IP, confirm it's not:
- A scoring engine check source
- A Black Team management IP
- An employee/business continuity check source

Blocking the scoring engine fails every subsequent service check for that
service. Verify with your team before unilateral blocks.

In a cloud environment, do not lock down RDP or block your own management
path. There is no physical console to fall back on.
