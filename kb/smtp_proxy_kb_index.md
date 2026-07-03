# SMTP Proxy KB Index — Read This First

Do NOT read the full KB files. Use this table to load only what the PRD needs.

---

## Step 1 — Classify the PRD

| PRD describes... | Load |
|---|---|
| UI pages only (Alerts, Events, Incidents, Settings page, SkopeIT) | `smtp_proxy_kb_ui.md` only |
| API/protocol only (smtplib, SMTP commands, feature flags, logs, relay config) | `smtp_proxy_kb_api.md` only |
| Both UI and API | Both files |
| Unclear | Ask user before reading |

---

## Step 2 — Load only relevant sections within the file

### smtp_proxy_kb_api.md — section guide

| Feature area in PRD | Read sections |
|---|---|
| Feature flags (enable/disable, toggle, sync) | §3, §16 |
| Email relay config / MSA / domain routing | §4, §16 |
| RT policy creation | §5 |
| Sending emails / SMTP protocol / commands | §6, §15, §16 |
| Log verification / SMTPREQ / SMTPRES / DKIM logs | §7 |
| DKIM signing / canonicalization | §3.4, §6.7, §7.4 |
| Connection persistence (front/back conn) | §3.1, §3.2, §8, §9 |
| Large file support (LFS) | §3.2, §6.5, §15 |
| Test setup / teardown patterns | §8, §9 |
| Kubernetes / pod commands | §1, §3.4 |
| Test data files | §12 |
| File generator | §14 |
| Delivery verification | §13 |
| Placeholder variables | §16 (always read for any API test) |

**Minimum for any API test plan: §16 (placeholders) + the sections matching your PRD.**

### smtp_proxy_kb_ui.md — section guide

| Feature area in PRD | Read sections |
|---|---|
| SMTP Settings page (MSA config, Exchange/Gmail/Custom) | §17.1–17.9 |
| Record Subject Line feature | §17.9 |
| Source IP allow list / ipset | §17.7 |
| Alerts / Application Events / Incidents new fields | §18.1–18.10 |
| SkopeIT filter / column customization | §18.4–18.8 |
| Feature flag toggle for UI visibility | §18.9, §17.12 |
| Accessibility / keyboard navigation | §17.11 |
| Sending traffic to seed SkopeIT tables | §17.14 |

**Minimum for any UI test plan: §18.1–18.5 + the sections matching your PRD.**

---

## Step 3 — Always read regardless of PRD type

- `~/.claude/skills/smtp-testplan-generator/templates/testrail_format_reference.md` — CSV column spec + placeholder variables (essential for output)
- `~/.claude/skills/smtp-testplan-generator/templates/confluence_template.md` — Confluence page structure (essential for output)

**Do NOT read files in `examples/` at runtime. Format is fully captured in testrail_format_reference.md.**
