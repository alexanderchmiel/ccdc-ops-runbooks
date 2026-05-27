# Incident Response Report Template

The most common failure on IR reports is technical output without business
context. The grader is reading as if they're a CISO. Write accordingly.

## Structure

```
1. Executive Summary
2. Timeline
3. Technical Details (Supporting Artifacts)
4. Root Cause
5. Remediation Taken
6. Prevention
```

## Executive Summary

Plain language. What service or business function was affected. No jargon.
Target audience is a non-technical executive.

Bad:
> A 4728 event was generated indicating a user was added to the Domain Admins
> group. The account was disabled and removed from the group.

Better:
> An unauthorized administrative account was created on our identity system at
> 10:47 AM. This account had full access to all employee credentials and
> business data. The account has been removed and the credential system has
> been secured. No business operations were impacted during the incident.

## Timeline

Chronological. Every event with a timestamp. Detection → confirmation → action → verification.

```
10:42:13  Security event 4728 generated on DC02
10:42:30  Alert triaged — confirmed unauthorized
10:43:15  Account "newadmin" disabled
10:43:45  Account removed from Domain Admins group
10:44:20  Source IP identified as 10.X.X.X
10:44:35  Source IP blocked at Windows Firewall
10:45:00  Source IP blocked at pfSense
10:47:00  Full domain audit confirms no other rogue accounts
10:50:00  KRBTGT password reset (first of two)
11:00:00  KRBTGT password reset (second)
```

## Technical Details — Supporting Artifacts

This is where the screenshots, event log exports, and command output go.
Everything that proves the incident occurred and that you remediated it.

- Event log export showing the 4728 event
- `Get-ADUser` output showing the rogue account before removal
- Firewall rule screenshot with timestamp in the rule name
- pfSense log showing connection from source IP
- Group membership output after remediation

## Root Cause

What allowed this to happen. Be specific.

Bad:
> The attacker created an account.

Better:
> The ms-DS-MachineAccountQuota attribute was set to the default value of 10,
> allowing any authenticated user to add machines to the domain. The attacker
> leveraged a compromised employee credential to add a rogue machine, then
> used that machine to escalate privileges via [specific technique].

## Remediation Taken

Exactly what you did. Past tense. Specific commands or actions.

- Disabled rogue account "newadmin"
- Removed account from Domain Admins group
- Reset KRBTGT password twice with 10-minute interval
- Set ms-DS-MachineAccountQuota to 0
- Created GPO restricting machine joins to administrators only
- Blocked source IP at both Windows Firewall and pfSense

## Prevention

What configuration prevents recurrence. This is the part that scores well.

- ms-DS-MachineAccountQuota permanently set to 0
- GPO "Restrict_Machine_Joins" linked to domain root
- Monitoring rule added for event ID 4728 with immediate alert
- Quarterly privileged group audit added to operations checklist

## What Made the Difference at Quals

The team got points docked for:
- Brief, list-based summaries
- "Bare minimum wording"
- No business impact narrative
- Technical detail in the summary section instead of artifacts

The team got points for:
- Clear identification of root cause
- Forensic timestamps
- Removed access after compromise

Write the summary like you're emailing a CISO. Put technical detail in artifacts.
Show timestamps for everything. Document what configuration change prevents
recurrence.

## Submission Timing

Submit reports while the incident is still active, not after full remediation.
The competition rules say IR reports submitted during active incidents recover
more points than ones submitted retroactively.

Draft the executive summary the moment you see the alert. Fill in remediation
as you do it. Don't wait until the end to start writing.
