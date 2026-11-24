# COE: The idea-tending roles failed on an expired login for 18 days while reporting success

> **Date of incident:** 2026-09-18 to 2026-10-05 (also 2026-06-20 to 06-21 and 2026-08-18 to 08-20)
> **Date of writeup:** 2026-10-12 (requested by RED in the 2026-H1 COE review)
> **Author(s):** CLI Claude (Opus 5.5) + RED
> **Status:** Reviewed. Fixes done with evidence; closes when its controls have passed for 30 days (from 2026-10-06).
> **Source:** [internal record - withheld]

*A Correction of Error (COE) is a blameless incident write-up: timeline, five whys, contributing factors, fix, lessons, action items. This public copy has private specifics removed; each removal is marked.*

## Summary

[internal space - withheld] is a system in which scheduled Claude roles tend a collection of ideas, each role on its own schedule ([role names - withheld]; ten in all). Every one of them runs Claude without a person present, using a stored login. When that login expired, every run printed "failed to authenticate" and did nothing, but each launcher script ended with a harmless final command that always succeeds, so the scheduler recorded success. For 18 days the idea-tending system was asleep and nothing said so. A Claude Mind doing its own scheduled audit found it; RED logging in again fixed it; the launchers now fail loudly.

## Timeline

- **2026-06-20 to 06-21**: first episode (an authentication error), 2 days. Unnoticed at the time.
- **2026-08-18 to 08-20**: second episode, 3 days. Unnoticed at the time.
- **2026-09-08 to 09-18**: a separate run gap (no runs at all), noted later; cause not recorded.
- **2026-09-18 03:30**: third episode starts. Every role fails every run; the scheduler reports success throughout.
- **2026-10-05**: a scheduled Claude Mind tending [internal space - withheld] runs a reachability audit, finds the failures in the roles' logs, and flags them for RED. It does not try to fix them itself: that was outside its standing, and the fix needed a human to log in.
- **2026-10-05**: RED logs in again; an unattended run is verified to work.
- **2026-10-06 03:30**: first normal run after the fix. The launchers were rewritten to fail loudly the same day.

(Dates from the roles' own logs: the "starting" lines before each authentication failure.)

## Impact / Blast Radius

- **Users affected:** none external. The idea-tending system, and a shared space for the Minds (one role could not wake to read a proposal waiting for it there).
- **Duration:** 18 days (plus 2 and 3 days earlier).
- **User-visible symptom:** none. That is the problem.
- **Data impact:** none lost; the tending simply did not happen.

## Root Cause (5 Whys)

1. **Why did the system stop working?** The stored login expired, so every unattended Claude call failed.
2. **Why did nobody know?** The scheduler said every run succeeded.
3. **Why did it say success?** Each launcher's last command was one that always succeeds, so the script's exit code was always 0, whatever happened before it.
4. **Why was that never caught?** No check looked at what the roles *did*, only that they ran. And the system was not in the registry of long-running processes, so no one was looking at all.
5. **Root:** a job's success was defined as **"the script finished", not "the work happened"**. The same swallow-and-report-success shape as COE 2026-07-27, whose audit for it (action item `2026-07-27#6`) had not run yet.

## Contributing Factors

- The login expires [cycle details withheld], and nothing renews it unattended.
- The earlier, shorter episodes healed themselves or were renewed by an interactive session, so the pattern never got a name.

## Detection

- **Found by:** a Claude Mind's own scheduled audit, not by monitoring.
- **Sooner?** Yes: any check of "did the last run do work" would have caught it on day 1.

## The Fix

- **Code:** every Claude-backed launcher now captures its output, treats a nonzero exit **or** an authentication-failure message as FAILED, logs it, fires a desktop notification, and exits nonzero (with a distinct code for authentication). Tested with a stand-in for Claude in three cases: failure, an authentication message with a success exit code, and real success. The original launchers were kept as a backup.
- **Process:** the system is now in the registry of long-running processes, with a daily cadence.

## Lessons Learned

- **Pattern:** silent success (pattern A in the 2026-H1 review).
  **Why it matters:** a job that cannot tell "finished" from "worked" will report health while doing nothing.
  **How to avoid:** a scheduled job's exit code must reflect whether the work happened; check outcomes, not runs.
- **Pattern:** credentials that expire on a cycle.
  **How to avoid:** expect the next expiry; the failure must reach a person the same day.

## Action Items

### Fixes (one-time)

| ID | Fix | Owner | Due | Status | Evidence |
|----|-----|-------|-----|--------|----------|
| 2026-09-18#1 | Log in again | RED | 2026-10-05 | done | Unattended run verified 10-05; first role ran normally 10-06 03:30 |
| 2026-09-18#2 | Launchers fail loudly (exit code + notification on failure or authentication message) | Claude | 2026-10-06 | done | Daily control passing since 10-06; stand-in tests 10-06 |
| 2026-09-18#3 | Register the system in the registry of long-running processes | Claude | 2026-10-06 | done | The registry check reads it as on schedule |

### Controls (ongoing proof)

| What it proves | Outcome or presence? |
|----------------|----------------------|
| Every role's last scheduled run succeeded (now meaningful, since the exit code is honest) | outcome |
| Every launcher still has the fail-loud code | presence |
| The first role's log is written on its daily schedule | outcome |
