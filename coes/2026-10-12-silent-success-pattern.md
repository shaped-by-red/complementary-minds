# Three times a green light meant nothing: the silent-success pattern

> **Source:** the 2026-H1 COE review (held 2026-10-12) and the three COEs it names. [internal record - withheld]

In a little under three months, three unrelated systems in RED's workspace failed the same way. Each one kept reporting success while doing nothing.

**[2026-07-27](2026-07-27-silent-email-failure.md): the newsletter that never sent.** Century Dispatch's daily email stopped reaching subscribers for about ten days. The email provider had begun rejecting any broadcast whose audience held an `@example.com` address, and old test contacts were still in the list. The sending code logged the rejection as "non-fatal" and advanced its "last dispatched" date anyway, so its own health page said all was well. RED found it when the emails stopped arriving.

**[2026-08-12](2026-08-12-relay-watcher-silent-outage.md): the watcher that could not see.** A job that checks a shared mailbox between two machines every minute lost its way there when one machine's network address changed. For 8.5 days, about 12,300 runs, it logged "unavailable, skipped" and exited as a success. No notes arrived in that window, so nothing was lost: in the COE's words, "Luck, not design."

**[2026-09-18](2026-09-18-silent-auth-failure.md): the roles that could not log in.** A set of scheduled Claude roles stopped working for 18 days when their stored login expired. Each launcher script ended with a command that always succeeds, so the scheduler recorded success on every run. It had happened twice before, briefly, and went unnoticed both times. A Claude Mind running its own audit found it.

## What the review said

When RED and Claude held the overdue half-year COE review on 2026-10-12, these three became pattern A, **silent success**: "a job says OK while doing nothing." The review called it "the most common pattern by far." It also named a pattern that runs underneath this one and others, **self-consistent wrong state**: "each wrong belief makes the next check agree with it." A counter that advances on failure makes the health page agree with it; an exit code of 0 makes the scheduler agree with it.

RED's decision was **both layers**. The floor: every scheduled job is listed in a registry with a way to verify it, and a daily check proves each one alive. Above it, the critical few (daily email, backups, the relay) get checks of their *outcome*. On its first day the floor found another job that had been down for 25 days, which no per-system check covered.

## What the three COEs add

The review came after all three incidents and names the pattern; the two points below come from the COEs themselves, not the review.

The July COE had listed an action item to audit every critical job for the same shape. That audit had not run when the September failure began.

And each COE reached the same lesson in different words: tie success to the real outcome (delivery, reachability, work done), not to "the code ran." A check that sees nothing must say so. A lying success signal is worse than no signal, because it suppresses detection.
