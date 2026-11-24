# COE: Century Dispatch image / click-through mismatch

> **Date of incident:** 2026-04-07 (visible symptom). The drift began 2026-03-18.
> **Date of writeup:** 2026-04-07
> **Author(s):** CLI Claude (Opus 4.6, 1M context) + RED
> **Status:** Closed (all critical action items shipped)

## Summary

The April 7, 2026 Century Dispatch email (sent by the Cloudflare Worker cron at ~05:00 EST) showed one historical newspaper as the embedded thumbnail, but the click-through link led to a different newspaper for the same date. RED noticed when he clicked through to verify a thumbnail. Investigation revealed the bug was systemic across **every** dispatch sent since the bulk LOC fetch ran in mid-March — RED just hadn't clicked through previous emails. The root cause was a hardcoded URL in the Cloudflare Worker pointing at a stranded R2 namespace (`scans/`) that drifted out of sync with the rest of the dispatch artifact bundle. Fixed by making the worker read all asset URLs from the dispatch JSON instead of constructing them, with an `assets.scan` field added to the JSON schema and a fallback to `assets.thumbnail` for backwards compatibility. Total subscribers affected: a small friends-and-family list. Total broken emails: ~20 days' worth. Worker fix deployed and verified before the next morning's cron.

## Timeline

- **2026-03-13 ~04:00 UTC** — Initial 1926 site-data uploaded to R2 by hand. Dispatches, thumbnails, and murals all uploaded together; internally consistent at this moment.
- **2026-03-17 ~14:00** — A bulk fetch script [file name withheld] started by previous CLI Claude session. Fetches LOC newspaper data for years 1830-1930 to populate the historical archive (free work, `--no-summarize` mode).
- **2026-03-17 23:52** — Bulk fetch downloads new newspapers for 1926-04-07 (and other dates), overwriting the local scan file for that date [path withheld] with content from a different newspaper than the originally-published one. Local pipeline state silently diverges from published R2 state.
- **2026-03-18 07:01** — R2's `scans/` namespace re-uploaded by hand. The new (mismatched) scans go to R2. The matching `dispatch/`, `thumbnails/`, and `murals/` namespaces are NOT re-uploaded. Drift is now live in production.
- **2026-03-18 → 2026-03-30** — Civil War backfills and a second general backfill run. More dates get the same treatment.
- **2026-03-23 13:10** — Bulk fetch completes (37 years processed). Previous Claude writes "bulk fetch is running, don't kill it" in State of Mind earlier in the session, but doesn't update it after completion.
- **2026-03-23 → 2026-04-06** — Daily Cloudflare Worker cron sends dispatches. Each email's body shows the thumbnail (old paper, correct), each click-through goes to scans/ (new paper, wrong). Nobody notices because nobody clicks through.
- **2026-04-07 ~05:00 EST** — Worker cron sends April 7 dispatch. Same bug pattern.
- **2026-04-07 ~11:00** — RED clicks through. Notices the mismatch. Opens this session.
- **2026-04-07 ~11:15-12:30** — Investigation. Initially built on a false premise (that the bulk fetch was still running, per stale State of Mind note). Process registry findings reframe the picture: bulk fetch ended Mar 23, Civil War + general backfills also done.
- **2026-04-07 ~12:30** — Architecture mapped. Two parallel email systems identified (Worker + a legacy daily script [file name withheld]). Worker confirmed as production sender. Discovery: there's no automated `site-data → R2` sync; R2 is hand-loaded.
- **2026-04-07 ~12:35** — Three edits made [file names withheld]: the export script adds an `assets.scan` field, the worker reads the scan URL from `dispatch.assets.scan` (with fallback), the batch fetch script gets an immutable-dispatch guard.
- **2026-04-07 ~12:40** — Worker deploy via the deploy script [file name withheld] halts on a benign GitHub file-size warning (PowerShell ErrorActionPreference quirk). Bypassed by invoking `wrangler deploy` directly.
- **2026-04-07 ~12:45** — Worker version [version ID withheld] deployed. Health check OK. Simulated URL resolution confirms email body and click-through now reference the same file.
- **2026-04-07 ~13:00** — Two test emails sent to RED via test-send [endpoint withheld] for dates 1926-04-07 and 1926-04-08. RED visually confirms the click-through matches the email body in both. **Bug closed.**
- **2026-04-07 ~13:00 → 14:30** — Belt-and-suspenders work: the deploy script's PowerShell fix, a process registry with a status checker [internal files withheld], two new guardrails in the workspace instructions, COE archive scaffolding, this writeup.

## Impact / Blast Radius

- **Users affected:** Small friends-and-family subscriber list [count withheld]
- **Duration:** ~20 days of broken emails (Mar 18 — Apr 7)
- **User-visible symptom:** Email body showed Newspaper A. Clicking the "View full size" link or the thumbnail itself loaded Newspaper B for the same historical date. Inconsistent with the text summary in the email body, which described Newspaper A.
- **Data impact:** No data lost. R2's `scans/` namespace contains "wrong" content from a publishing perspective but is internally valid. The dispatch JSONs and thumbnails in R2 are correct and consistent with each other. After the worker fix, the stranded `scans/` namespace becomes irrelevant — the worker no longer references it.
- **Reputational / business impact:** Minimal. Subscriber list is friends RED knows personally; the issue is forgivable at this stage and was caught before the audience grew.

## Root Cause (5 Whys)

1. **Why did the email body and the click-through show different newspapers?**
   Because they were sourced from two different files. The email body image came from `dispatch.assets.thumbnail` (resolves to `thumbnails/scan-{date}.jpg`), and the click-through URL was hardcoded in the worker to `scans/scan-{date}.jpg`. Two files, two upload histories.

2. **Why did those two files have different content if they're keyed by the same date?**
   Because R2 was loaded by hand, in batches, at different times. The `thumbnails/` namespace was uploaded once on Mar 13. The `scans/` namespace was uploaded again on Mar 18 — *after* the bulk fetch had overwritten the local source files for those dates with a different newspaper. The two uploads were not atomic.

3. **Why did the bulk fetch overwrite local source files for already-published dates?**
   Because the batch fetch script had no concept of "this date is published, leave it alone." Its `--resume` flag only checked DB page count, not "is there a published dispatch JSON downstream?" The pipeline-to-publishing flow was implicit one-way; once published, the upstream pipeline kept churning the same date-keyed paths.

4. **Why was the upstream/downstream coupling not explicit?**
   Because the pipeline [internal project name withheld] was built first as a research/exploration tool — long before the publishing layer (the website's site data) existed. The publishing layer was added later for the email product. Nobody updated the pipeline to know about the publishing layer's contract. Two systems were welded together by shared filesystem paths, but neither system enforced the invariant "published artifacts are immutable."

5. **Why did the Worker hardcode `scans/scan-{date}.jpg` when nothing in the current pipeline writes to that path?**
   Because the Worker was written when the data layout was different — probably during an earlier architecture where `scans/` was an actively-maintained namespace. That architecture was deprecated, but the Worker code was never updated. The hardcoded path survived because **there was no test that compared the URLs the worker constructs against the URLs the pipeline actually publishes.** The contract was implicit.

**The actual root:** the worker constructed URLs *imperatively* instead of reading them from the dispatch JSON. That meant the worker held a *claim* about R2's layout that nothing was verifying. As soon as the layout drifted, the claim became false, and there was no mechanism to surface the drift.

## Contributing Factors

- **State of Mind staleness.** A previous Mind wrote "bulk fetch is running, don't kill it" in Claude's State of Mind journal [internal file - withheld] during an active fetch. The next Mind (this one) read it weeks later as if still true, and initially built the entire investigation on the wrong premise. The bulk fetch had finished on Mar 23. Time-bound facts in narrative documents decay.
- **Undocumented pipeline → R2 sync.** The mechanism for getting `site-data/` content into R2 was completely undocumented in the pipeline's architecture document [internal file - withheld] and elsewhere. It turns out to be hand-loaded via `wrangler r2 object put`, but there is no script, no schedule, and no record of when uploads happen.
- **Stranded local export directory for scans [path withheld].** A holdover from an earlier architecture. The directory exists on disk and gets opportunistically synced to R2's `scans/` namespace, but no current code path writes to it intentionally. It's a confusion-generator with no current purpose.
- **Deploy script PowerShell error handling.** The deploy script halted on a benign GitHub file-size warning during the urgent worker deploy. Cost ~5 minutes of confusion before being bypassed. PowerShell 7.4+'s `$PSNativeCommandUseErrorActionPreference` defaults to `$true`, which escalates native command stderr to terminating errors under `$ErrorActionPreference="Stop"`.
- **56MB `.wav` file in the website git repo** [path withheld]. This is the file that triggered the deploy script's halt. It's in an unused-assets directory and shouldn't be in git at all.
- **No end-to-end test of email rendering.** The Worker has a test-send endpoint [path withheld] that exists for exactly this purpose, but it was never used as a smoke check after deploys.
- **Two parallel email systems** (Worker + a legacy daily script [file name withheld]) made the architectural picture harder to read. Initially I couldn't tell which one was actually sending; only after grepping `wrangler.toml` and confirming `schtasks` showed no scheduled task did I realize the Worker was the production path.

## Detection

- **How was it found?** RED clicked through an email to verify a thumbnail. He noticed the linked image was a different newspaper than the one shown in the email body.
- **Could it have been found sooner?** Yes. Any of the following would have caught it:
  - An automated test-send [endpoint withheld] after every deploy (compares rendered HTML against expected URLs)
  - A daily sanity check that fetches a dispatch JSON and asserts every URL in `assets.*` returns 200 from R2
  - A visual diff between the email body image and the click-through target during local development
  - A "before publishing, verify all asset URLs match the dispatch JSON contract" pre-flight in the pipeline
- **Earliest signal that would have caught it:** the moment the Worker started constructing a URL that the pipeline didn't publish, an integration test would have failed. There was no integration test.

## The Fix

**Code changes** [file names, line numbers and one local path withheld]:
- The worker, in both of its email builders — the full-scan URL now reads from `dispatch.assets?.scan || dispatch.assets?.thumbnail` instead of being constructed from a base URL and the dispatch date. Backwards-compat fallback handles old dispatch JSONs in R2 that don't have `assets.scan`.
- The pipeline's export script — adds a `scan` entry to the assets dict, pointing at the thumbnail file. The thumbnail file is already a 1000px full-quality web scan per the docstring, so it doubles as the click-through target.
- The pipeline's batch fetch script — adds the **immutable-dispatch guard**: skips dates whose dispatch JSON exists in the website's published site data, unless `--force-republish` is passed. Verified working with a single-date test on `1926-04-08`.
- The website's deploy script — git push now uses `2>&1 | Out-Null` and checks `$LASTEXITCODE`. Non-zero exits print a yellow warning but no longer halt the script. Local commit is documented as the actual checkpoint.

**New infrastructure** [internal file names withheld]:
- A process registry at the workspace root — seeded with all known long-running processes (8 entries).
- A status checker — reads the registry, checks each entry's log file mtime, reports `ACTIVE | IDLE | STALE | OK | EXTERNAL`. Verified working.
- A COE archive at the workspace root with a README, a template, and this very document.
- Two new guardrails in the workspace instructions: check the process registry, and write a COE when something breaks.

**Worker deploy:** Version [version ID withheld] shipped 2026-04-07 ~12:45. Verified via a health check, simulated URL resolution, and two real test-send calls [endpoint withheld] (1926-04-07 and 1926-04-08, both Full edition) which RED confirmed visually in his inbox.

## Lessons Learned

1. **Asset bundles are atomic, even when their files share only a date.**
   When multiple files together describe one logical artifact (dispatch JSON + thumbnail + mural + scan), they must be uploaded together or pinned to a contract. Date-keyed naming implies coupling without enforcing it. **Pattern:** if you find yourself writing `scans/scan-${date}.jpg` and `thumbnails/scan-${date}.jpg` in different places, ask: what guarantees these refer to the same source? If the answer is "convention," you have a bug waiting to happen.

2. **Hardcoded URLs are claims about state.**
   If your code constructs a URL like `${BASE}/foo/${id}.jpg`, you are *claiming* that file exists at that path. Until something verifies the claim, it's just a wish. The contract should live in **data** (the dispatch JSON), not in **code** (the worker). The worker should *read*, not *construct*. This generalizes to any system where one component publishes files and another consumes them.

3. **Time-bound facts in shared narrative documents decay.**
   State of Mind notes, activity log entries, and any "as-of-this-moment" assertion will become wrong as soon as the moment passes. Either tag them with an explicit `as of YYYY-MM-DD`, or move them to a mechanically-verifiable place. The process registry and its status checker [internal files withheld] are the antidote for the specific case of long-running processes — they make "is X running" answerable in 2 seconds against actual log mtimes, instead of trusting a story.

4. **Scripts that worked in their original universe break when the universe changes.**
   The bulk fetch was correct when no dates were published. It became wrong as soon as some dates were published. The Worker was correct when `scans/` was an actively-maintained namespace. It became wrong as soon as that namespace was deprecated. Both scripts shipped with an implicit assumption about their environment that nothing checked. **Pattern:** scripts should be either *idempotent and self-checking*, or they should run in CI on every commit so drift gets caught immediately. Standalone scripts that operate on shared state are landmines.

5. **PowerShell native command stderr ≠ PowerShell error.**
   Under `$ErrorActionPreference="Stop"` in PowerShell 7.4+, any stderr output from a native command (git, wrangler, npm, etc.) escalates to a terminating error before any redirection takes effect. This is governed by `$PSNativeCommandUseErrorActionPreference`, which defaults to `$true`. The Unix-correct check is the **exit code**, not the **stderr stream**. Use `2>&1 | Out-Null; if ($LASTEXITCODE -ne 0) { ... }` instead of `2>$null`.

6. **The friends-and-family phase exists exactly for catching this class of bug.**
   This bug shipped to a tiny subscriber list, was caught quickly, and cost almost nothing in trust. The same bug at 10K subscribers would have been an unrecoverable trust event. **Don't apologize for catching things early — celebrate it.** The point of the early-stage product is to surface exactly these kinds of mechanism bugs so they don't survive into scale.

> A pattern across lessons 1, 2, and 4: **systems that work via convention are landmines; systems that work via contract are robust.** The fix for all three is the same shape — make the implicit explicit, in data, and verify it.

**Load-bearing lessons to add to the cross-project lessons file [internal file - withheld]** (action item #10): #1, #2, #3, and #4 above.

## Action Items

| # | Action | Owner | Status |
|---|---|---|---|
| 1 | Worker reads scan URL from `dispatch.assets.scan` (with fallback to thumbnail) | CLI Claude | done |
| 2 | The export script emits `assets.scan` in dispatch JSON | CLI Claude | done |
| 3 | The batch fetch script's immutable-dispatch guard with `--force-republish` override | CLI Claude | done |
| 4 | The deploy script's git push uses exit-code check, not stderr suppression | CLI Claude | done |
| 5 | Process registry + status checker [internal files withheld] | CLI Claude | done |
| 6 | Two guardrails in the workspace instructions (process registry, COE archive) | CLI Claude | done |
| 7 | COE archive scaffolded with README + template + biannual review cadence | CLI Claude | done |
| 8 | This COE writeup as the inaugural archive entry | CLI Claude | done (this) |
| 9 | Move 56MB `.wav` out of git into LFS or out of repo | RED | pending |
| 10 | Add load-bearing lessons (1-4 above) to the cross-project lessons file | next session | pending |
| 11 | Document pipeline → R2 sync mechanism in the pipeline's architecture document | RED or future Mind | pending |
| 12 | Add automated test-send [endpoint withheld] smoke check to worker deploy or daily cron | future | pending |
| 13 | Delete stranded `scans/` namespace in R2 (cleanup, not urgent) | RED, when convenient | pending |
| 14 | First biannual COE review (end of June 2026) | RED + a Mind | scheduled |

## Appendix

### Key files touched
[internal file list withheld: the worker, two pipeline scripts, the deploy script, and the new registry, status checker, guardrail and COE archive files]

### Worker deploy trail
- Pre-fix version: unknown (not captured before deploy)
- Post-fix version: [version ID withheld]
- Deployed at: 2026-04-07 ~12:45 EST
- Cron schedule: `0 10 * * *` (10:00 UTC daily, ~5/6 AM EST depending on DST)
- URL: [worker address withheld]

### Test verification
Two test-send calls [endpoint, request bodies and message IDs withheld] confirmed the fix end-to-end: Full edition, dates 1926-04-07 and 1926-04-08, sent to RED.

RED visually confirmed both: email body and click-through reference the same newspaper.

### What did NOT need to be done

- **R2 was not re-uploaded.** The worker fallback (`assets.scan || assets.thumbnail`) makes every existing dispatch JSON in R2 self-heal at send time. The "broken dispatches in R2" were never actually broken — only the worker's interpretation of them was broken.
- **Already-sent emails were not retroactively fixed.** RED explicitly accepted the trade-off: the few subscribers who received broken emails will see the inconsistency permanently in their inboxes, but the archive going forward is correct. *"Its ok. We had a glitch doing something fun, new (old?) and creative."*
- **The stranded local export directory for scans was not deleted.** It's harmless now and cleanup can wait.
- **Opaque-hash filenames were not implemented.** This was an earlier proposal in the investigation, but turned out to be unnecessary once the simpler "make the JSON the contract" fix was identified. The hash-based namespacing remains a possible future improvement if we ever need to support multiple papers per date in R2.

### Notes from RED

> *"This is also a good learning opportunity. We should have for all projects we do a log of running/ongoing processes and their location."* (→ became the process registry)
>
> *"I'm not about blame. Blame serves no purpose. It's not constructive."* (→ saved as feedback memory; informs the entire COE framing)
>
> *"Its ok. We had a glitch doing something fun, new (old?) and creative. I'm ok with our first couple emails having this error, we don't need to retroactively try to fix emails."* (→ scope decision that simplified the repair phase enormously)
>
> *"COE is more descriptive and specific than after-action which is vague military terminology... There are some real gems from [employer withheld]'s mechanistic thinking. It's one of the reasons I really like it there."* (→ COE archive concept and biannual review cadence)
