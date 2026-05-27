# Finding Unknown Hosts on the Network

When you know a host exists somewhere but don't know its IP, work through these
in order. Each method is faster than the next — only fall through when one fails.

## Method 1 — Ask AD Directly

Fastest. Your DC already knows about other DCs through replication.

```powershell
Get-ADDomainController -Filter * |
  Select Name, IPv4Address, Site, Domain

# Forest-wide
Get-ADForest | Select Domains, Sites, GlobalCatalogs
Get-ADForest | Select -ExpandProperty GlobalCatalogs

# Across all domains
Get-ADForest | Select -ExpandProperty Domains | ForEach-Object {
  Write-Host "`n=== Domain: $_ ==="
  Get-ADDomainController -Filter * -Server $_ |
    Select Name, IPv4Address, Domain
}
```

If only one DC appears, replication may be broken — move to Method 2.

## Method 2 — DNS SRV Records

Every DC registers SRV records automatically. Even cross-subnet hosts are
findable this way if DNS is reachable.

```powershell
# DC discovery
Resolve-DnsName '_ldap._tcp.dc._msdcs.chefops.local' -Type SRV
Resolve-DnsName '_kerberos._tcp.dc._msdcs.chefops.local' -Type SRV

# nslookup equivalent
nslookup -type=SRV _ldap._tcp.dc._msdcs.chefops.local

# Get all A records in the zone
Get-DnsServerResourceRecord -ZoneName 'chefops.local' -RRType A
```

## Method 3 — Routing Table

Reveals what subnets are reachable.

```powershell
route print
Get-NetRoute | Select DestinationPrefix, NextHop | Where DestinationPrefix -ne "0.0.0.0/0"
arp -a
```

If you can ping the gateway, you can probably reach other subnets through it.

## Method 4 — Targeted Port Scan

Port 389 + 88 together on the same IP = DC. No other service combo matches.

```powershell
# Replace 10.X.X with the target subnet
1..254 | ForEach-Object {
  $ip = "10.X.X.$_"
  $ldap = Test-NetConnection -ComputerName $ip -Port 389 `
    -WarningAction SilentlyContinue -InformationLevel Quiet
  $kerb = Test-NetConnection -ComputerName $ip -Port 88 `
    -WarningAction SilentlyContinue -InformationLevel Quiet
  if ($ldap -and $kerb) { Write-Host "DC at: $ip" }
}
```

### Parallel scan for speed

```powershell
$jobs = 1..254 | ForEach-Object {
  $ip = "10.X.X.$_"
  Start-Job -ScriptBlock {
    param($ip)
    $r = Test-NetConnection -ComputerName $ip -Port 389 `
      -WarningAction SilentlyContinue -InformationLevel Quiet
    if ($r) { $ip }
  } -ArgumentList $ip
}
$jobs | Wait-Job | Receive-Job
$jobs | Remove-Job
```

## Method 5 — Find via Known Neighbor Service

If you know one host on the target subnet, scan for that to find the subnet,
then scan that subnet for what you actually want.

```powershell
# Gitea = port 3000
1..254 | ForEach-Object {
  $ip = "10.X.X.$_"
  $r = Test-NetConnection $ip -Port 3000 -WarningAction SilentlyContinue -InformationLevel Quiet
  if ($r) { Write-Host "Gitea/Grafana at: $ip" }
}

# Keycloak = 8080 or 8443
# Web services = 80 or 443
# SSH = 22
# RDP = 3389
```

## Method 6 — pfSense Console

If your team has the pfSense admin console open:

- Diagnostics → ARP Table (all recently seen hosts)
- Diagnostics → Routes (all known subnets)
- Interfaces → see connected networks

## Confirming It's What You Think It Is

```powershell
$ip = '<DISCOVERED_IP>'

# Run through every DC port
@(53,88,135,389,445,636,3268,3389) | ForEach-Object {
  $r = Test-NetConnection -ComputerName $ip -Port $_ -WarningAction SilentlyContinue
  Write-Host "Port $_ : $($r.TcpTestSucceeded)"
}

# If it's a DC, ask AD about it
Get-ADDomainController -Server $ip
```

## Decision Tree

| Situation | Method |
|-----------|--------|
| Haven't tried anything yet | 1 — Ask AD |
| AD only returns local DC | 2 — DNS SRV |
| DNS times out | 3 — Check routes |
| Have subnet, need IP | 4 — Port 389+88 scan |
| Subnet unknown, scan too slow | 5 — Find a known neighbor first |
| Team has pfSense access | 6 — pfSense ARP/routes |
| All methods fail | Ask Black Team (document prior attempts) |
