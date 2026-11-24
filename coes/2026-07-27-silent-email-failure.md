# COE: Century Dispatch daily email silently stopped sending for 10 days

> **Date of incident:** 2026-07-18 (sending stopped) to 2026-07-27 (discovered)
> **Date of writeup:** 2026-07-27
> **Author(s):** CLI Claude + RED
> **Status:** Reviewed
> **Source:** [internal record - withheld]

*A Correction of Error (COE) is a blameless incident write-up: timeline, five whys, contributing factors, fix, lessons, action items. This public copy has private specifics removed; each removal is marked.*

## Summary

Century Dispatch's daily newsletter stopped reaching subscribers on 2026-07-18, and the failure went unnoticed for about 10 days. The scheduled job and the content pipeline were healthy the whole time; the break was at the email-delivery layer. The email provider rolled out a policy (around 2026-07-18) that rejects an *entire* broadcast with an error if the audience contains any contact whose address ends in `@example.com`. The audience still held leftover `@example.com` test contacts from spring testing, so every daily broadcast failed at the send step. The sending code treated the send error as "non-fatal", swallowed it, and advanced its own success counter regardless. It reported success while sending nothing. Fixed by removing the test contacts, pruning any `@example.com` contact automatically before every send, making the success counter advance only on a real send, and adding a self-alert email on failure.

## Timeline

- **2026-03 to 2026-05**: `@example.com` test contacts added to the subscriber audiences during feature testing; never removed.
- **~2026-07-18**: the email provider begins enforcing a policy that rejects any broadcast whose audience contains an `@example.com` address. Daily broadcasts start failing at the send step. Broadcasts are still *created* (as drafts), just never *sent*.
- **2026-07-18 to 2026-07-27**: the scheduled job fires daily, swallows the error, and advances its "last dispatched" date anyway. Drafts pile up; zero emails delivered. No alert.
- **2026-07-27**: RED returns [personal detail withheld], notices he has not received dispatches, and reports it.
- **2026-07-27**: diagnosis. The sender's own stats showed the last dispatch date as today (falsely "healthy"); the provider's dashboard showed every broadcast stuck in "Draft" since 07-18. A diagnostic that reproduced the real send call returned the exact error: the audience contained an address ending in `@example.com`.
- **2026-07-27**: today's issue sent by hand. Test contacts removed. Sender hardened and redeployed. The backlog was deliberately **not** backfilled: a 10-day gap is cleaner than 20 stale emails at once.

## Impact / Blast Radius

- **Users affected:** the full subscriber list ([subscriber count - withheld]; small, early-stage).
- **Duration:** about 10 days (2026-07-18 to 2026-07-27).
- **User-visible symptom:** no daily dispatch arrived.
- **Data impact:** none. No corruption. The 10 unsent issues exist as drafts and were intentionally left unsent (their "on this day 100 years ago" dates had passed).
- **Reputational / business impact:** low, given the small known list. But a [business detail - withheld] product silently not delivering is exactly the failure mode to prevent as the list grows.

## Root Cause (5 Whys)

1. **Why did subscribers stop getting the daily email?**
   Because every daily broadcast failed at the send step.

2. **Why did the send step fail?**
   Because the email provider rejected it: the audience contained `@example.com` contacts, which the provider now refuses to send to.

3. **Why were `@example.com` contacts in a production audience?**
   Because test contacts added during earlier testing were never removed, and the provider changed its policy (~07-18) to reject the whole broadcast over them rather than just bouncing them.

4. **Why did no one notice for 10 days?**
   Because the sender swallowed the send error as "non-fatal" and advanced its "last dispatched" date unconditionally, so its success signal (and its stats page) reported healthy while nothing sent.

5. **Why did the sender report success on total failure?**
   The actual root: a critical-path operation had **no invariant tying its "success" to real delivery**. The counter advanced on "the job fired", not on "the send succeeded", and there was no alert on failure. The unstated assumption was "if the code runs without throwing, it worked."

## Contributing Factors

- Test data (`@example.com`) left in a production audience: a latent landmine.
- A third-party policy change broke us with zero change on our side.
- Every provider error was logged as "non-fatal" with no aggregation or alerting, and the logs were only visible live, so a swallowed error left no durable trace.
- The stats "health" signal measured "the job fired", not "delivered": an actively misleading indicator.

## Detection

- **How was it found?** User report (RED noticed missing emails).
- **Could it have been found sooner?** Yes, about 10 days sooner.
- **What signal would have caught it earlier?** Either (a) a failure-triggered alert email, or (b) an external check comparing the last dispatch date to today's expected date. Both now exist or are recommended.

## The Fix

1. **Removed** the `@example.com` test contacts from both audiences (RED, in the provider's dashboard).
2. **Auto-prune:** the sender now deletes any `@example.com` contact from the audience before each send, so a future stray one cannot sink the whole broadcast.
3. **Truthful success signal:** the "last dispatched" date and the days-dispatched count now advance **only** when a broadcast actually sends. On total failure the sender records the failure date and time in durable storage.
4. **Self-alert:** on a failed daily send, the sender emails RED immediately through a one-to-one (transactional) send, which is not subject to the broadcast policy, so it would have fired on day 1 of this incident.

## Lessons Learned

- **Never let a process report success on silent failure.** A lying success signal is worse than no signal: it actively suppresses detection. Tie "success" to the real outcome (delivery), not to "the code ran."
- **"Non-fatal" swallowed errors on a critical path are dangerous.** If a step is critical, its failure must change a durable state and raise an alert, not just print to a log nobody reads.
- **Test data in production is a landmine.** `@example.com` sat harmless for months until a third-party policy change weaponized it.
- **Third-party policy changes are an outage class of their own.** Nothing on our side changed; a vendor's rule did. Alerting (not just change control on our own code) is the defense.
- **Verify, do not assert.** During diagnosis the first guess (a quota or paywall) was wrong and would have led to paying for a plan we did not need; the usage page contradicted it, and only reproducing the actual send call revealed the true error. Get the real error before prescribing the fix.

## Action Items

- [x] Remove `@example.com` contacts from both audiences.
- [x] Auto-prune `@example.com` before every send.
- [x] Success counter advances only on a real send; a failure is recorded durably.
- [x] Self-alert email to RED on a failed daily send.
- [ ] (Recommended) External heartbeat monitor that checks the sender's stats daily and alerts if the last dispatch date is behind today's expected date. It catches the rarer "the job or the sender is entirely down" case that the self-alert cannot.
- [ ] Audit other critical-path jobs for the same swallow-and-report-success pattern. *(Public-copy note, not in the original: this audit had not yet run when the same pattern struck again on 2026-09-18; see that COE.)*
- [ ] RED: [credential housekeeping item - withheld].
