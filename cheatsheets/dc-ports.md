# Cheatsheet — Domain Controller Ports

## Must Stay Open

```
53     TCP/UDP  DNS
88     TCP/UDP  Kerberos
135    TCP      RPC endpoint mapper
389    TCP/UDP  LDAP
445    TCP      SMB (SYSVOL/GPO)
464    TCP/UDP  Kerberos password changes
636    TCP      LDAPS
3268   TCP      Global Catalog
3269   TCP      Global Catalog SSL
3389   TCP      RDP — keep open in cloud
5985   TCP      WinRM HTTP
5986   TCP      WinRM HTTPS
443    TCP      ADFS (if used)
49152-65535     RPC dynamic range
```

## Kiosk/POS Minimum

```
53     TCP/UDP  DNS
88     TCP/UDP  Kerberos
389    TCP      LDAP
445    TCP      SMB
```

## Always Block

```
21     FTP
23     Telnet
139    NetBIOS
137    NetBIOS name (UDP)
138    NetBIOS datagram (UDP)
25     SMTP (DC isn't mail)
80     HTTP (DC isn't web)
```

## Quick Commands

```powershell
# Audit
Get-NetFirewallProfile | Select Name, Enabled, DefaultInboundAction
Get-NetFirewallRule -Direction Inbound -Action Allow -Enabled True

# Block an IP fast
New-NetFirewallRule -DisplayName "Block-$(Get-Date -Format HHmm)" `
  -Direction Inbound -RemoteAddress <IP> -Action Block

# Disable SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
```
