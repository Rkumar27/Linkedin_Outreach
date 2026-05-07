---
description: First-time setup for the linkedin-outreach plugin — checks Claude-in-Chrome connection, captures recruiter signature, and verifies LinkedIn Recruiter login.
---

Run the one-time setup for the `linkedin-outreach` plugin.

Plugin root: `/Users/rohitkumar/Documents/Claud Plugin/linkedin-outreach`

### Step 1 — Verify Claude in Chrome extension

Check whether `mcp__Claude_in_Chrome__*` tools are available in this session. If none are available, tell the user:

> This plugin drives Chrome via the **Claude in Chrome** extension. It doesn't appear to be installed or connected.
>
> 1. Install the extension: https://claude.ai/chrome
> 2. Open the extension in Chrome and sign in with your Claude account.
> 3. Enable the MCP connection to Claude Code (follow the extension's onboarding).
> 4. Restart Claude Code and re-run `/setup`.

Stop and wait for them to confirm before moving on.

### Step 2 — Capture recruiter identity

Ask the user for:
- Full name (used as `{Recruiter Name}` in InMail signatures)
- Title (optional, e.g., "Senior Technical Recruiter")
- Company (optional)

Write to `config/recruiter.json`:
```json
{ "name": "...", "title": "...", "company": "..." }
```

### Step 3 — Verify LinkedIn Recruiter login

Use `mcp__Claude_in_Chrome__navigate` to open `https://www.linkedin.com/talent/home`.

Inspect the resulting URL (via `read_page` or `get_page_text`):
- If the URL contains `/login`, `/uas/login`, or `/checkpoint` → tell the user:
  > A LinkedIn login page is open in Chrome. Please sign in to LinkedIn Recruiter there, then reply "done" here.
  After they reply, `navigate` back to `https://www.linkedin.com/talent/home` and re-check.
- Otherwise → confirm login succeeded.

### Step 4 — Confirm

Print:
```
linkedin-outreach is ready.
  Recruiter: <name>
  Runtime:   Claude in Chrome (no local deps)
Next steps:
  /source <JD>   — shortlist + send InMails
  /replies       — draft replies to recent InMail responses
```
