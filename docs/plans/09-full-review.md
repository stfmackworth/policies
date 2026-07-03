# Plan 09 — Full Application Review (Round 3)

Fable review, 3 July 2026, against commit `9dff64f`. Whole-codebase read: `index.html` (1,717 lines), `data.js` (42 policies + config), `README.md`, plans 01–08 history. Every finding below was verified against the live code — line numbers are per `9dff64f`.

**How to use this document (junior agents):** each item has an ID, severity, location, evidence, fix, and a verify step. Work in the batches suggested at the end. Items marked **⚠ JAMES DECIDES** involve policy *content* or destructive choices — do not action them without his explicit answer; everything else is safe to implement directly. House rules apply: pure vanilla JS/HTML/CSS, zero dependencies, surgical patches, one logical unit per commit, commit messages explain the why.

**Severity:** P1 = real risk to the church or users, do first · P2 = misleading or fragile, do soon · P3 = polish, batch up.

---

## A. Content integrity — duplicates and contradictions

These are the most important findings in this review. The library currently presents contradictory governance documents as equally "active", which is a real-world compliance risk for a charity, not a cosmetic issue.

### R1 · P1 · p15 vs p42 — two live Finance policies that CONTRADICT each other ⚠ JAMES DECIDES

`data.js:231` (p15 Financial Controls Policy) and `data.js:564` (p42 Finance Policy) are both active, both PCC-required, and give **different answers to the same questions**:

| Question | p15 §3/§4 says | p42 §9/§11 says |
|---|---|---|
| Who approves spend? | OM up to £500 · OM + PCC officer £500–£2,500 · Finance Cttee £2,500–£10,000 · PCC over £10,000 | Treasurer + budget holder up to £200 · Standing Cttee £200–£999 · PCC £1,000+ |
| Cash banked within | 5 working days | "ideally within the same calendar month" |
| Cash counted by | two **unrelated** people | two members of PCC or Finance Committee |
| Finance Cttee chair | Operations Manager (p15 §1) | includes Treasurer, Church Leader + PCC nominees (p42 §2) |

A treasurer following p15 and a warden following p42 would each believe the other is breaching policy. p42 is v1.5, prepared by the Treasurer and **PCC-approved 18 June 2025** — almost certainly the authoritative document. p15 reads like an earlier best-practice draft.

**Fix (after James/PCC confirms):** archive p15 as superseded by p42 (see R7 for the mechanism). If any p15 clause is worth keeping (e.g. fraud-prevention §5, which p42 lacks), it should move *into* p42 via a PCC-noted amendment — a wording decision for James, not an agent.
**Verify:** library shows one active finance-controls document; archived p15 points to p42.

### R2 · P1 · p19 vs p38 — duplicate Complaints Policy ⚠ JAMES DECIDES

`data.js:278` (p19) and `data.js:507` (p38) are both titled "Complaints Policy" and both active. p38 is comprehensive (12 sections), names the designated person, and was **PCC-approved 25 March 2026**. p19 is a 3-section generic version with a *different* response timetable (written response in 28 days vs p38's staged process). A complainant reading p19 gets the wrong process.

**Fix:** archive p19, superseded by p38 (mechanism in R7). Recommended default — flag to James, then action.
**Verify:** searching "complaints" surfaces one active policy.

### R3 · P1 · p20 vs p39 — duplicate Conflict of Interest Policy ⚠ JAMES DECIDES

`data.js:289` (p20) and `data.js:527` (p39), same story: p39 is the CC29-referenced version **PCC-approved 25 March 2026**, owner PCC Chair. p20 is the earlier generic draft. Their declaration procedures differ in detail.

**Fix:** archive p20, superseded by p39.

### R4 · P1 · p17 vs p37 — two building-hire policies, each incomplete ⚠ JAMES DECIDES

`data.js:256` (p17 Buildings Hire Policy) and `data.js:494` (p37 Church Hiring Policy) both govern hiring the building — p37 is **not** about staff recruitment. They don't contradict so much as bisect the subject: p17 alone has permitted uses, hirer safeguarding conditions, insurance minimums, and cancellation terms; p37 alone has hire rates, the Standing Committee approval route, and ChurchSuite booking process (PCC-agreed 18 July 2024). A hirer needs both documents to know the rules.

**Fix:** merge into one policy — suggest folding p17's sections into p37 (the PCC-approved one) and archiving p17. The merged wording needs James's sign-off since both have PCC history. Note p37's review date (July 2025) is already overdue — merging is the natural moment for the PCC re-review James already owes it.

### R5 · P2 · p32 vs p42 §7 — Reserves stated twice (currently consistent, will drift)

`data.js:436` (p32 Reserves Policy, 4 detailed sections) and p42 §7 (3 bullet points) both state the 3-months-outgoings target. They agree today; two sources of truth will eventually diverge. **Fix:** shorten p42 §7 to the target figure plus "see Reserves Policy (32)" — one-line wording change, safe for a junior agent once James nods. Add mutual cross-links per R21.

### R6 · P2 · p16 vs p36 — IT Security vs IT Equipment & Acceptable Use overlap

`data.js:244` (p16) and `data.js:473` (p36) are legitimately distinct (systems security vs issued-device rules) but both legislate passwords and approved-systems/data-storage. Keep both; de-duplicate by making p36 reference p16's password standard instead of restating it, and add cross-links (R21). Low urgency. (p01 vs p41 Safeguarding is a *legitimate* pair — operational policy vs the CofE-required PCC statement — needs only cross-links, no merge.)

### R7 · P2 · Build the "archived / superseded" mechanism the above items need

The clean way to retire p15/p19/p20/p17 is **not** deleting them from `data.js` (breaks deep links, leaves unexplained numbering gaps, loses history). Instead:

1. The `status` field already exists on every policy and in the editor (`index.html:508`) but is **never rendered or filtered anywhere** — `badge-draft`/`badge-review` CSS (`index.html:155–156`) is dead code. Implement it: library and dashboard counts exclude `status:'archived'` by default; a small "Show archived" toggle in the library filter bar includes them, greyed, with an "Archived" badge.
2. Add an optional `supersededBy:"p38"` field. When an archived policy is viewed (e.g. via an old deep link), show a banner: *"This policy was superseded by Policy 38 — Complaints Policy"* linking to the successor.
3. Then flip p19/p20 (and p15/p17 after James confirms) to `status:'archived'` with `supersededBy`.

This is the single highest-value code change in the review because R1–R4 all depend on it.
**Verify:** `?policy=p19` still resolves, shows the banner, links to p38; library defaults to 38-ish active policies; badge-total counts active only.

---

## B. Bugs

### R8 · P2 · Stored XSS via unescaped policy `id` in onclick attributes

`index.html:1101` (library cards), `1030` (dashboard recent), `1437` and `1448` (compliance lists) all interpolate `p.id` raw into `onclick="viewPolicy('${p.id}')"`. Built-in and editor-created ids are safe, but **`importData` (line 1590) validates only `typeof p.id==='string'`** — an import file with `id: "x');alert(document.cookie);('"` executes on click, and a double-quote in the id breaks out of the attribute entirely. Same class of issue as the proto-pollution guard fixed in Plan 06.
**Fix:** in `importData`, reject customs whose id fails `/^[A-Za-z0-9_-]{1,64}$/` (count them in the "skipped" tally); belt-and-braces, wrap ids in `esc()` at the four render sites.
**Verify:** import a crafted file → entry skipped and reported; hand-inject one into localStorage → renders inert.

### R9 · P2 · Compliance "All Policies by Status" shows every policy green

`index.html:1448`: every row is hardcoded `compliance-item dot-green` / `compliance-dot green` regardless of review status — an overdue policy (p37, p42) shows a green dot on the very page meant to flag it. The list is literally titled "by Status".
**Fix:** derive the dot from `parseReviewDate` using the same thresholds as `generateReviewPack()`'s `pStatus()` (red = overdue, amber = due <3mo, green = current); factor that helper out rather than duplicating it a third time.
**Verify:** p37 shows red today.

### R10 · P2 · Compliance "PCC Adoption Required" implies 20+ policies await adoption

`index.html:1435–1445` lists **every** `pccRequired` policy under amber "PCC Adoption Required" — including p38–p42, whose own text records their PCC adoption dates. A churchwarden reading this page concludes the PCC has a mountain of outstanding adoptions; actually most are done.
**Fix:** the `lastReviewed`/`reviewedBy` fields shipped in Plan 08 are the tool. (a) Mechanical data pass: populate `lastReviewed`/`reviewedBy` in `data.js` for policies whose sections already state an adoption date (p37 "18th July 2024", p38/p39/p40 "25th March 2026", p41 "20th May 2026", p42 "18th June 2025") — dates are in the text, no invention needed. (b) Split the list: "Adopted" (has `reviewedBy` containing PCC — green, show date) vs "Awaiting adoption / re-adoption" (amber). Overdue-for-review PCC policies belong in amber even if once adopted.
**Verify:** p38 listed green with "Adopted 25 March 2026"; p15/p19/p20 (pre-archive) amber.

### R11 · P3 · Library grid ignores policy number order

`renderLibrary()` (`index.html:1068`) iterates raw `data.js` array order; p35 is physically last in the file (`data.js:589`), so the grid runs …41, 42, 35. Compliance sorts by number; the library should too.
**Fix:** sort `filtered` by `Number(p.number||99)` before rendering (stable for customs).

### R12 · P3 · "Recent Policies" isn't recent

`index.html:1029`: `policies.slice(-5)` = last five array entries — currently p39–p42 + p35, none of which is recent activity. **Fix:** prefer customs sorted by real recency (see R14 for storing a sortable timestamp), topped up with the highest-numbered built-ins; or rename the card "Highest-numbered policies" — cheap honesty if the data isn't there.

### R13 · P3 · Review-year typo becomes 1927

`getReviewDate()` (`index.html:1248`) trusts the year input; typing "27" produces `"April 27"`, which `parseReviewDate` turns into the year **1927** (2-digit years map to 1900+) — the policy silently shows overdue. The input's `min=2024/max=2099` doesn't constrain typed values.
**Fix:** clamp in `getReviewDate()`: if `y<100`, add 2000; if outside 2024–2099, fall back to `defaultReviewYear()` and show the existing `ed-review-hint`.

### R14 · P3 · `lastModified` is write-only, and stored in an unsortable format

Set on every save (`index.html:1376`, `1413`) as `"July 2026"`, displayed nowhere, and month-name strings can't sort (blocks R12). **Fix:** store ISO (`new Date().toISOString().slice(0,10)`), render human-readable in the policy-view meta grid for custom policies, and format existing values leniently.

### R15 · P3 · `driveFolder` config key is dead — URL hardcoded

`data.js:41` defines `driveFolder`; `index.html:690` hardcodes the same URL as `DRIVE_FOLDER_URL` and uses that everywhere. Changing the config does nothing. **Fix:** replace the constant with a getter reading `config.driveFolder` (config isn't populated until `init()`, so resolve at call time, falling back to the current URL).

### R16 · P3 · People page never flags a missing DBS

`renderPeople()` (`index.html:1503`) renders the green DBS chip when `dbs` is truthy and **nothing** otherwise — the `.person-dbs.missing` CSS (`index.html:258`) is unreachable. A staff row with no DBS looks identical to "not applicable". **Fix:** for `staff` entries only, render the red "DBS missing/expired" chip when `dbs` is falsy.

### R17 · P3 · Validation dialogs offer only a "Cancel" button

`openModal('Cannot Save',…,[])` (`index.html:1360`, `1362`, `1395`, `1397` and the import-result modals) renders a footer whose only button reads "Cancel" — confusing when the dialog is informational. Also `deletePolicy` (`index.html:1427`) uses a bare `alert()`, inconsistent with the modal pattern. **Fix:** when `actions` is empty, label the lone button "OK"; replace the `alert()` with `openModal`.

### R18 · P3 · Two stale statements in data.js

`data.js:40` — comment says the password hash is "update via Settings": there is no Settings UI for it, and `init()` deliberately force-overrides the hash from data.js anyway. Point the comment at README § "Changing the admin password". `data.js:45` — header says "35 POLICIES"; there are 42.

### R19 · P3 · Policy view hardcodes church name in two spots

`index.html:1164` ("Applies to all volunteers and staff at St Francis Mackworth") and `1171` (PSO fallback `'Joanne Baillie'`) bypass config that's respected elsewhere in the same function. Use `config.church?.name` / drop the personal-name fallback to a role label.

---

## C. Workflow & UX (end-user focus)

### R20 · P2 · No unsaved-changes guard in the editor

Nothing warns before discarding edits: Cancel (`index.html:582`), any sidebar click, browser Back (admin pages don't push history, so Back leaves the editor instantly), or Escape-driven flows all silently lose work. James drafting a 12-section policy and mis-clicking loses everything.
**Fix:** track a dirty flag (set by the existing `oninput` handlers + section add/move/remove; cleared on save/open); intercept `showPage()` away from the editor while dirty with a confirm modal; add a `beforeunload` handler while dirty. Keep it simple — no autosave engine.

### R21 · P2 · "See also" cross-references between policies

The dedup work (R5, R6, p01/p41) and plain usability both want it: optional `related:["p03","p16"]` on policies, rendered as clickable chips under the policy header. Volunteers reading Lone Working should see Pastoral Care and Risk Management one tap away. Small build, big navigational win, and data.js entries can be populated mechanically for the obvious pairs (Safeguarding cluster, Finance cluster, IT pair, hire pair).

### R22 · P2 · Nothing tells a new volunteer where to start

The dashboard is a compliance cockpit — useful to James, meaningless to a new kids-ministry volunteer, who actually needs 4 documents (Code of Conduct p02, Safeguarding p01/p41, Lone Working p03, Expenses p22). **Fix:** a reader-visible "New here? Start with these" card on the dashboard (a hardcoded curated list is fine and matches the curated-suggestions philosophy), or lightweight audience collections ("I'm a volunteer / I run a ministry / I'm on the PCC"). This is the single best improvement for the 95% of users who aren't admins.

### R23 · P3 · Search can't find a policy by number

`renderLibrary` matches title/tagline/category/body — typing "36" (how staff actually refer to policies) finds nothing unless "36" appears in body text. Add `p.number` to the `inMeta` check. One line.

### R24 · P3 · Saved-as-PDF files get the generic filename

Browser print-to-PDF names the file from `document.title` ("STF Policy Hub"). In `downloadPDF()`/print flow, set `document.title` to `` `${p.number} ${p.title} — STF` `` before `window.print()` and restore after — every exported policy self-names. Tiny, delightful.

### R25 · P3 · Admin session evaporates on every reload

`role` is a plain variable; any refresh logs James out. Deliberate simplicity, but consider `sessionStorage` persistence (per-tab, gone on close — still safe on shared church machines). Optional; James's call on the trade-off.

### R26 · P3 · Print the whole handbook

`generateReviewPack()` proves the pattern (hidden window + print CSS). A "Print full handbook" variant — every active policy, full text, cover page, category dividers — is what a PCC pack, an archdeacon's visitation, or an insurance audit actually wants. Medium effort, reuses `pStatus`/print CSS.

### R27 · P3 · Accessibility gaps (batch)

- `#lib-search` (`index.html:475`) has placeholder but no label → add `aria-label="Search policies"`.
- PCC toggle (`index.html:555`) is a `<button>` with no state semantics → `role="switch"` + `aria-checked`, updated on toggle.
- Editor section heading/body inputs are placeholder-labelled only → `aria-label` with section number.
- Emerging-topics "Use this template" buttons are fine; modal focus management is now good (Plan 06) — no further action there.

### R28 · P3 · Self-host the two Google Fonts

`index.html:7–8` is the app's only external dependency. Google Fonts CDN leaks visitor IPs to Google — awkward for a church whose own Data Protection policy (p08) promises minimal processing, and German case law has found exactly this a GDPR breach. Download the two families as woff2, serve from a `fonts/` folder, swap the `<link>` for `@font-face`. Bonus: works offline in the vestry, and honours the zero-external-dependency spirit fully.

---

## D. Housekeeping

### R29 · P3 · README drift

`README.md:16`, `28`, `41` still describe `equipment-issue-form.html` as the linked printable form — since `9dff64f` the IT policy links the **Google Doc** instead. Update the three mentions. While in there: line 8 "Forty-two policies" will need revising after the archive work (say "42 shipped, N active" or similar).

### R30 · P3 · Orphaned `equipment-issue-form.html` ⚠ JAMES DECIDES

Unlinked as of `9dff64f` but still deployed. Options: delete (tidy, Google Doc is now the master) or keep as an offline/print fallback and re-link it as a secondary option on p36. Either is fine — but decide, don't drift.

### R31 · P3 · Commit messages carry a UTF-8 BOM

Recent commit subjects begin with an invisible `U+FEFF` (see `git log --oneline` — "﻿Point IT Equipment…"), because PowerShell 5.1's `Set-Content -Encoding utf8` writes a BOM and `git commit -F` keeps it. Harmless but scruffy in tooling. Fix the recipe: write the temp message with `[IO.File]::WriteAllText($path, $msg)` (BOM-less) instead of `Set-Content`. Record the corrected recipe in `docs/plans/00-OVERVIEW.md` working-style notes.

### R32 · P3 · Local repo clutter

`node_modules/`, `package.json`, `package-lock.json`, `make-form.js`, `IT-Equipment-Issue-Form.docx` sit in the working tree from the DOCX generation. All correctly gitignored — no repo action needed — but the folder can be deleted locally once James confirms the Google Doc is final.

---

## E. Suggested execution batches

| Batch | Items | Notes |
|---|---|---|
| 1. Ask James | R1–R4 decisions, R30 | One conversation: confirm p42/p38/p39 as authoritative, agree the p17→p37 merge, decide the orphan form. Nothing in batch 3 content lands before this. |
| 2. Archive mechanism | R7, R11 | Code only, no content decisions needed. Unblocks everything in section A. |
| 3. Content resolution | R1–R6 data edits, R10a (adoption dates), R21 (`related` data) | After batches 1–2. Mechanical `data.js` edits — statuses, supersededBy, lastReviewed/reviewedBy, related, cross-ref one-liners (R5/R6 wording needs James's nod). |
| 4. Bug sweep | R8, R9, R10b, R13–R19 | Independent small fixes; 2–3 commits. |
| 5. UX round | R20, R22, R23, R24, R27 | The end-user batch. R22 curated list contents need a quick 👍 from James. |
| 6. Optional / later | R12, R25, R26, R28, R29, R31, R32 | Nice-to-haves; README (R29) should ride with whichever batch changes behaviour it describes. |

**Verification (no test suite):** browser-check per item as noted, plus the standing flows — deep links (`?policy=p19` post-archive!), back/forward, admin login, editor save/copy/delete, import/export (include an R8 hostile file), mobile drawer at 375px, keyboard-only pass, print flows.
