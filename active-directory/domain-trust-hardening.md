# Domain Trust Setup and Hardening

If you're setting up a trust between two domains, you need access to both DCs
simultaneously. Both sides have to agree to the trust relationship.

```
DC for domain A                DC for domain B
       |                              |
Run trust wizard here    AND   Run trust wizard here
       |______________________________|
                 Trust forms
```

Trying to build a trust from one side only is why our first attempt failed.

## Verifying What Already Exists

```powershell
Get-ADTrust -Filter * | Format-List *
```

Check:
- TrustType (External, Forest, Shortcut)
- TrustDirection (Bidirectional, Inbound, Outbound)
- TrustAttributes (transitive or not)
- SIDFilteringQuarantined (should be True)
- SelectiveAuthentication (should be True)

## Creating the Trust

On the first DC:

```powershell
# Verify which DC you're on first
Get-ADDomain | Select DNSRoot, PDCEmulator

netdom trust ad.chefops.local `
  /domain:ad.oceancrest.local `
  /twoway `
  /add `
  /passwordt:<TrustPassword>
```

Same on the other DC with arguments swapped. Use the SAME password on both sides
— that's the shared secret.

```powershell
# Verify
Get-ADTrust -Filter * | Select Name, TrustType, TrustDirection
```

## Hardening Immediately After Creation

The moment a trust goes up, Red Team will try to abuse it. Apply these the same
minute the trust is created.

### Enable SID Filtering

Without this, a compromised account in the trusted domain can forge SID history
to claim Domain Admin in the trusting domain. Full forest compromise vector.

```powershell
netdom trust ad.chefops.local `
  /domain:ad.oceancrest.local `
  /quarantine:yes

# Verify
Get-ADTrust -Filter * | Select Name, SIDFilteringQuarantined
```

### Enable Selective Authentication

Users from the trusted domain can only access resources you explicitly grant.
Without this, any user from one side can authenticate to anything on the other.

```powershell
netdom trust ad.chefops.local `
  /domain:ad.oceancrest.local `
  /SelectiveAuth:yes
```

## Audit Cross-Domain Group Memberships

```powershell
# Run from each DC
$groups = @('Domain Admins','Enterprise Admins','Schema Admins',
  'Backup Operators','DNSAdmins')

foreach ($g in $groups) {
  $members = Get-ADGroupMember -Identity $g -ErrorAction SilentlyContinue
  $cross = $members | Where-Object {$_.distinguishedName -like '*oceancrest*'}
  if ($cross) {
    Write-Host "CROSS-DOMAIN MEMBER IN $g" -ForegroundColor Red
    $cross | Select Name, SamAccountName
  }
}
```

Anything that turns up here is either Red Team persistence or misconfiguration.
Remove it.

## pfSense Coordination

For the trust to function, these ports need to be open between the two DCs:

```
DC-to-DC across pfSense:
  Port 53  TCP/UDP  - DNS
  Port 88  TCP/UDP  - Kerberos
  Port 389 TCP      - LDAP
  Port 445 TCP      - SMB
  Port 135 TCP      - RPC
  Port 49152-65535  - RPC dynamic range
```

Everything else between the two networks should be blocked at pfSense.

## Attack Path You're Protecting Against

```
Red Team compromises Kiosk (branch domain side)
            ↓
Bidirectional trust exists
            ↓
Red Team uses branch credentials to query primary domain LDAP
            ↓
Full domain enumeration
            ↓
Finds Kerberoastable service account
            ↓
Lateral movement to core infrastructure
```

SID filtering and selective auth break this chain at step 3.
