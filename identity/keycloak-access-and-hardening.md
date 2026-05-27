# Keycloak Access and Hardening

Keycloak provides SSO for services behind it. If it goes down, every service
depending on it for auth starts failing checks.

## Finding It

```powershell
# DNS first
Resolve-DnsName keycloak.chefops.local
Resolve-DnsName keycloak

# AD computer objects
Get-ADComputer -Filter * | Select Name, IPv4Address

# Scan the subnet for Keycloak ports
1..254 | ForEach-Object {
    $ip = "10.X.X.$_"
    $r8080 = Test-NetConnection $ip -Port 8080 -WarningAction SilentlyContinue -InformationLevel Quiet
    $r8443 = Test-NetConnection $ip -Port 8443 -WarningAction SilentlyContinue -InformationLevel Quiet
    if ($r8080 -or $r8443) {
        Write-Host "Keycloak possibly at: $ip"
    }
}
```

## Accessing the Admin Console

| URL | Version |
|-----|---------|
| `http://<IP>:8080/auth/admin` | Keycloak 16 and older |
| `http://<IP>:8080/admin` | Keycloak 17+ (Quarkus) |
| `https://<IP>:8443/auth/admin` | HTTPS old |
| `https://<IP>/admin` | If proxied behind 443 |

Try the root URL too — newer Keycloak has a landing page with an Administration
Console link.

## Default Credentials to Try

```
admin / admin
admin / password
admin / keycloak
keycloak / keycloak
```

## If All Credentials Fail

If you have SSH/console to the Keycloak host:

```bash
# Keycloak 17+
cd /opt/keycloak/bin
./kc.sh start-dev &

./kcadm.sh config credentials \
  --server http://localhost:8080 \
  --realm master \
  --user admin \
  --password admin
```

```bash
# Older Keycloak (Wildfly)
/opt/jboss/keycloak/bin/add-user-keycloak.sh -u admin -p NewPassword123!
systemctl restart keycloak
```

## If It's Down

```bash
systemctl status keycloak

# If running in Docker
docker ps | grep keycloak
docker logs <CONTAINER_ID>

# Process check
ps aux | grep keycloak

# Logs
journalctl -u keycloak -n 50
```

## First Actions Once Inside

### Change admin password

Top right → Admin → Manage Account → Password

### Check the realms

Left sidebar has a dropdown at the top showing "Master". You should see another
realm for your environment. Don't make changes in the Master realm.

### Verify LDAP federation

Your Realm → User Federation → ldap entry → Test Connection / Test Authentication

If broken, users can't auth through Keycloak to anything. Fix the bind credentials
or URL:

```
Connection URL:    ldap://<DC_IP>:389
Bind DN:           CN=Administrator,CN=Users,DC=chefops,DC=local
Bind Credential:   <domain admin password>
Users DN:          CN=Users,DC=chefops,DC=local
```

### Check clients

Your Realm → Clients. Each entry is a scored service. Note them all. Do NOT
delete any.

### Check users

Your Realm → Users → View all users. Anyone you don't recognize, especially
with admin roles, investigate and remove.

## Harden It

```
Your Realm → Realm Settings → Login tab
  Brute Force Detection → ON
  Max Login Failures → 5
  Wait Increment → 60 seconds

Your Realm → Realm Settings → Sessions tab
  Review and shorten token timeouts if they're long
```

## Why It Matters

Keycloak ties into AD. If Red Team owns the Keycloak admin account, they can:

- Disable users in bulk (service check failures)
- Change client configurations to break SSO
- Inject themselves into the user list with admin roles
- Re-route authentication to a fake LDAP

LDAP federation health is the single most important thing to verify. Everything
else is secondary.
