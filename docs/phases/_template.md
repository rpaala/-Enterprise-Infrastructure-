# Phase N: <Title>

**Status:** Not started | In progress | Done
**Dates:** <start> to <end>
**Effort (actual):** <evenings/weekends>

## Goal

What this phase set out to prove, in one or two sentences.

## What I built

- Components, tools and versions
- Links to code in this repo (e.g. `terraform/`, `ansible/`)

## Test / exit criteria

Copy this phase's exit checks from the [phase plan](../phase-plan.md). Every check needs a pass before the next phase starts. A failed check goes in the troubleshooting log, then gets fixed and run again.

| # | Check | How to test (command or action) | Expected result | Result | Date | Evidence |
|---|---|---|---|---|---|---|
| 1 | | | | Pass / Fail | | Screenshot, log or link |
| 2 | | | | | | |

- [ ] All exit checks pass
- [ ] Regression: earlier phases' key checks still pass (e.g. backups run, VPN connects, cluster nodes Ready)
- [ ] Evidence saved (screenshots blurred, logs sanitized)

**Go / no-go:** Go | No-go. Blocking issue, if any: <one line>

## Architecture decision

What I chose, the alternatives, and why. (Copy to `docs/decisions/` as an ADR.)

## Security decision

Only if the phase touched security: the threat, the control, the trade-off.

## Diagram

Link to an updated diagram in `diagrams/`. Use placeholders only.

## Troubleshooting log

| Date | Issue | Resolution |
|---|---|---|
| | | |

## Lessons learned

-

## Interview evidence

- Screenshot, recording, pipeline run, or measured result (e.g. "restored a VM in 7 minutes")

## Pre-commit privacy check

- [ ] No real IPs, subnets, hostnames, domains, MACs, serials, usernames, keys or tokens
- [ ] Screenshots blurred (addresses, hostnames, account names)
- [ ] gitleaks passes locally (`gitleaks detect --no-git -v`)
