# Kiosk and POS Isolation

Kiosks and POS terminals are the lowest-trust devices on the network. They need
just enough domain connectivity to authenticate, nothing else.

Real-world breaches like Target 2013 started exactly this way — attackers got
into POS systems through a vendor network and pivoted to the main domain.

## Attack Chain You're Preventing

```
Red Team compromises kiosk
        ↓
Kiosk can reach DC on port 389 (LDAP)
        ↓
Red Team enumerates all users, groups, SPNs
        ↓
Kerberoasting attack from kiosk
        ↓
Cracks service account password offline
        ↓
Lateral movement to core infrastructure
```

Cut this chain at step 2.

## OU Setup

```powershell
New-ADOrganizationalUnit -Name "Kiosks" -Path "DC=chefops,DC=local"
New-ADOrganizationalUnit -Name "POS_Terminals" -Path "DC=chefops,DC=local"
New-ADOrganizationalUnit -Name "Windows_POS_Accounts" -Path "DC=chefops,DC=local"

# Move kiosk computer objects
Move-ADObject -Identity "CN=KIOSK01,CN=Computers,DC=chefops,DC=local" `
  -TargetPath "OU=Kiosks,DC=chefops,DC=local"
```

## Dedicated Kiosk Service Account

Not a local account. Not a regular domain user. A purpose-built kiosk account
with the minimum possible permissions.

```powershell
New-ADUser -Name "kiosk-svc" `
  -SamAccountName "kiosk-svc" `
  -UserPrincipalName "kiosk-svc@chefops.local" `
  -Path "OU=Windows_POS_Accounts,DC=chefops,DC=local" `
  -AccountPassword (ConvertTo-SecureString "K!0sk$ecure2026!!" -AsPlainText -Force) `
  -Enabled $true `
  -PasswordNeverExpires $true `
  -CannotChangePassword $true
```

`PasswordNeverExpires` is intentional. If the kiosk password expires overnight,
the restaurant can't open in the morning. That's a business continuity failure.

## Restrictive GPO for the Kiosk OU

Create a GPO linked specifically to the Kiosks OU.

### Deny network access from sensitive systems

`Computer Configuration > Windows Settings > Security Settings > Local Policies > User Rights Assignment`

- "Access this computer from the network" — Remove Everyone, allow only SYSTEM,
  Administrators, and your specific service account

### Deny interactive logon for domain admins on kiosks

- "Deny log on locally" — add Domain Admins, Enterprise Admins
- "Deny log on through Remote Desktop Services" — add Domain Admins

If Red Team gets onto a kiosk, they can't use cached admin credentials.

### Disable cmd, PowerShell, scripting hosts

`User Configuration > Administrative Templates > System`

- "Don't run specified Windows applications" — Enable, add:
  - `cmd.exe`
  - `powershell.exe`
  - `powershell_ise.exe`
  - `wscript.exe`
  - `cscript.exe`
  - `mshta.exe`
  - `regedit.exe`

## Firewall Rules — Outbound from Kiosk

Block everything except what authentication strictly needs.

```powershell
# Block kiosk-to-kiosk (no lateral SMB between kiosks)
New-NetFirewallRule -DisplayName "Block Kiosk to Kiosk" `
  -Direction Outbound -Protocol TCP `
  -RemoteAddress <KIOSK_SUBNET> -Action Block

# Block kiosk reaching core ChefOps network
New-NetFirewallRule -DisplayName "Block Kiosk to ChefOps Core" `
  -Direction Outbound -Protocol TCP `
  -RemoteAddress <CHEFOPS_CORE_SUBNET> -Action Block

# Block RDP outbound
New-NetFirewallRule -DisplayName "Block RDP Outbound" `
  -Direction Outbound -Protocol TCP -RemotePort 3389 -Action Block

# Block WinRM outbound
New-NetFirewallRule -DisplayName "Block WinRM Outbound" `
  -Direction Outbound -Protocol TCP -RemotePort 5985,5986 -Action Block
```

What stays allowed (minimum for domain auth):

```
Port 88 TCP/UDP  - Kerberos
Port 53 TCP/UDP  - DNS
Port 445 TCP     - SMB (for GPO via SYSVOL)
Port 389 TCP     - LDAP
```

## Inbound to Kiosks

Only the DC should ever initiate a connection to a kiosk.

```powershell
Set-NetFirewallProfile -Profile Domain -DefaultInboundAction Block

New-NetFirewallRule -DisplayName "Allow DC Management Inbound" `
  -Direction Inbound -Protocol TCP `
  -RemoteAddress <DC_IP> -Action Allow
```

## pfSense Coordination

Defense in depth — the firewall rules above are host-level. pfSense enforces
the same at network level so even if Red Team modifies Windows Firewall on
the kiosk, the packets don't leave the subnet.

```
BLOCK: Kiosk subnet → ChefOps Core subnet (any port)
BLOCK: Kiosk subnet → Kiosk subnet (lateral between kiosks)
ALLOW: Kiosk subnet → DC IP (88, 53, 389, 445 only)
ALLOW: DC → Kiosk subnet (management)
BLOCK: Kiosk subnet → any other destination (default deny)
```

## Three-Wall Mental Model

```
Wall 1 — GPO (what the kiosk account can DO)
Wall 2 — Windows Firewall via GPO (where the kiosk can REACH)
Wall 3 — pfSense (enforced at NETWORK level regardless of host config)
```

Red Team breaks Wall 1 by compromising the OS → hits Wall 2. Modifies Wall 2
on the host → hits Wall 3 at the router. No single control failure leads to
full compromise.
