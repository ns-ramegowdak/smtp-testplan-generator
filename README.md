# smtp-testplan-generator

A Claude Code skill that generates complete, TestRail-ready test plans for SMTP Proxy features directly from a Design Spec.

## What it produces

Every run writes a TestRail CSV, plus a QE test-plan page in the team's real Confluence template — either published live or written locally, your choice:

| Output | Use |
|---|---|
| `smtp_testplan/{TICKET_ID}_{feature_name}_testrail.csv` | Import directly into TestRail via "Import Test Cases → CSV" |
| Confluence page (matches the team's `ENG`-space QE test-plan format) | Either **published live** to a Confluence space via the Atlassian MCP connection, or written locally to `smtp_testplan/{TICKET_ID}_{feature_name}_confluence.md` for you to paste in yourself |

## How to invoke

Type `/smtp-testplan-generator` in Claude Code. The skill will ask for:

1. Jira ticket ID (e.g. `QE-83414`) — can be skipped if item 4 is a Jira URL for the same ticket
2. Feature name in snake_case (e.g. `machine_generated_email`)
3. Test scope — UI only / API only / Both
4. Design Spec source — a Jira issue URL/key or a Confluence page URL (fetched automatically via the Atlassian MCP connection), or pasted content if you'd rather not use MCP
5. Your name (used as QE Owner / QE Members in TestRail and Confluence)
6. (Optional) Test focus — areas to emphasize or deprioritize
7. Whether the Confluence output should be published live via MCP or written locally as a `.md` file
8. If publishing live: the target Confluence space, plus optional metadata (parent page, QE Epic Link, Release Schedule, Dev Members, Test Rail Link, Testing Estimate)

If item 4 is a URL, the skill fetches the Design Spec directly — no copy-pasting needed. Jira tickets that link out to a Confluence design doc are detected and offered as an additional fetch. If the MCP fetch fails or isn't available, it falls back to asking you to paste the content.

The skill walks you through several confirmation checkpoints before writing anything:
- What it fetched (Phase 0.5) — confirm it grabbed the right Design Spec
- Requirements summary (Phase 1) — verify extraction from Design Spec
- Test case list (Phase 3) — add/remove/reprioritize before output
- Confluence space/parent (Phase 4, if publishing live) — confirm before creating a page in a shared space
- Output summary (Phase 4) — file paths / page URL + counts

## Prerequisites

- Claude Code CLI installed and authenticated
- Invoke from any directory — the skill is installed globally at `~/.claude/skills/`
- Atlassian MCP connection (optional but recommended) — needed to fetch Design Specs by URL/key and to publish the Confluence page live. Without it, fall back to pasting the Design Spec content and writing the Confluence output locally.

## Skill layout

```
~/.claude/skills/smtp-testplan-generator/
├── SKILL.md               ← skill definition (loaded by Claude Code)
├── README.md              ← this file
├── kb/
│   ├── smtp_proxy_kb_index.md    ← read first every run (routing table)
│   ├── smtp_proxy_kb_api.md      ← API/backend patterns (smtplib, kubectl, logs)
│   └── smtp_proxy_kb_ui.md       ← UI layer patterns (Selenium page objects, Alerts/Events)
├── templates/
│   ├── testrail_format_reference.md   ← CSV column spec + placeholder variables
│   └── confluence_template.md         ← Confluence page section skeleton
└── examples/
    ├── machine_generated_emails_testrail.csv    ← sample output (reference only)
    └── machine_generated_emails_confluence.md   ← sample output (reference only)
```

Generated test plans go to `smtp_testplan/` inside your **current working directory** (not inside the skill directory).

## Design principles

- **Design Spec-only scope** — test cases come strictly from what the Design Spec states; nothing inferred
- **Token-efficient** — KB index read first; only relevant sections loaded per Design Spec type
- **Executable steps** — every step names exact Python methods, kubectl commands, or smtplib calls
- **Placeholder variables** — no hardcoded IPs, tenant IDs, or pod names; all use `<PLACEHOLDER>` tokens from `testrail_format_reference.md`

---

## How to update the KB

The KB captures reusable test patterns. Update it when you encounter something new that will apply to future features — not feature-specific details (those belong only in the generated test plan).

### When to update

| You encountered... | Update this file |
|---|---|
| A new Selenium page object or method for Alerts/Events/Settings page | `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_ui.md` |
| A new smtplib pattern, kubectl command, or log format | `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_api.md` |
| A new feature flag API or SMTP API endpoint | `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_api.md` |
| A new page added to SkopeIT (e.g. new Incidents sub-page) | `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_ui.md` §18 |
| A new section type not covered by existing index entries | `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_index.md` |

### What NOT to add to the KB

- Feature-specific flag names, field names, or value lists — these belong in the test plan only
- Hardcoded paths or personal environment values
- Content already in `testrail_format_reference.md` or `confluence_template.md`

### How to add a new KB section

1. Add the section to the appropriate KB file (`smtp_proxy_kb_api.md` or `smtp_proxy_kb_ui.md`) with a numbered heading (e.g. `## 19. New Feature Pattern`)
2. Add a row to `smtp_proxy_kb_index.md` Step 2 section guide mapping the feature area to the new section number
3. Test it: invoke the skill with a Design Spec that would need the new section and verify the skill reads it

### How to update an existing pattern

Edit the relevant section directly. KB files are plain Markdown — no special format required beyond the section numbering convention.
