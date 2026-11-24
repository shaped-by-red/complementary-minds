# COE: The relay watcher was blind for 8.5 days and reported success every minute

> **Date of incident:** 2026-08-12 14:35 to 2026-08-21 03:44
> **Date of writeup:** 2026-10-12 (requested by RED in the 2026-H1 COE review)
> **Author(s):** CLI Claude (Opus 5.5) + RED
> **Status:** Reviewed. Fixes done with evidence; closes when the daily relay check has passed for 30 days (from 2026-10-12).
> **Source:** [internal record - withheld]

*A Correction of Error (COE) is a blameless incident write-up: timeline, five whys, contributing factors, fix, lessons, action items. This public copy has private specifics removed; each removal is marked.*

## Summary

The relay is a shared mailbox between two of RED's computers, through which two Claude Minds (one on each machine) leave each other notes. A scheduled job checks it every minute for new notes and alerts RED. The job reached the second machine by a fixed network address. When that machine's address changed, every run logged "relay unavailable, skipped" and exited as a success, for 8.5 days: about 12,300 runs. Nothing was missed (no notes arrived in that window), but the detection layer was dead and said so only in a log no one reads. The job now reaches the machine by name, alerts after an hour of misses, and a daily check proves the whole relay loop works.

## Timeline

- **2026-07-28 22:29**: last note surfaced before the outage. (A quiet stretch followed: no notes, nothing logged, which is normal.)
- **2026-08-12 14:35**: first "relay unavailable" line. From here every run fails to reach the relay.
- **2026-08-13 to 08-20**: 1,440 "unavailable" lines a day (one a minute). The scheduler recorded success every time.
- **2026-08-21**: in an interactive session the fallback path to the relay was reconnected, which healed it. The last "unavailable" line is 03:44. (How the problem was noticed was not recorded.)
- **2026-08-25**: 13 more misses (a brief blip).
- **2026-10-06**: the fixed address replaced with the machine's name; escalation added (one alert after 60 consecutive misses, one on recovery). The second machine was offline for about twelve hours that day (738 misses), and the new escalation was in place by the end of it.
- **2026-10-12**: a daily outcome check of the whole relay loop added.

(All times from the watcher's own log, counted per day. *Correction made during the writeup:* the process registry and the 2026-H1 review draft both said the outage ran "2026-07-28 to 08-21, 14,852 runs". The log shows it began 08-12; 07-28 was just the last note before a normal quiet period. Fixed in the registry the same day: a small instance of pattern B, "stories treated as facts".)

## Impact / Blast Radius

- **Users affected:** none external. Cross-machine coordination between the two Claude Minds.
- **Duration:** 8.5 days blind.
- **Data impact:** none. No notes arrived in the window, so none were missed. Luck, not design.

## Root Cause (5 Whys)

1. **Why couldn't the watcher see the relay?** The second machine's network address moved, and the job used a fixed address.
2. **Why did the fallback path fail too?** It pointed at the same stale address.
3. **Why did nobody know?** The job treated "unreachable" as "not an error" and exited as a success, every minute.
4. **Why was that the design?** Short outages are normal (the second machine sleeps), and alerting every minute would be noise.
5. **Root:** no difference between a blip and an outage. **A check that grades zero items should fail, not pass.**

## Contributing Factors

- [network configuration details - withheld].
- The log grew by 1,440 identical lines a day; no one reads a log like that.

## Detection

- **Found by:** an interactive session, by chance.
- **Sooner?** Yes. "Unreachable for over an hour" was always detectable; nothing was asking.

## The Fix

- **Code:** the watcher (and a companion relay tool) now address the second machine by name, not by number. The watcher keeps a running count of consecutive misses: one desktop alert after about an hour of misses, one on recovery.
- **Monitoring:** a daily check of the whole relay loop (2026-10-12): the relay is reachable, nothing is stuck in our outbox, every inbound note has a receipt, nothing has gone unanswered for over 3 days.

## Lessons Learned

- **Pattern:** silent success (pattern A in the 2026-H1 review), and hidden environment assumptions (pattern C: a hardcoded address).
  **How to avoid:** address machines by name; a check that sees nothing must say so, and escalate when "nothing" lasts.

## Action Items

### Fixes (one-time)

| ID | Fix | Owner | Due | Status | Evidence |
|----|-----|-------|-----|--------|----------|
| 2026-08-12#1 | Machine name instead of fixed address in the watcher and the companion tool | Claude | 2026-10-06 | done | [internal file - withheld] |
| 2026-08-12#2 | Escalate after 60 consecutive misses; alert on recovery | Claude | 2026-10-06 | done | [internal file - withheld] |
| 2026-08-12#3 | Daily outcome check of the whole relay loop | Claude | 2026-10-12 | done | Check passing 10-12; a planted-fault test caught 3 of 3 faults |

### Controls (ongoing proof)

| What it proves | Outcome or presence? |
|----------------|----------------------|
| The relay is reachable; messages flow both ways; none stuck or unanswered | outcome |
| The watcher's heartbeat is written every minute | outcome |
