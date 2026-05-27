# Runbooks

Printable PDFs used during live competition. Designed for quick reference
under pressure — every PowerShell command, every what-if scenario, every
fallback path.

## Files

| File | When to Use |
|------|-------------|
| `Runbook_1_DC01_Hardening.pdf` | First access to an uncompromised DC. Full sequence from persistence hunt through hardening. |
| `Runbook_2_POS_Hardening.pdf` | After Black Team redeploys a compromised POS. Order matters — firewall before domain join. |
| `Runbook_3_Finding_DC01.pdf` | Methodology for discovering hosts on unknown subnets. Six methods in priority order. |
| `Runbook_4_DC02_Hardening.pdf` | Day 2 on a DC you already hardened. Re-verification first, then items commonly missed under Day 1 time pressure. |

## Format Notes

Each PDF follows the same structure:

- Cover with version and scope
- Phased approach (numbered phases work in order)
- Code blocks with the exact PowerShell needed
- What-if callouts for each failure scenario
- Tables for symptom → cause → fix lookups
- Color coding: green = note, yellow = warning, red = danger

Built for printing single-sided so you can flip through during competition.
