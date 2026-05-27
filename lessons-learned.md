# Lessons Learned — NECCDC 2026

Notes on what went well, what didn't, and what we'd do differently.
Written after the fact. Some of this stings.

## What We Got Right

**OU structure and least privilege.** We built out the OU hierarchy early on
Day 1 and pushed users into the right buckets. Made GPO application
predictable. Worth the time investment.

**Persistence hunt before hardening.** Running the four-command check before
touching configs caught two scheduled tasks we would have hardened around
otherwise. Always hunt first.

**Black Team ticket templates.** When the POS went down, having a pre-written
ticket template saved time and kept the question competent enough that we
weren't charged points for it.

## What We Got Wrong

**Tried to build a trust from one side only.** First attempt at the cross-domain
trust failed because we only had RDP to one DC. Should have confirmed both DCs
were accessible before starting. Lost time we didn't have.

**Didn't access Keycloak until Day 2.** Keycloak federates auth for multiple
scored services. Should have been a Day 1 priority alongside the DC. Lesson:
identity infrastructure first, services second.

**Under-documented Day 1 GPO changes.** We configured PowerShell logging GPOs
but didn't screenshot the gpresult output on multiple hosts. Same exact mistake
the quals feedback called out. Documentation has to happen in real time, not
after the fact.

**Lost the POS for too long.** Once it was compromised we spent too long
trying to recover it ourselves before submitting the Black Team ticket. Should
have a hard time limit — 15 minutes of self-recovery attempts, then ticket.
The point cost of the ticket is less than the lost service score.

## What Surprised Us

**Red Team adjusted between days.** Day 2 attacks were different from Day 1.
They had watched our hardening and targeted what we hadn't gotten to yet.
This is obvious in retrospect.

**Scoring engine timing matters.** Restarting ADFS to apply a config change
caused a service check failure during the restart window. Plan service
restarts during quieter scoring windows if you can predict them.

**Business continuity scoring is significant.** Five points per failed
employee access check adds up. We were too aggressive on one firewall rule
and locked out a check. Less hardening would have scored higher than the
hardening that broke the check.

## What We'd Do Differently

**Pair on the trust.** Two people on the trust setup at once, one on each
DC. Don't try to do it serial across two RDP sessions.

**Identity stack first.** Keycloak, then DC, then services. Anything that
authenticates anything else gets priority.

**Document before hardening.** Open a screenshot folder and a notes file
before touching anything. Every change gets a before/after pair. Doesn't
matter if it looks excessive — the IR points come from documentation.

**Pre-write inject responses.** Have templated executive summary openings
ready. The technical work is easy. The professional writing under time
pressure is what kills scores.

**Don't trust your Day 1 hardening on Day 2.** Re-verify everything in the
first 10 minutes of Day 2. Red Team works overnight.

## What I Wish I'd Known Going In

Most of the points are in two places: documentation quality and business
impact communication. The technical work is necessary but not sufficient.
The teams that score highest can articulate *why* what they did matters to
a non-technical executive.

If I had to put one piece of advice on a sticky note for next year's team:
**screenshot everything with a visible timestamp, write the executive summary
first, fill in the technical work as you go.**
