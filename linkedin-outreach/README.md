# linkedin-outreach

Claude Code plugin that automates LinkedIn Recruiter outreach: sources candidates from a JD, shortlists them, sends personalized InMails in batches of 50, and drafts responses to replies for your approval.

## Runtime

**No Node, no Python, no Playwright.** This plugin drives Chrome entirely through the [Claude in Chrome](https://claude.ai/chrome) extension's MCP tools. Setup is installing the extension once; the plugin has zero dependencies of its own.

## Install

1. **Install the Claude in Chrome extension** from https://claude.ai/chrome. Sign in and enable the MCP connection to Claude Code (the extension walks you through this).
2. **Clone / copy this plugin folder.** Nothing to build — it's just skill and command markdown files.
3. Run `/setup` inside Claude Code.

## First-time setup (`/setup`)

Runs a 3-step wizard:
1. Confirms `mcp__Claude_in_Chrome__*` tools are available.
2. Prompts for your recruiter name / title / company (written to `config/recruiter.json`).
3. Opens `https://www.linkedin.com/talent/home` in Chrome and waits for you to log in.

Your LinkedIn session lives in your normal Chrome profile, so it persists like any other site.

## Usage

- `/source <path-to-JD | pasted-JD>` — parses the JD, applies filters, shortlists up to 50 candidates, sends InMails, stops for your go-ahead before the next batch. First run of a JD is a dry-run (fills the message but doesn't click Send) so you can sanity-check the copy.
- `/replies` — reads your LinkedIn Recruiter inbox, classifies each reply, drafts responses, waits for your approval (`approve all`, `approve 1,3,5`, or edit) before sending.

## Safety

- Claude evaluates candidates **only from visible search-result snippets** — it does not open full profiles.
- InMail personalization lines cite only concrete snippet signals. If no signal is visible, the personalization line is **omitted** rather than fabricated.
- Replies are always **draft → you approve → send**. Never auto-sent.
- Outbound InMails are **fully auto** in batches of 50 (per your config), with a randomized 8–20s pause between sends. Plugin stops after 50 and asks whether to continue.
- If an MCP tool errors or a critical element (Message / Send button) can't be found, Claude stops the pipeline and reports — no retry loops.

## Files

```
.claude-plugin/plugin.json       plugin manifest
skills/source-and-inmail/        outbound flow SKILL.md
skills/handle-replies/           reply flow SKILL.md
commands/setup.md                /setup
commands/source.md               /source
commands/replies.md              /replies
config/recruiter.json            your signature vars (git-ignored)
```

## Known caveats

- LinkedIn rewrites its DOM occasionally — expect the SKILL.md JS snippets to need a touch-up ~1–2× per year.
- Automated interaction with LinkedIn is a gray area even on paid Recruiter tiers. Check your org's contract.
- Quality of shortlist depends on snippet richness; if search snippets are too sparse, Claude will reject more candidates than ideal (the "when unsure, skip" safety rule).
- Because Claude drives every click turn-by-turn, a full 50-InMail batch is slower and uses more context than a headless script would. The built-in 8–20s pause between sends is the dominant factor either way.
