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

## 1. Build the boolean search

Translate the brief into an Advanced Search: title/keywords for the role family
(be tight — adjacent/generic titles hurt precision, per the Screening rules'
title-match rule), location, seniority, and any must-have skills. Prefer a
tighter query that returns fewer, better-matched people over a broad one you
have to wade through.

Run the search **from inside the role's project** (see §2) — its own
"Recruiter search" tab, not a standalone global search — so every result's
"Save to pipeline" action is already scoped to this project.

## 1a. Applying filters — go slow, verify each one

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

## 1b. Apply the target company list — paste the whole list in ONE action

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

## 2. Finding or creating the role's project

Maya keeps her own project per role — this replaces any external
role/shortlist tracker, and it's kept separate from whatever project the
recruiter already uses to manually track that req.

- Name it **"\<Role title\> - Maya Sourcing"** (add location only if needed
  to disambiguate two open reqs with the same title). Never reuse or write
  into a project that doesn't have that suffix — that's the recruiter's own
  project, not Maya's.
- Navigate to **Projects** (`/talent/projects`) and use the **"Search for a
  project"** box to check for an existing project with that exact name before
  creating a new one. This is the collision check from `SKILL.md` step 4.
- If none matches, create a new one. Use `find` to locate the project-creation
  control on the Projects page (its exact placement can shift) rather than a
  hardcoded selector.
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

## 4. Stage 1 — dedup / already-engaged signals, at the card level

Do this **before** opening any profile — it's the main cost lever, since it
skips the expensive open+read cycle entirely for candidates you'd reject
anyway. Read each card's text and reject/skip on any of:

```js
// cardText = the visible text of one result card
const inATS        = /\bIn Comeet\b/i.test(cardText);           // swap "Comeet" for your ATS's name
const alreadyContacted = /Contacted on/i.test(cardText);        // "Contacted on <date> by <name>"
const alreadyInPipeline = /Change stage/i.test(cardText) && !/Save to pipeline/i.test(cardText);

const alreadyEngaged = inATS || alreadyContacted || alreadyInPipeline;
```

`alreadyInPipeline` works because a fresh, untouched candidate's card shows a
**"Save to pipeline"** button; a candidate already in *some* project's
pipeline shows **"Change stage" / "Archive"** instead, plus a stage label like
"In contacted" or "In replied".

Exclude all of these from the ranked shortlist (you may note them separately,
but they never occupy one of the ~20 slots, and you never spend an open+read
cycle confirming them further). Keep a running count to report at the end.

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

Keep scrolling and extracting until you have enough **Stage-1 survivors** (not
already-engaged, plausible on title/location/seniority) to deliver ~20 after
full screening. If the pool is thin, widen the query slightly and note that to
the recruiter rather than padding with poor matches.

## 8. Stage 2 — score, then write straight into the project

For each Stage-1 survivor: open the profile, extract the history, and run the
Scoring rubric in `SKILL.md` — Must-Have Gates first, then the weighted score
(Core requirements 50% / Experience 35% / Stability 15%), then the band. For
everyone who lands Good Match (60+) or above:

1. **Save to pipeline** — from the candidate's card or open profile, click
   **Save to pipeline**. Because you're working from inside the role's
   project's own search tab (§2), this saves straight into that project.
2. **Add the rationale** — open the **⋯** menu on the candidate and choose
   **Add note**. Lead with the band, then the score breakdown, then a short
   evidence-based rationale, e.g.:
   `Strong Match — Score: 87/100 — Core requirements 90/100 (50%), Experience
   85/100 (35%), Stability 80/100 (15%). 6y B2B SaaS AE, hit 130%+ quota 3
   years running, direct healthcare-vertical experience.`
   Leave visibility on its default, **"Members of \<project\>"**, so the
   whole team can see it. **Don't try to add a "Good Match"/"Strong Match"
   tag** — LinkedIn Recruiter's tag list (⋯ → Add tag) is a fixed,
   pre-existing set per account with no free-text or on-the-fly creation
   (verified live: typing a new name shows no "create" option, just a
   checklist of existing tags). Use an existing tag only if one already
   clearly means the same thing; otherwise the note is enough.

Stop once you have ~20 genuine fits, ~50 profile opens, or the pool runs out
— see `SKILL.md` for the fit-gate, the profile-open cap, and the
ceiling-not-floor rule.
