# SMTP Proxy KB Index — Read This First

Do NOT read the full KB files. Use this table to load only what the Design Spec needs.

---

## Step 1 — Classify the Design Spec

| Design Spec describes... | Load |
|---|---|
| UI pages only (Alerts, Events, Incidents, Settings page, SkopeIT) | `smtp_proxy_kb_ui.md` only |
| API/protocol only (smtplib, SMTP commands, feature flags, logs, relay config) | `smtp_proxy_kb_api.md` only |
| Both UI and API | Both files |
| Unclear | Ask user before reading |

---

## Step 2 — Load only relevant sections within the file

### smtp_proxy_kb_api.md — section guide

| Feature area in Design Spec | Read sections |
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

**Minimum for any API test plan: §16 (placeholders) + the sections matching your Design Spec.**

### smtp_proxy_kb_ui.md — section guide

| Feature area in Design Spec | Read sections |
|---|---|
| SMTP Settings page (MSA config, Exchange/Gmail/Custom) | §17.1–17.9 |
| Record Subject Line feature | §17.9 |
| Source IP allow list / ipset | §17.7 |
| Alerts / Application Events / Incidents new fields | §18.1–18.10 |
| SkopeIT filter / column customization | §18.4–18.8 |
| Feature flag toggle for UI visibility | §18.9, §17.12 |
| Accessibility / keyboard navigation | §17.11 |
| Sending traffic to seed SkopeIT tables | §17.14 |

**Minimum for any UI test plan: §18.1–18.5 + the sections matching your Design Spec.**

---

## Step 2b — Line-number map (for targeted offset/limit reads)

Use these line ranges with the Read tool's `offset`/`limit` params instead of reading the
whole KB file. Add ~3 lines of buffer on either end; slight over-read is fine and still far
cheaper than a full-file read.

**Fallback rule:** if the sections you need cover more than ~60% of a file's total lines
(444 for api, 330 for ui), just read the whole file — at that point separate ranged reads
cost more in overhead than one pass.

### smtp_proxy_kb_api.md (444 lines)

| Section | Lines |
|---|---|
| §1 Infrastructure Overview | 8–34 |
| §2 SMTP Proxy VIP | 36–43 |
| §3 Feature Flags (all) | 45–135 |
| §3.1 Tenant-Scoped Feature Flags | 47–63 |
| §3.2 Feature Flag Names | 97–106 |
| §3.3 Staged Config Flags | 108–122 |
| §3.4 DKIM Feature Flag | 124–134 |
| §4 Email Relay Config | 138–174 |
| §5 Real-Time Policy | 177–184 |
| §6 Sending Emails (all) | 188–257 |
| §6.1 Standard Send | 190–200 |
| §6.2 EHLO/HELO | 202–206 |
| §6.3 Raw SMTP Commands | 208–216 |
| §6.4 RSET/NOOP | 218–222 |
| §6.5 SMTP Exceptions | 224–231 |
| §6.6 EmailBuilder | 233–242 |
| §6.7 DKIM-Signed Email | 244–256 |
| §7 Log Verification (all) | 261–307 |
| §7.1 Fetching Logs | 263–267 |
| §7.2 SMTPREQ Pattern | 269–276 |
| §7.3 SMTPRES Pattern | 278–285 |
| §7.4 DKIM Log Patterns | 287–295 |
| §7.5 Log Parser/Verifier | 297–304 |
| §7.6 Error Code 558 | 306–307 |
| §8 Test Setup Pattern | 311–324 |
| §9 Test Teardown Pattern | 326–336 |
| §10 Test File Organization | 340–358 |
| §11 Test Markers | 361–370 |
| §12 Test Data Files | 374–377 |
| §13 Downstream Mail Server | 381–393 |
| §14 File Generator | 396–402 |
| §15 Known SMTP Response Codes | 406–422 |
| §16 Common Placeholder Variables | 426–444 |

### smtp_proxy_kb_ui.md (330 lines)

| Section | Lines |
|---|---|
| §17 SMTP Settings Page Tests (all) | 8–204 |
| §17.1 Page Object and Navigation | 12–27 |
| §17.2 Nav Bar Assertions | 29–43 |
| §17.3 MSA Cards | 45–56 |
| §17.4 Exchange MSA Edit Modal | 58–77 |
| §17.5 Custom MSA Backend Helpers | 79–89 |
| §17.6 smtpSettings.json Verification | 91–107 |
| §17.7 Source IP Allow List | 109–125 |
| §17.8 Domain Validation Rules | 127–136 |
| §17.9 Record Subject Line Feature | 138–148 |
| §17.10 SkopeIT Subject Line Integration | 150–162 |
| §17.11 Accessibility Tests | 164–178 |
| §17.12 UI Fixtures (feature flag provisioner) | 180–186 |
| §17.13 UI Input Data Files | 188–194 |
| §17.14 Generate SMTP Traffic for UI Tests | 196–204 |
| §18 Alerts/Events/Incidents (all) | 208–330 |
| §18.1 Page URLs and Navigation | 212–226 |
| §18.2 Alerts Table Columns | 228–231 |
| §18.3 Application Events Table Columns | 233–236 |
| §18.4 Feature-Flag-Controlled Column Pattern | 238–242 |
| §18.5 Side Panel Structure | 244–267 |
| §18.6 Conditional Field Display Pattern | 269–279 |
| §18.7 Filter Pattern | 281–294 |
| §18.8 Column Customize Dialog | 296–302 |
| §18.9 Feature Flag Toggle for UI Column Visibility | 304–315 |
| §18.10 Accessing Side Panel Fields by Label | 317–330 |

> If the KB files are ever edited, these line numbers go stale — re-derive them (grep for
> `^##` / `^###` headers) before trusting this table again.

---

## Step 3 — Always read regardless of Design Spec type

- `~/.claude/skills/smtp-testplan-generator/templates/testrail_format_reference.md` — CSV column spec + placeholder variables (essential for output)
- `~/.claude/skills/smtp-testplan-generator/templates/confluence_template.md` — Confluence page structure (essential for output)

**Do NOT read files in `examples/` at runtime. Format is fully captured in testrail_format_reference.md.**
