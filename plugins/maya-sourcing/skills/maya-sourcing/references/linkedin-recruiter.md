# Searching LinkedIn Recruiter and extracting candidates

This is the mechanical part of a run: turning the signed-off brief into a
LinkedIn Recruiter search and pulling real candidate profiles out of the page.
It uses the Claude in Chrome tools and requires the recruiter to be logged into
their own LinkedIn Recruiter seat in the browser.

## Prerequisites

- The user is logged into LinkedIn Recruiter (`linkedin.com/talent/...`) in a
  Chrome tab connected to Claude in Chrome. If not, stop and ask them to log in
  — you cannot and must not enter their credentials.
- You have the signed-off brief.

## 1. Build the boolean search

Translate the brief into an Advanced Search: title/keywords for the role family
(be tight — adjacent/generic titles hurt precision, per the Global title-match
rule), location, seniority, and any must-have skills. Prefer a tighter query
that returns fewer, better-matched people over a broad one you have to wade
through.

Navigate to the Recruiter Advanced Search and run it.

## 2. Extract candidates from the virtualized list

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

## 3. Deduplicate against the ATS

Anyone already in the applicant tracking system must not take a shortlist slot.
On each card, the ATS shows as activity text like "In Comeet-Parent". Detect it:

```js
/\bComeet\b/i.test(cardText)   // true = already in ATS, exclude from shortlist
```

Exclude these from the ranked shortlist (you may note them separately, but they
never occupy one of the ~20 slots).

## 4. Work around javascript_tool truncation

`javascript_tool` truncates long output strings, which corrupts extraction if
you dump everything at once. Two habits fix this:

- Pull data in **small batches** (about 6 candidates per call), not all at once.
- Sanitize noisy digit runs and URL punctuation before returning strings:

  ```js
  s.replace(/\d{4,}/g, '#').replace(/[?&=]/g, ' ')
  ```

## 5. Accumulate across scrolls

Because the DOM only holds visible cards, keep a running accumulator keyed by
profile ID so you don't lose people as they scroll out of view. A page-scoped
object persisted to localStorage survives navigation within the session:

```js
window.__M = window.__M || {};              // keyed by profileId
// ...merge each newly-read card in...
localStorage.setItem('__MAYA_RUN', JSON.stringify(window.__M));
```

Restore from localStorage after any navigation.

## 6. Fill to the target count

Keep scrolling and extracting until you have enough **non-ATS** candidates to
deliver ~20 after screening. If the pool is thin, widen the query slightly and
note that to the recruiter rather than padding with poor matches.

## 7. Hand off to scoring

Once you have the raw pool, score each candidate 0–100 against the brief and the
layered Notion rules, attach a short rationale and explicit flags, take the top
~20, and write them to the role's shortlist database (see SKILL.md).
