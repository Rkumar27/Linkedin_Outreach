---
description: Source + shortlist + send InMails from a Job Description. Usage: /source <path-to-JD-file | pasted JD text>
---

Invoke the `source-and-inmail` skill at `/Users/rohitkumar/Documents/Claud Plugin/linkedin-outreach/skills/source-and-inmail/SKILL.md` and follow it end-to-end.

Input: `$ARGUMENTS` — either a file path to read, or the JD text pasted inline. If it's a path, read the file. If it's pasted text, use it directly.

Before starting Step 1, verify `config/recruiter.json` has a non-empty `name` field. If not, tell the user to run `/setup` first and stop.
