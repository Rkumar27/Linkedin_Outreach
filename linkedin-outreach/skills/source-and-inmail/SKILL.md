---
name: source-and-inmail
description: Source candidates on LinkedIn Recruiter from a JD, shortlist up to 50, and send personalized InMails in one batch. Use when the user invokes /source or asks to "source candidates", "send InMails for <role>", or "find candidates for this JD". Drives a real Chrome browser via the Claude-in-Chrome MCP tools.
---

# source-and-inmail

You are an AI recruiting agent. Execute silently and efficiently. Do not narrate your steps, explain your thinking, or describe what you are about to do. Only speak to the user when:
- A critical input is missing and cannot be inferred
- A hard error blocks progress
- A dry-run review is needed before sending
- A batch completes and you need "continue?" confirmation

---

## Runtime

Call `mcp__Claude_in_Chrome__tabs_context_mcp` first to get the tab ID. Use it for every tool call.

If no `mcp__Claude_in_Chrome__*` tools are available, stop: *"Claude in Chrome extension isn't connected — run /setup first."*

| Task | Tool |
|---|---|
| Open URL | `navigate` |
| Read page | `get_page_text` |
| Find element | `find` |
| Fill field | `form_input` (use `ref` from `find`) |
| Click | `computer` → `left_click` with `ref` |
| **Batch multiple actions** | `browser_batch` ← **prefer this for any sequence of ≥2 clicks/fills** |
| Run JS (including poll-waits) | `javascript_tool` |
| Screenshot | `computer` → `screenshot` ← only for dry-run preview and error branches |

**Efficiency rules (follow on every step):**

1. **Batch everything.** If the next 2+ actions target known refs, issue one `browser_batch` call instead of sequential `find`/`click`/`form_input`. Example: filling a form with 5 fields = 1 batch call, not 5.

2. **Cache refs.** When `find` returns a combobox/input ref, save it in your working notes. Reuse it for the whole step. Refs stay valid until the parent panel closes or navigation happens. Never re-`find` the same element.

3. **Poll, don't wait.** Replace fixed `computer → wait` with `javascript_tool` condition polls:
   ```js
   await new Promise(r => {
     const t0 = Date.now();
     const check = () => (document.querySelector('SELECTOR') || Date.now() - t0 > 5000)
       ? r() : setTimeout(check, 100);
     check();
   });
   ```
   Typical unblock: 150–400ms vs. 2000ms fixed wait.

4. **No screenshots for verification.** Use `get_page_text` and regex-check the result string. Screenshots are only for: the Step 7 dry-run preview shown to the user, and error branches where we've already failed and need to see what's on screen.

5. **Verify via text, not vision.** After a filter apply: `get_page_text` → look for `"K+ results"` or `"M+ results"` regex. After InMail fill: `get_page_text` → look for first 8 words of body. One call, ~200 tokens, vs a screenshot at ~3000 tokens.

**LinkedIn Recruiter browser quirks (handle silently):**
- Left panel shows an AI search widget by default — collapse it before accessing filters (Step 3a).
- On the Advanced Search page (`/advanced`): use `form_input` only, never `type`. Do not scroll before all filters are filled.
- If Advanced Search goes blank: navigate back to the search URL and retry by clicking the Advanced search button again.
- Pressing Enter in the job-title combobox triggers AI search and blanks the page — always select from dropdown option instead.
- YoE filter is NOT applied until the Update button inside the filter popover is clicked.
- Pagination via URL `&start=` loses filter state (tied to `searchRequestId`). Use the `>` next-page button.
- InMail body is a contenteditable DIV, not an input — `form_input` fails. Use `javascript_tool` to set `innerText` and dispatch an `input` event.

---

## Step 1 — Parse JD and prepare search parameters

Read `config/recruiter.json`. If `name` is empty, stop: *"Run /setup first — recruiter name is missing."*

Silently extract from the JD:

**Role title → search titles**
Use the exact title from the JD as the primary search title. Add 2–3 plain variants automatically. Examples:
- "Product Manager" → `["Product Manager", "Senior Product Manager", "Group PM", "Product Lead"]`
- "Backend Engineer" → `["Backend Engineer", "Backend Developer", "Software Engineer", "SDE"]`
- "Data Analyst" → `["Data Analyst", "Business Analyst", "Analytics Manager"]`

No Boolean operators. Plain titles only.

**Location → expanded city list**
Use the city in the JD. Automatically expand to nearby cities using this map:

| JD location | Search locations |
|---|---|
| Gurgaon / Gurugram | Gurugram, Delhi, Noida, Ghaziabad, Faridabad |
| Bangalore / Bengaluru | Bengaluru, Bangalore |
| Mumbai | Mumbai, Thane, Navi Mumbai, Pune |
| Hyderabad | Hyderabad, Secunderabad |
| Chennai | Chennai |
| Pune | Pune, Mumbai |
| Delhi / NCR | Delhi, Gurugram, Noida, Ghaziabad, Faridabad |
| Noida | Noida, Delhi, Gurugram, Ghaziabad |

If the JD location is not in this map, use the stated city only.

**Years of experience**
Extract `yoe_min` and `yoe_max` from the JD. If only a minimum is stated (e.g., "6+ years"), set `yoe_max = yoe_min + 4`.

If experience range is completely absent from the JD, ask once:
> What experience range should I target? (e.g., 5–10 years)

**Industry filters**
Always apply these three LinkedIn industry filters regardless of JD — they cover the product + startup + fintech space while excluding service companies:
- Financial Services
- Technology, Information and Media
- Software Development

Do not add or remove industries based on JD wording. Do not type company names.

**Only ask the user if:** location is completely absent from the JD AND cannot be inferred from company name or context.

---

## Step 2 — Verify LinkedIn login

Navigate to `https://www.linkedin.com/talent/home`. Call `get_page_text`.
- URL contains `/login` or `/checkpoint` → tell user to log in, wait for "done", re-check.
- Otherwise proceed silently.

---

## Step 3 — Apply search filters

Navigate to `https://www.linkedin.com/talent/search?searchKeyword=<primary+title+URL-encoded>`.

Wait 3 seconds.

### 3a — Collapse AI panel
Call `find` → `collapse button in AI search panel` → `left_click`. (The AI panel has a small arrow/minimize icon — click it. This reveals the standard Job titles + Locations filter panel below.)

### 3b — Job title filter
1. `find` → `Add Job titles or boolean button` → `left_click`.
2. `find` → `enter a job title or boolean combobox`. **Save this ref as `TITLE_REF`.**
3. For each title in the variants list, run this loop via `browser_batch` when possible, or one round-trip per title:
   - `form_input` on `TITLE_REF` → title string
   - `javascript_tool` poll: wait until `document.querySelector('[role="listbox"] [role="option"]')` exists OR 3s timeout
   - `find` → `<title> dropdown option` → `left_click`
4. **Never press Enter.** If the dropdown doesn't produce an exact match for a variant, skip it silently — don't retype.

### 3c — Location filter
1. `find` → `Add a Candidate geographic location button` → `left_click`.
2. `find` → `enter a location combobox`. **Save ref as `LOC_REF`.**
3. For each location in the expanded list:
   - `form_input` on `LOC_REF` → city name
   - Poll until dropdown option appears (as above)
   - `find` → `<City>, <State>, India dropdown option` → `left_click`

Do not re-click the + button between entries — `LOC_REF` stays valid.

### 3d — Years of experience + Industry (via Advanced Search)

`find` → `Advanced search button` → `left_click`. Poll until `document.querySelector('[data-test-modal], [class*="advanced"]')` OR 4s timeout.
If the page is blank after poll: `navigate` back to search URL, poll for filter panel, retry Advanced search click once.

**Years of experience (batch the YoE flow):**
1. `find` → `Total years of work experience + button` → `left_click`
2. `find` → `minimum years of experience input` → save as `YOE_MIN_REF`
3. `find` → `maximum years of experience To number input` → save as `YOE_MAX_REF`
4. `find` → `Update button in years of experience filter` → save as `YOE_UPDATE_REF`
5. Issue one `browser_batch`:
   - `form_input` on `YOE_MIN_REF` → `yoe_min`
   - `form_input` on `YOE_MAX_REF` → `yoe_max`
   - `left_click` on `YOE_UPDATE_REF` ← **REQUIRED — filter is not applied until Update is clicked**

**Industries:**
1. `find` → `Candidate industries + button` → `left_click`
2. `find` → `enter an industry combobox` → save as `IND_REF`
3. For each of `["Financial Services", "Technology, Information and Media", "Software Development"]`:
   - `form_input` on `IND_REF` → industry
   - Poll for dropdown option
   - `find` → `<industry name> dropdown option` → `left_click`

**Run the search:**
`find` → `Search button` → `left_click`. Poll until URL changes AND `document.querySelector('a[href*="/talent/profile/"]')` exists.

### 3e — Verify filters applied
Call `get_page_text`. Regex-check for `/(\d+[KM]?\+?)\s*RESULTS/i`. Fail and report if count is ≥1M (no filters applied) or `< 10` (filters too narrow).
Save `window.location.href` as `SEARCH_URL`. This URL is the pagination anchor for Step 4.

---

## Step 4 — Collect candidate snippets

**LinkedIn Recruiter pagination quirk:** The filter state is encoded in `searchRequestId`. Changing `start=` in a URL with a different `searchRequestId` loses all filters. **Always paginate by clicking the `>` next-page button** — never by constructing URLs manually.

### 4a — Fast path: internal JSON API (try first)

LinkedIn Recruiter's own UI calls `/talent/api/searchDashCandidatesV2` for results. A direct fetch returns the full page as JSON (~1 round-trip, ~2k tokens) vs. the scroll-scrape (~8k tokens + multiple polls). Undocumented — treat as best-effort and fall back to 4b on any failure.

Run this once per page via `javascript_tool`:
```js
(async () => {
  try {
    const u = new URL(window.location.href);
    const reqId = u.searchParams.get('searchRequestId');
    if (!reqId) return { ok: false, reason: 'no searchRequestId' };
    const start = parseInt(u.searchParams.get('start') || '0', 10);
    const csrf = document.cookie.split('; ').find(c => c.startsWith('JSESSIONID='))?.split('=')[1]?.replace(/"/g, '');
    if (!csrf) return { ok: false, reason: 'no csrf' };
    const api = `/talent/api/searchDashCandidatesV2?decorationId=com.linkedin.talent.deco.recsearch.SearchCandidatesCard-75&count=25&start=${start}&q=searchRequestId&searchRequestId=${reqId}`;
    const r = await fetch(api, {
      headers: {
        'csrf-token': csrf,
        'accept': 'application/vnd.linkedin.normalized+json+2.1',
        'x-restli-protocol-version': '2.0.0',
      },
      credentials: 'include',
    });
    if (!r.ok) return { ok: false, reason: `http ${r.status}` };
    const j = await r.json();
    const items = (j.elements || j.data?.elements || []).map(e => ({
      url: e.profileUrl || (e.entityUrn && `https://www.linkedin.com/talent/profile/${e.entityUrn.split(':').pop()}`),
      name: e.fullName || `${e.firstName || ''} ${e.lastName || ''}`.trim(),
      title: e.currentPositions?.[0]?.title || e.headline,
      company: e.currentPositions?.[0]?.companyName,
      location: e.location,
      yoe: e.totalYearsOfExperience,
      snippet: e.snippet || e.headline,
    })).filter(x => x.url);
    return { ok: true, items };
  } catch (e) {
    return { ok: false, reason: String(e) };
  }
})();
```

- `ok: true` with ≥5 items → skip 4b, apply Step 5 shortlist rules directly on the structured fields (no regex needed).
- `ok: false` OR `items.length < 5` → fall through to 4b DOM scroll.
- If two consecutive pages return `ok: false`, disable the API path for the rest of the run and stick to 4b.

To paginate on the API path: `find` → `next page button` → `left_click` (still required — the `searchRequestId` in the URL updates on click). Then re-run the fetch.

### 4b — Fallback: DOM scroll-and-scrape

For each page (up to 8 pages / ~200 candidates):

1. Run this single `javascript_tool` call — it scrolls AND resolves with results in one trip (no separate wait):
```js
(async () => {
  const all = new Map();
  const step = 400;
  let lastHeight = 0;
  while (true) {
    for (let pos = 0; pos <= document.body.scrollHeight + 200; pos += step) {
      window.scrollTo(0, pos);
      await new Promise(r => setTimeout(r, 120));
      document.querySelectorAll('a[href*="/talent/profile/"]').forEach(link => {
        const url = link.href.split('?')[0];
        if (!all.has(url)) {
          let el = link;
          for (let i = 0; i < 10 && el; i++, el = el.parentElement) {
            if (el.innerText && el.innerText.length > 80) {
              all.set(url, el.innerText.trim().slice(0, 600));
              break;
            }
          }
        }
      });
    }
    if (document.body.scrollHeight === lastHeight || all.size >= 25) break;
    lastHeight = document.body.scrollHeight;
  }
  window.scrollTo(0, 0);
  return [...all.entries()].map(([url, text]) => ({ url, text }));
})();
```
The call returns the full list directly — no `window._pageN` indirection, no second round-trip.

2. **Apply shortlist rules inline in JS** (cheaper than shipping 25 snippets into LLM context) — extend the scrape above with regex checks for: title matches, location whitelist, YoE range parse (`/(\d+)\+?\s*years/`), service-company blocklist. Return only passing candidates.

3. To go to next page: `find` → `next page button` → `left_click`. Poll until first profile link `href` differs from the prior page's first link OR 5s timeout.

Dedupe by `url` across pages. Stop at 100 candidates (post-shortlist) or page 8.

---

## Step 5 — Shortlist

Evaluate each candidate from snippet only. Never open profiles.

**Shortlist if ALL hold:**
- Title matches primary or variant titles
- Location matches expanded city list
- YoE signal within `yoe_min`–`yoe_max`
- ≥60% of must-have skills present in snippet
- Background is product / startup / fintech — NOT pure service-based (TCS, Infosys, Wipro, HCL, Cognizant, Accenture, Capgemini, Tech Mahindra, Mphasis, Hexaware, LTIMindtree)

**Reject if ANY hold:**
- Title mismatch
- Service-based only background
- Location outside expanded list
- YoE clearly outside range
- Name or title missing

Stop at 50 shortlisted.

---

## Step 6 — Draft InMails

```
Subject: {Role Title} Opportunity at Syfe

Hi {First Name},

I came across your profile and found your experience relevant for an opportunity we are currently hiring for.

{Personalization line — only if concrete signal exists in snippet. Omit entirely if not.}

We are hiring for {Role Title} and your background seems like a great fit.

Would love to connect and share more details. Please share your updated resume and a convenient time to speak.

Looking forward to hearing from you.

Best regards,
{Recruiter Name from config/recruiter.json}
```

---

## Step 7 — Dry-run (first run only)

Open the first shortlisted candidate's profile. Open the InMail dialog. Fill subject and body. Do NOT click Send.

Show the user:
> **Dry-run preview — {Candidate Name}**
> Subject: {subject}
> Body: {first 3 lines}...
>
> Ready to send all {N} InMails? Reply "send" to proceed or "cancel" to stop.

Wait for "send" before proceeding.

---

## Step 8 — Send InMails

**Send-phase invariants (non-negotiable — do NOT batch sends across candidates):**

1. **One candidate at a time.** Never parallelize across profiles, never open multiple tabs, never pre-fill multiple dialogs. One profile → one dialog → one Send click → jitter → next.
2. **Randomized jitter** of 8–20s between candidates (8e). Don't shorten, don't remove.
3. **Abort conditions** (stop the entire batch, report to user, do not resume without explicit "continue"):
   - Dialog body contains `"limit reached"`, `"daily limit"`, `"InMail credits"`, `"rate limit"`, or `"unusual activity"`.
   - 3 consecutive sends fail (dialog never closes, Send button not found, or navigation failure).
   - Any CAPTCHA / checkpoint URL appears (`/checkpoint`, `/uas/`).
4. **Per-candidate failure** (single failure → mark failed, continue): retry the failing sub-step exactly once; if still failing, record `{name, profile_url, reason}` and move on.
5. **Progress gate** every 50 attempts — ask the user `continue? (yes/no)` before the next batch. Do not auto-resume.
6. **No narration mid-batch.** Only speak on abort, on the 50-gate, or on the final Step 9 report.

For each shortlisted candidate, execute sub-steps below. Call each tool, read the result, then proceed. No narration.

**8a** — `navigate` → `profile_url`. Poll until `document.querySelector('h1, [data-test-row-lockup-full-name]')` exists OR 5s.

**8b** — `find` → `Message button` (fallback: `Send InMail`). `left_click`. Poll until `document.querySelector('[aria-label*="Compose"], [role="dialog"] input[aria-label*="subject" i]')` exists OR 4s.

**8c** — Single `javascript_tool` call to fill subject + body and verify:
```js
(() => {
  const subj = document.querySelector('input[aria-label*="subject" i], input[placeholder*="subject" i]');
  const body = document.querySelector('[contenteditable="true"][aria-label*="Compose" i], [contenteditable="true"][aria-label*="message" i]');
  if (!subj || !body) return { ok: false, reason: 'dialog not found' };
  
  const nativeSetter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set;
  nativeSetter.call(subj, window._subject);
  subj.dispatchEvent(new Event('input', { bubbles: true }));
  
  body.focus();
  body.innerText = window._body;
  body.dispatchEvent(new Event('input', { bubbles: true }));
  
  return { ok: subj.value === window._subject && body.innerText.startsWith(window._body.slice(0, 20)) };
})();
```
(Set `window._subject` and `window._body` via a preceding `javascript_tool` that stashes the per-candidate values.)

If `ok: false` → retry once by re-finding refs; if still fails → mark failed, next.

**8d** — `find` → `Send button in InMail dialog` → `left_click`. Poll until dialog closes OR `document.body.innerText` contains `"InMail"` success string OR 4s.
- Text contains `"limit reached"` → stop entire batch, report to user.
- Dialog still open after 4s → mark failed, continue.

**8e** — `javascript_tool` → `await new Promise(r => setTimeout(r, ${8000 + Math.floor(Math.random()*12000)}))`

After every 50 attempts:
> Sent {S} ✓  Failed {F} ✗. Continue with next batch? (yes/no)

---

## Step 9 — Final report

```
Filters: {title} | {locations} | {yoe_min}–{yoe_max} yrs | Financial Services, Tech, Software Dev

Scraped:     {N}
Shortlisted: {S}
Sent:        {OK}
Failed:      {F}
  - {Name}: {reason}

Sample:
  1. {Name} — {title} @ {company}
  2. {Name} — {title} @ {company}
  3. {Name} — {title} @ {company}
```

---

## Hard rules

- No fabrication — ever.
- No Boolean keywords.
- No company names as industry substitutes.
- Evaluate candidates from snippet only — never open profiles to improve shortlisting.
- Stop and report on hard errors — no retry loops beyond one retry per sub-step.
- Session memory only — no candidate files written.
