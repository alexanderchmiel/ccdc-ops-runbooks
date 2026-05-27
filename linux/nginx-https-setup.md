# Nginx HTTPS Troubleshooting

When the HTTPS endpoint isn't responding or the cert won't validate, work
through these in order.

## First — What's Actually Failing

```bash
# From the local machine
curl -v https://localhost
curl -v https://<IP>

# From another host on the network
curl -v https://<IP>
curl -k https://<IP>   # -k skips cert validation
```

If `curl -k` works but `curl` doesn't, it's a cert trust issue, not TLS itself.
If both fail, the problem is deeper.

## Is Nginx Even Running

```bash
systemctl status nginx
ss -tlnp | grep -E ':80|:443'

# If failed to start
journalctl -xe -u nginx
nginx -t
```

`nginx -t` will tell you the exact line of any config error. Run it before
every reload.

## Config Sanity Check

```bash
nginx -t

# Find the active config
cat /etc/nginx/sites-enabled/*
ls -la /etc/nginx/sites-enabled/
ls -la /etc/nginx/conf.d/
```

Working baseline config:

```nginx
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name <IP_OR_HOSTNAME>;

    ssl_certificate     /etc/nginx/ssl/site.crt;
    ssl_certificate_key /etc/nginx/ssl/site.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

## Certificate Problems

### Does the cert exist?

```bash
ls -la /etc/nginx/ssl/
find / -name "*.crt" 2>/dev/null | grep -v proc
find / -name "*.pem" 2>/dev/null | grep -v proc
```

### Is it expired?

```bash
openssl x509 -in /etc/nginx/ssl/site.crt -noout -dates
openssl x509 -in /etc/nginx/ssl/site.crt -noout -subject -issuer
```

### Does the cert match the key?

```bash
openssl x509 -noout -modulus -in /etc/nginx/ssl/site.crt | md5sum
openssl rsa  -noout -modulus -in /etc/nginx/ssl/site.key | md5sum
```

The two md5 hashes MUST match. If different, the cert and key are mismatched
— common cause of "SSL handshake failed" errors.

### Generate a self-signed cert (if needed)

```bash
mkdir -p /etc/nginx/ssl

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/site.key \
  -out /etc/nginx/ssl/site.crt \
  -subj "/CN=<IP_OR_HOSTNAME>/O=Org/C=US"

chmod 600 /etc/nginx/ssl/site.key
chmod 644 /etc/nginx/ssl/site.crt
```

### Convert a provided .pfx (Windows format)

```bash
openssl pkcs12 -in certificate.pfx -nocerts -out site.key -nodes
openssl pkcs12 -in certificate.pfx -clcerts -nokeys -out site.crt
```

## Firewall

```bash
# iptables
iptables -L -n | grep -E '80|443'

# ufw
ufw status
ufw allow 80/tcp
ufw allow 443/tcp
```

From another host:

```powershell
Test-NetConnection -ComputerName <IP> -Port 443
```

## HTTP to HTTPS Redirect Not Working

```bash
# Is port 80 actually listening?
ss -tlnp | grep :80

# Is there a redirect block in the config?
grep -n "return 301" /etc/nginx/sites-enabled/*
```

If missing, add a separate server block on port 80 with `return 301`. Test and
reload:

```bash
nginx -t && systemctl reload nginx
```

## Quick Symptom Reference

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `curl -k` works, `curl` fails | Self-signed not trusted | Add to trust store |
| Connection refused on 443 | Nginx not listening on 443 | Add `listen 443 ssl` |
| Nginx fails to start | Config syntax or missing cert | `nginx -t` |
| Cert/key md5 mismatch | Wrong pair | Regenerate matching pair |
| HTTP works, HTTPS doesn't | Missing SSL block | Add 443 block |
| HTTPS works, HTTP doesn't redirect | No redirect block | Add `return 301` |
| Works locally, not from network | Firewall | Open 443 |

## Always Reload, Don't Restart

```bash
nginx -t && systemctl reload nginx
```

Reload is graceful — existing connections aren't killed. Restart kills active
sessions and counts against service uptime.
