# Searching LinkedIn Recruiter and extracting candidates

This is the mechanical part of a run: finding or creating the role's project,
turning the signed-off brief into a LinkedIn Recruiter search, pulling real
candidate profiles out of the page, and writing qualified ones straight into
the project. It uses the Claude in Chrome tools and requires the recruiter to
be logged into their own LinkedIn Recruiter seat in the browser.

## Prerequisites

- The user is logged into LinkedIn Recruiter (`linkedin.com/talent/...`) in a
  Chrome tab connected to Claude in Chrome. If not, stop and ask them to log in
  — you cannot and must not enter their credentials.
- You have the signed-off brief.

## 0. Work like a human — pace yourself and never fight security

You are driving the recruiter's own LinkedIn Recruiter seat. Machine-speed
behavior is what gets seats rate-limited or restricted, so act like a person for
the whole run:

- **Pace every action.** Leave a short, slightly varied pause between actions — a
  beat between clicks, a longer pause after a page loads — instead of firing
  back-to-back at machine speed.
- **One profile at a time.** Open a profile, read it, pause, then move on. Don't
  open many profiles or tabs in a burst.
- **Keep volume sane per session.** Cap how many profiles you open and actions
  you take in a single run; spread large sourcing across sessions rather than
  hammering everything at once.
- **Navigate naturally.** Scroll to load results the way a person would; don't
  rapid-fire hundreds of programmatic scrolls.
- **Only the recruiter's own logged-in seat.** Never create an account, never
  enter credentials, never open a second session.
- **Back off on friction — never fight it.** If LinkedIn shows a slowdown, an
  "unusual activity" notice, a verification step, or a CAPTCHA, **stop
  immediately and hand control back to the recruiter.** Do not attempt to solve a
  CAPTCHA or work around a security check — that violates LinkedIn's terms and is
  the fastest way to get the seat banned.

## 0a. Read efficiently — minimize tokens, not just time

Pacing (§0) controls wall-clock speed to protect the seat. This controls
token cost, which is separate and just as real — full profile opens are the
single biggest cost driver in a run, so how you read each one matters:

- **Scope reads to the relevant content, not the whole page — and this isn't
  just a cost rule, it's an accuracy one.** A candidate profile page also
  renders a "Recruiting Tools" sidebar (Similar Profiles, Recommended
  matches) showing *other* candidates, and a naive full-page text grab
  mixes their headlines and job titles in with the one you're actually
  scoring. This caused a real false-positive live: a keyword hit for "Epic"
  that looked like it was on the candidate's own profile was actually from
  a different person's card in that sidebar. The cheap fix — LinkedIn always
  prints a "Recruiting Tools" heading right where that sidebar starts, on
  every profile, so cut the text there before searching it for anything:

  ```js
  const full = document.body.innerText;
  const idx = full.indexOf('Recruiting Tools');
  const profileText = idx > -1 ? full.slice(0, idx) : full;
  ```

  Run every keyword/date/gate check against `profileText`, never the raw
  page text. For the hard, no-exceptions gates specifically (Epic/VBC-style
  must-haves), pair this with a scoped `read_page` (pass `ref_id` for the
  profile section) as a second, DOM-structural way to exclude the sidebar —
  don't rely on the text-boundary trick alone when a wrong answer would
  wrongly pass or fail a categorical must-have.
- **Prefer text over screenshots — this means routine confirmations and
  dropdown clicks too, not just reading.** `find`, `get_page_text`, and
  targeted `javascript_tool` queries return plain text at a fraction of the
  token cost of a screenshot. After a stage change, save, hide, or archive
  action, confirm it landed by checking the toast/status text or the new
  stage label with a text query — not by taking a screenshot to read a
  sentence you could grep for. For picking an option out of a stage-picker
  dropdown, don't screenshot to find its pixel position and click by
  coordinate — LinkedIn's menu position shifts between the screenshot and
  the click often enough to land on the wrong option (verified live: this
  mis-set several candidates' stages this session). Instead, find the
  option by its exact visible text and click that element directly:

  ```js
  const opt = Array.from(document.querySelectorAll('li, div, button'))
    .find(el => el.textContent.trim().endsWith('Maybe')); // match the label, not a numbered prefix
  opt.click();
  ```

  **Two different dropdowns use two different label formats — matching only one
  causes silent misclicks on the other.** Verified live: the "Save to pipeline"
  dropdown (candidate not yet saved) lists plain labels ("uncontacted"), but the
  "Change stage" dropdown (candidate already saved) numbers every option ("1.
  uncontacted", "2. contacted", ...). A selector built for one format can match
  a decoy element under the other format and silently do nothing, or a stale
  screenshot coordinate can land one row off — this mis-set three consecutive
  candidates' stages in one run before being caught only because each one
  happened to get double-checked. Match both formats at once and require the
  element to be a real leaf (no children), not a wrapper that merely contains
  the text elsewhere on the page:

  ```js
  const target = 'uncontacted';
  const opt = Array.from(document.querySelectorAll('li, div, button, span'))
    .find(el => el.children.length === 0 &&
                new RegExp(`^(\\d+\\.\\s*)?${target}$`, 'i').test(el.textContent.trim()));
  opt.click();
  ```

  **The menu renders asynchronously after the dropdown arrow is clicked —
  verified live: searching for the option text immediately after that click
  can return nothing, because the menu's contents haven't painted yet.**
  This is a safe failure (the search finds no match, so nothing gets
  clicked) rather than a wrong one, but it still needs handling: put a short
  `computer` wait (around 1 second) between clicking the dropdown arrow and
  running the text-match search, and if the search still comes back empty,
  wait once more and retry before falling back to a screenshot. Never treat
  "no match found" as "already done" — that gap is exactly how a candidate
  ends up silently un-dispositioned.

  Then immediately confirm with a text check (not a screenshot) that the
  candidate's stage label actually changed — a click that silently missed
  its target is worse than a visible misclick, since nothing on screen
  flags it. Reserve `computer` screenshots for genuinely visual questions
  you can't answer from text — whether an element actually rendered, whether
  a layout is broken. If you catch yourself screenshotting after every
  single click just to read a one-line confirmation, that's the anti-pattern
  this bullet exists to prevent.
- **Batch sequences in one `browser_batch` call** wherever the steps are
  deterministic (scroll-then-read, open-then-extract) instead of separate
  round trips — this cuts overhead from re-transmitting context on every call.
- **Read each profile once, fully.** Extract everything needed for the
  Must-Have Gates and all three scoring dimensions in a single pass. Going
  back to re-check something on a profile you already read multiplies cost
  for no accuracy gain.
- **Never navigate a candidate's open profile tab away to check something
  else, then navigate back.** Verified live: doing this while mid-evaluation
  (leaving an open profile to go re-read the Project description, then
  returning to the same profile URL) very likely re-triggers a LinkedIn
  Recruiter profile-view charge on that candidate a second time — the
  credit meter appears to count page loads of a profile, not unique
  candidates. If you need to check something else while a profile is open,
  open it in a **separate tab** instead, so the candidate's own tab never
  reloads.
- **If a click doesn't produce the expected result, don't retry the same
  blind action repeatedly.** Verified live: a "Save to pipeline" dropdown
  failed to open after a click, and instead of checking why, the next step
  was three more blind attempts (a different element, the same coordinate
  again) — four tries and several screenshots before anything was actually
  diagnosed. After one click + the standard wait (§0a above), if the
  expected menu/state still isn't there, take a single screenshot to see
  what's actually on screen before trying again. Two failed attempts total
  is the limit — past that, stop and flag it rather than continuing to
  guess.

## 1. Build the boolean search

Translate the brief into an Advanced Search: title/keywords for the role family
(be tight — adjacent/generic titles hurt precision, per the Screening rules'
title-match rule), location, seniority, and any must-have skills. Prefer a
tighter query that returns fewer, better-matched people over a broad one you
have to wade through. Start the Job titles facet with the literal title from
the brief — see §1a for when and how to widen it to a researched set of
equivalent titles, which is a fallback lever for once this focused pool is
exhausted, not part of the initial search.

Run the search **from inside the role's project** (see §2) — its own
"Recruiter search" tab, not a standalone global search — so every result's
"Save to pipeline" action is already scoped to this project.

**Continuing an existing role? Check search history first, don't rebuild from
scratch.** Go to **Recruiter search history**
(`/talent/search/recruiter-search-history`) and find this project's most
recent entry — it shows the exact boolean query last used, tagged by project
name. Reopen and reuse it as the starting point instead of reconstructing the
whole keyword string from the brief again; only adjust what's actually
changed. Saves real time and avoids drifting from a query that already worked.

## 1a. When the pool runs out: widen the Job titles facet first

Run the initial search on the literal title from the brief and take that
pool all the way to exhaustion (or ~20 dispositioned fits, whichever comes
first — see `SKILL.md` workflow steps 6–7) before touching anything. Only
once that focused pool is genuinely used up does title-widening come in.
This is the only widening lever approved so far — loosening boolean keywords
or softening a must-have is **not** part of this workflow; if the pool is
still thin after widening titles, stop and tell the recruiter rather than
loosening anything else on your own.

To widen: the Job titles facet takes multiple entries. Research what other
titles real companies use for the identical job — this is role-agnostic,
not a fixed list, and applies the same way to any function. For a "Director
of Customer Success" role that might mean seniority variants (Director /
Senior Director / VP / Head of) crossed with function-label variants
(Customer Success / Client Success / Account Management / Client
Partnerships). For a "Full Stack Engineer" role it might mean Full Stack
Developer, Software Engineer (Full Stack), Full Stack Software Engineer,
Senior/Staff Full Stack Engineer, or plain "Software Engineer" at companies
that don't split front/back-end titling at all. Same principle either way: a
Series B startup's "Head of Customer Success" and an enterprise's "VP of
Customer Success" can be the exact same job at the exact same seniority —
the difference is company-specific titling convention, not a real
difference in role. A search scoped to one literal title string misses
those candidates on a labeling technicality, not because they're a worse
fit.

This is a precision-preserving move, not a precision-loosening one — the goal
is still the right function and seniority, just expressed as a researched
set of equivalent labels instead of a single string that undercounts the
real market. A title-only miss (same job, different label) is a cheaper,
more precise fix than broadening the substantive requirements, and it's
usually the real reason a pool looked thinner than the market actually is.
Note to the recruiter what titles you added and why.

## 1b. Add a Qualification — semantic matching on top of keywords

Alongside the boolean keyword string, use LinkedIn's own **"+ Qualification"**
field (in the filter panel, under "No qualifications evaluated") for the
must-haves that are hard to express as a keyword AND/OR — verified live: it
takes a plain sentence (e.g. "Experience managing enterprise healthcare
technology accounts and renewals") and matches on meaning, not exact wording,
which is exactly what nuanced must-haves need. This is a built-in LinkedIn
feature — no external tool, no added cost.

- It measurably tightens the pool: on a live test this took one search from
  "1K+" results down to 492, all still keyword-matched, now also ranked by
  relevance to that sentence.
- Cards that match well show a **"High qualification relevance"** badge —
  read this at Stage 1, for free, as an extra plausibility signal alongside
  title/location/seniority, before deciding whether a card is worth opening.
- Add one qualification per distinct must-have that keywords can't capture
  well (e.g. a scope-of-role or outcome-based requirement); don't overload it
  with everything from the brief — the boolean keywords still carry the hard
  filters (title, skills, tools).

## 1c. Sanity-check the pool size before committing to it

After building the query (keywords + filters + any Qualification), look at
the result count and **"See search breakdown"** (verified live: shows a bar
chart of the pool by current/past company, and other dimensions via the
"View" dropdown) before scrolling through candidates:

- **Still huge (1K+)** — the query is still too broad; tighten the keyword
  string or add a Qualification before spending time scrolling.
- **Very small (a handful)** — likely over-constrained; consider whether a
  filter or keyword is too strict before concluding the market is thin.
- Use the breakdown to sanity-check composition too — e.g. if company results
  are dominated by one or two employers, that's worth noticing before you
  conclude the pool represents the market broadly.

## 1d. Applying filters — go slow, verify each one

Filters are the foundation of the search. A filter that silently didn't land is
worse than not applying it at all — it gives you a falsely confident broad pool.
**One filter at a time, verified before moving on.** Never apply multiple filters
in a row. Apply one, confirm it registered (look for the chip/pill/badge LinkedIn
shows for active filters), then move to the next.

**Use `find` to locate the input, not hardcoded selectors.** Describe what you're
looking for in plain language (e.g. "the Location input", "the Seniority
checkboxes") — this is more reliable than CSS selectors as LinkedIn's DOM changes.

**For autocomplete inputs (Location, Job title, etc.):** type the value slowly
with the `computer` type action, then pause (~1s) for the suggestion dropdown to
appear, then use `find` to locate the matching suggestion and click it. If the
dropdown disappears before you click, don't retype immediately — wait a beat,
re-focus the input, and try again. Confirm the chip appeared before moving on.

**For checkboxes and toggles:** click the filter group to expand it, pause for
the panel to open, then click the specific option. Verify it's checked.

**If a filter isn't sticking after two clean attempts**, note it and move on
rather than looping — tell the recruiter at the end which filters couldn't be
applied so they can set them manually.

## 1e. Apply the target company list — paste the whole list in ONE action

When the recruiter chose "Target companies" in intake, read
`../references/target-companies.md` (or the same file relative to
`SKILL.md`: `references/target-companies.md`) and add every listed company in
a **single paste**. Never add them one at a time through the autocomplete —
for a long list that is painfully slow and resolves ambiguous names wrongly.

Build a **newline-separated** string of the company names, then dispatch a
synthetic paste event on the Companies input. Target the input by selector and
focus it first. (A real `Cmd+V` cannot work here: under browser automation the
page never holds OS focus — `document.hasFocus()` is `false` — so the clipboard
API and OS paste are blocked. The synthetic paste below is what works.)

```js
const input = document.querySelector('input[placeholder*="company" i]');
input.focus();
const dt = new DataTransfer();
dt.setData('text/plain', companyNames.join('\n'));   // one name per line
input.dispatchEvent(new ClipboardEvent('paste', {
  clipboardData: dt, bubbles: true, cancelable: true
}));
```

**Critical — how to know it worked (this is where a naive run goes wrong):**
LinkedIn creates the company chips **asynchronously**, and the input box stays
**empty** with no synchronous success signal. That empty input is NOT a failure.
Do **not** read `input.value` to decide success — it stays `""` even when the
paste worked, and if you treat that as failure you will wrongly fall back to
adding names one by one.

Instead, wait ~2 seconds, then **count the company pills**:

```js
await new Promise(r => setTimeout(r, 2000));
const added = document.querySelectorAll('.facet-pill__label').length;   // > 0 = success
```

A single dispatch adds the entire list. Companies already present are de-duped
automatically. **Never fall back to adding companies one at a time.** If the
pill count is still 0 after one retry, stop and tell the recruiter — do not
type names individually. This narrows the pool *alongside* the keyword
string, never instead of it.

## 2. Opening the project and reading/writing its brief

The recruiter creates and names their own project — **Maya never creates or
searches for one on her own; she asks which project to work in** (`SKILL.md`
step 2) and opens exactly that one.

- Navigate to **Projects** (`/talent/projects`) and open the named project
  directly (use the **"Search for a project"** box to find it by name if it's
  not on the first page).
- Once inside, go to **⚙ (gear icon, top right) → Project details** to read
  the **Project description** field — this is a genuine free-text textarea
  (verified live), and it's where the signed-off brief lives. Empty = new
  role, go to full intake. Already has text = continuing role, read it back
  and route to "Working on a role that already exists" in `SKILL.md`.
- After sign-off (new role) or after confirming/updating (continuing role),
  write the current, canonical brief into that same **Project description**
  field via **Edit → Save**. Overwrite, don't append — it should always read
  as the current brief, not a history of every version.
- Once you're in the right project, use its own left-sidebar **Recruiter
  search** tab to run the search described in §1 — this keeps every result
  scoped to this project automatically.

## 3. Extract candidates from the virtualized list

The results list is **virtualized / lazy-rendered** — only the visible cards
exist in the DOM. You must scroll to force new cards to render, then read them.

- Scroll with the `computer` tool's mouse-wheel `scroll` action at roughly
  `[940, 400]` (over the results pane). This triggers the IntersectionObserver
  that renders the next batch. `browser_batch` can chain several scroll+read
  steps and is faster than one call at a time.
- Read profile links with `javascript_tool`:

  ```js
  document.querySelectorAll('a[href*="/talent/profile/"]')
  ```

- The profile ID is parsed from the href:

  ```js
  href.split('/talent/profile/')[1].split(/[?\/]/)[0]   // e.g. "AEMAAANPZEQBY8..."
  ```

- The real, linkable profile URL is then:

  ```
  https://www.linkedin.com/talent/profile/{profileId}
  ```

  These are Recruiter-seat URLs — they open inside Recruiter and require the
  user's login. That is expected; they are the correct links to store.

## 4. Prior engagement / ATS history

LinkedIn Recruiter surfaces a lot of history right on the card and on the
full profile — "In contacted," "In replied," an accepted/declined InMail,
"Applied to \<job\>," "In Comeet." This splits into two very different
buckets:

**ATS-sync or applied ("In Comeet" or your ATS's name, "Applied to \<job\>")
— route at the card level, before opening anything.** Read each card's text:

```js
// cardText = the visible text of one result card
const atsOrApplied = /In Comeet|Applied to/i.test(cardText);
```

If true, save straight to the **"Already in ATS"** pipeline stage using the
**"Save to pipeline" dropdown arrow** (same one-action mechanic as any other
disposition — see §8). No profile read, no gate, no score, no note. The
recruiter reviews "Already in ATS" manually inside the project. Opening the
real ATS record or the in-app "Comeet" tab to find out what actually
happened there costs a full profile-open cycle per candidate, which is too
expensive for a bucket this size — don't open either one.

**Everything else — contacted, replied, an accepted/declined InMail, any
InMail stage on another project — changes nothing about how a candidate
gets screened.** Every one of these candidates proceeds to the same full
profile-open and scoring pass in §8, regardless of what history shows, and
lands in the same outcome set as anyone else: Hide, Maybe, uncontacted, or
Moved Recently - Less than 1 year (see §8). There's no mirroring a
candidate's contacted/replied/InMail stage from another project into this
one — that history is still visible to the recruiter on the candidate's own
profile page whenever they open it; it's just never used to shortcut or
change a decision. Companies often run several differently-named projects
for what's really one role, so seeing this kind of history on a candidate
is common and expected — it's not a signal to act on, beyond the ATS/applied
case above.

## 5. Work around javascript_tool truncation

`javascript_tool` truncates long output strings, which corrupts extraction if
you dump everything at once. Two habits fix this:

- Pull data in **small batches** (about 6 candidates per call), not all at once.
- Sanitize noisy digit runs and URL punctuation before returning strings:

  ```js
  s.replace(/\d{4,}/g, '#').replace(/[?&=]/g, ' ')
  ```

## 6. Accumulate across scrolls

Because the DOM only holds visible cards, keep a running accumulator keyed by
profile ID so you don't lose people as they scroll out of view. A page-scoped
object persisted to localStorage survives navigation within the session:

```js
window.__M = window.__M || {};              // keyed by profileId
// ...merge each newly-read card in...
localStorage.setItem('__MAYA_RUN', JSON.stringify(window.__M));
```

Restore from localStorage after any navigation.

## 7. Fill to the target count

Keep scrolling and extracting until you have enough **Stage-1 survivors**
(plausible on title/location/seniority) to deliver ~20 after full screening. If the pool is thin — or the search exhausts before reaching
~20 dispositioned fits — the one approved widening lever is researching and
adding more equivalent titles to the Job titles facet (§1a). Never loosen
boolean keywords or soften a must-have on your own — if the pool is still
thin after widening titles, stop and tell the recruiter what limited it
rather than padding the shortlist with poor matches to hit a number.

## 8. Stage 2 — score, then write straight into the project

**"Open the profile" means the actual full LinkedIn profile page, not the
expanded search-results card.** The search-results list has a "Show all"
toggle that expands the job-history list inline — that is still Stage 1's
card, not Stage 2. It's tempting to treat the expanded card as "good enough"
since it shows full dates and titles, but it's missing the About/summary
section, the candidate's full skill list (not just LinkedIn's auto-picked
"matches"), and recommendations — exactly the content that resolves
borderline calls (a specialization that reframes a title, a stated
skill the JD needs that the card didn't surface, a recommendation that
corroborates or contradicts a claimed scope). For every Stage-1 survivor,
without exception — not just the ones that look borderline from the card —
navigate to their full profile before applying any gate or score. Never
gate or score off the expanded card alone.

Once on the full profile, **check whether the candidate's current employer is
the hiring company itself, before anything else.** LinkedIn Recruiter search
results can include a company's own employees (flagged as "Internal
candidates" in the Spotlights row on the search page). An internal candidate
isn't a real external sourcing result — skip them straight away (Hide, no
scoring, no gates) without stopping to ask. This is a standing rule, not a
one-off judgment call.

Next, **check for a "Current project" tag.** The candidate view shows a line like `In 1 project ·
<project name> · Current project` when they're already in the project you're
working from. If that tag is present, skip them — they're already covered by
a prior run on this exact project, regardless of what Stage 1's card-level
check suggested. This is the authoritative check that makes repeat runs on
the same project safe.

Otherwise, extract the history — including the About section, full skills
list, and any recommendations — and run the Scoring rubric in `SKILL.md` —
Must-Have Gates first (fresh-hire handled specially, see below), then the
weighted score (Core requirements 50% / Experience 35% / Stability 15%),
then the band.

**Every candidate gets a disposition — never move on without one.**

For **No Go** (or any other failed gate): click **Hide** on the candidate's
card or profile. Verified live: this collapses the card to "won't appear in
any of your search results for this project" and increments the project's
Hidden count — it's project-scoped and reversible by the recruiter, not
destructive. This is what makes a later run on the same project cheap: a
hidden candidate is filtered out of this project's search entirely, so
there's no Stage-1 or Stage-2 cost re-encountering them at all.

For **Good Match / Strong Match / Not Sure / a parked fresh-hire** (anyone
who gets saved at all), save and stage in **one action**: click the dropdown
arrow next to **Save to pipeline** (not the button itself) — this opens
**"Save to pipeline stage..."** with every stage listed, and clicking one
saves the candidate and sets that stage together (verified live). Pick:

- Good Match / Strong Match → **uncontacted** (the default stage — nobody's
  been contacted yet).
- Not Sure → **Maybe** (verified live: an existing account-wide stage,
  available on every project, no setup needed).
- Fresh-hire, otherwise a Good Match+ → **Moved Recently - Less than 1
  year** (also verified live, account-wide).

Every candidate who reaches this point goes through the same gate/score/
stage decision — prior engagement (contacted, replied, InMail) never routes
a candidate around it. The one exception is an ATS-sync/applied signal,
which is diverted to "Already in ATS" at §4, before ever reaching this
Stage-2 list at all.

That's the entire disposition — no note, no tag. **No notes, ever** — the
stage alone carries the band; adding a note is extra clicking for
information the recruiter can already see from the stage and the profile
itself. **No tags, ever** — LinkedIn Recruiter's tag list (⋯ → Add tag) is a
fixed, pre-existing set per account with no free-text or on-the-fly creation
(verified live: typing a new name shows no "create" option, just a
checklist of existing tags).

Stop once you have ~20 Good Match/Strong Match fits, or the pool genuinely
runs out — no cap on how many profiles you open to get there — see
`SKILL.md` for the fit-gate and the ceiling-not-floor rule. Not Sure,
parked fresh-hires, and the ATS-synced/applied bucket don't count toward
that ~20.
