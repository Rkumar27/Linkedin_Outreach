---
name: handle-replies
description: Read recent LinkedIn Recruiter InMail replies, classify each, draft a tailored response, and send only after user approval. Use when the user invokes /replies or asks to "check LinkedIn replies", "reply to candidates", or "handle InMail responses". Drives Chrome via Claude-in-Chrome MCP tools.
---

# handle-replies

You triage replies to past InMails on LinkedIn Recruiter. You **never auto-send** — every reply draft is shown to the user for approval.

## Runtime

You drive Chrome via **`mcp__Claude_in_Chrome__*` MCP tools**. Before calling any MCP tool, verify availability. If unavailable, tell the user: *"Claude in Chrome extension isn't connected — run /setup first."*

Useful MCP tools:

| Action | Tool |
|---|---|
| Open / switch URL | `mcp__Claude_in_Chrome__navigate` |
| Read current page as text | `mcp__Claude_in_Chrome__get_page_text` or `read_page` |
| Find elements | `mcp__Claude_in_Chrome__find` |
| Type into a field | `mcp__Claude_in_Chrome__form_input` |
| Click | `mcp__Claude_in_Chrome__computer` |
| Run JS for scraping | `mcp__Claude_in_Chrome__javascript_tool` |

## Plugin root

`/Users/rohitkumar/Documents/Claud Plugin/linkedin-outreach` — read `config/recruiter.json` for the signature.

## Steps

### Step 1 — Verify login

`navigate` to `https://www.linkedin.com/talent/home`. If the URL contains `/login`, `/uas/login`, or `/checkpoint`, prompt the user to log in manually, wait, and re-check. Do not proceed until logged in.

### Step 2 — Read the inbox

`navigate` to `https://www.linkedin.com/talent/inbox?filter=UNREAD`. Wait for the list to render.

Use `javascript_tool` to extract threads:
```js
[...document.querySelectorAll('a[href*="/talent/inbox/"]')]
  .map(a => ({ thread_url: a.href, preview: a.innerText.trim() }))
  .filter(t => t.thread_url);
```

For each thread (up to 25), open it by `navigate` to `thread_url`, then scrape:
- `candidate_name` — from the thread header (usually the first `<h1>` or `<h2>` in the detail panel).
- `last_message` — the text of the last message bubble. A working JS: take `li.msg-s-message-list__event` or `div[role="listitem"]` under the message panel, and use the final one's `innerText`.
- `timestamp` — `<time>`'s `datetime` attribute in the last bubble.

If zero threads, tell the user "No unread replies." and exit.

### Step 3 — Classify each thread

Classify `last_message` into exactly one category:

| Category | Signal |
|---|---|
| `interested` | "Yes, I'm open", "tell me more", "sounds interesting" |
| `wants_call` | Proposes / asks for a time to talk |
| `question` | Specific question (comp, location, stack, visa, WFH, etc.) |
| `negotiating` | Pushes back on comp, location, seniority |
| `not_interested` | Polite no, "not looking", "not a fit" |
| `other` | Anything unclear |

### Step 4 — Draft a reply per thread

Use these templates. Keep ≤120 words. Use `candidate_first_name` and the recruiter name from `config/recruiter.json`. **Never fabricate company specifics** (comp, team size, stack) — if asked something you don't have, say you'll follow up.

**`interested`**
```
Hi {First Name},

Great to hear! Could you share your updated resume and a couple of time slots that work for a quick intro call this week?

Best,
{Recruiter}
```

**`wants_call`**
```
Hi {First Name},

Happy to set up a call. {Mirror their proposed time, or: "Does <tomorrow / this week> work — mornings or afternoons?"} Please share your resume ahead of the call if you haven't already.

Best,
{Recruiter}
```

**`question`**
```
Hi {First Name},

Thanks for reaching out. {Answer the specific question ONLY if the answer is in the JD or recruiter config. Otherwise: "Let me confirm the specifics on <topic> and get back to you shortly."}

In the meantime, could you share your updated resume and a time that works for a quick call?

Best,
{Recruiter}
```

**`negotiating`**
```
Hi {First Name},

Thanks for being upfront. I'll need to check {what they pushed on} with the hiring team and get back to you. In the meantime, a quick call would help me understand what you're looking for — are you free {this week}?

Best,
{Recruiter}
```

**`not_interested`**
```
Hi {First Name},

Totally understand — thanks for letting me know. If anything changes or you'd like me to keep you in mind for future roles, just say the word.

Best,
{Recruiter}
```

**`other`** — draft a short, neutral acknowledgment and flag to the user that it needs a custom reply.

### Step 5 — Present drafts and ask for approval

Show a numbered list with each thread's category, candidate name, last-message excerpt, and the draft reply. Then ask:

> Reply "approve all", "approve 1,3,5", "skip 2", or edit any draft by saying "edit 2: <new text>". I'll send only the approved ones.

Do **not** send anything not explicitly approved.

### Step 6 — Send approved replies

For each approved draft:
1. `navigate` to the thread URL.
2. `find` the reply textbox (contenteditable or a textbox labeled "Reply"/"Message").
3. `form_input` the body.
4. `find` the Send button → click it.
5. Sleep 5–12s before the next send.

On failure, log and continue; report at the end.

### Step 7 — Report

```
Replies read:     <N>
Classified:       interested: X, wants_call: Y, question: Z, negotiating: A, not_interested: B, other: C
Approved sent:    <OK>
Failed:           <F>  (with reasons)
Skipped:          <SK>
```

## Hard rules

- **Never auto-send.** All replies require explicit user approval.
- **Never fabricate** comp, team details, stack, or commitments. When unknown, say "let me confirm and get back".
- If a reply contains a complaint, legal concern, or escalation, classify as `other` and flag prominently — do not draft a form reply.
- If an MCP tool errors or a critical selector is missing, stop and report — do not retry-loop.
