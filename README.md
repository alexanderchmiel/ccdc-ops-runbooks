# CCDC Blue Team Operations Reference

Notes and runbooks built during preparation for and live play at NECCDC 2026 Regionals.
Written to be useful under pressure — commands first, explanation second.

Not a tutorial. A reference.

## What's Here

- Active Directory hardening from first login through full lockdown
- Persistence hunting methodology
- Domain trust setup and what to harden the moment it goes live
- Windows firewall configuration
- POS and kiosk network isolation
- Keycloak identity management notes
- Nginx HTTPS troubleshooting
- Incident response templates
- PDF runbooks used during live competition

## Context

Built for a Windows/Linux hybrid environment running:

Active Directory · pfSense · Keycloak · Nginx · Grafana · Gitea · Semaphore · Teleport · Windows POS · Kiosks

Scenario: MSP protecting a food services client across two domains.

## Structure

```
active-directory/    AD hardening, persistence hunting, trust setup, kiosk isolation
windows/             Firewall config, services to disable, POS hardening
linux/               Nginx TLS troubleshooting
identity/            Keycloak access and hardening
incident-response/   IR template and event ID reference
network/             Finding hosts when you don't know the subnet
runbooks/            Printable PDFs for competition day
cheatsheets/         Single-page quick references
```

## Usage

Each file is self-contained. Skim the table of contents in each one and jump to what you need.

The runbooks folder has the printable PDFs we actually used during the live event.

## Notes

These are competition notes, not a polished knowledge base. There are rough edges, things we
never got to, and decisions made under time pressure. Left in as written.
