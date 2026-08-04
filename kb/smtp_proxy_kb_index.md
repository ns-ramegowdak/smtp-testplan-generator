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
| Sending via a specific MSA type (Gmail/O365 Graph API/Postfix/Custom) | §16 |
| Async DLP / "SnowyOwl" (job-based scanning, Ceph upload, fallback-on-error) | §17 |
| Outlook Plugin / REST email ingestion (port 8000, EPDLP gateway) | §18 |
| Health check service (`/status`, `/health`, rate limiting) | §19 |
| DKIM negative/tampering tests (broken signature, degradation) | §20, §3.4, §6.7, §7.4 |
| Machine-generated email / BOT detection | §21 |
| Placeholder variables | §22 (always read for any API test) |

**Minimum for any API test plan: §22 (placeholders) + the sections matching your Design Spec.**

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
| Custom Tenant Identification settings page (NPLAN-6151 WebUI) | §23.1 |
| DNS domain validation UI (classic and NGWEB) | §23.2 |
| Self-addressed email on the Incidents page | §23.3 |
| Machine-generated email in SkopeIT | §23.4 |
| Combined/multi-feature regression scenarios (UI) | §23.5 |
| SMTP Settings page old-modal/new-modal migration state | §23.6 |

**Minimum for any UI test plan: §18.1–18.5 + the sections matching your Design Spec.** Also skim
§23 if the Design Spec touches Tenant Identification, Domain Validation, Self-Addressed Detection,
Machine-Generated Email, or the Settings page — several of these areas have UI test coverage that
isn't obvious from §17/§18 alone.

---

## Step 2b — Line-number map (for targeted offset/limit reads)

Use these line ranges with the Read tool's `offset`/`limit` params instead of reading the
whole KB file. Add ~3 lines of buffer on either end; slight over-read is fine and still far
cheaper than a full-file read.

**Fallback rule:** if the sections you need cover more than ~60% of a file's total lines
(686 for api, 405 for ui), just read the whole file — at that point separate ranged reads
cost more in overhead than one pass.

*(Last re-derived 2026-08-04 from actual `^##`/`^###` header positions — see the warning
below; it had drifted by up to 64 lines after §3.3 in the api file before this refresh.)*

### smtp_proxy_kb_api.md (686 lines)

| Section | Lines |
|---|---|
| §1 Infrastructure Overview | 8–35 |
| §2 SMTP Proxy VIP | 36–44 |
| §3 Feature Flags (all) | 45–160 |
| §3.1 Tenant-Scoped Feature Flags | 47–96 |
| §3.2 Feature Flag Names | 97–107 |
| §3.3 Staged Config Flags | 108–146 |
| §3.4 DKIM Feature Flag | 147–160 |
| §4 Email Relay Config | 161–199 |
| §5 Real-Time Policy | 200–210 |
| §6 Sending Emails (all) | 211–283 |
| §6.1 Standard Send | 213–224 |
| §6.2 EHLO/HELO | 225–230 |
| §6.3 Raw SMTP Commands | 231–240 |
| §6.4 RSET/NOOP | 241–246 |
| §6.5 SMTP Exceptions | 247–255 |
| §6.6 EmailBuilder | 256–266 |
| §6.7 DKIM-Signed Email | 267–283 |
| §7 Log Verification (all) | 284–340 |
| §7.1 Fetching Logs | 286–291 |
| §7.2 SMTPREQ Pattern | 292–300 |
| §7.3 SMTPRES Pattern | 301–309 |
| §7.4 DKIM Log Patterns | 310–319 |
| §7.5 Log Parser/Verifier | 320–335 |
| §7.6 Error Code 558 | 336–340 |
| §8 Test Setup Pattern | 341–355 |
| §9 Test Teardown Pattern | 356–369 |
| §10 Test File Organization | 370–406 |
| §11 Test Markers | 407–419 |
| §12 Test Data Files | 420–426 |
| §13 Downstream Mail Server | 427–441 |
| §14 File Generator | 442–451 |
| §15 Known SMTP Response Codes | 452–471 |
| §16 Unified Email Sender Interface | 472–502 |
| §17 Async DLP ("SnowyOwl") Test Utilities (all) | 503–554 |
| §17.1 Async DLP phases, error codes, and Prism metrics | 537–554 |
| §18 Outlook Plugin REST API Testing Pattern (all) | 555–619 |
| §18.1 Deferred event payload (`defer_event_gen`) response schema | 597–619 |
| §19 Health Check Testing Pattern | 620–640 |
| §20 DKIM Tampering Utilities | 641–653 |
| §21 Machine-Generated Email Detection | 654–667 |
| §22 Common Placeholder Variables | 668–686 |

### smtp_proxy_kb_ui.md (405 lines)

| Section | Lines |
|---|---|
| §17 SMTP Settings Page Tests (all) | 8–215 |
| §17.1 Page Object and Navigation | 12–28 |
| §17.2 Nav Bar Assertions | 29–44 |
| §17.3 MSA Cards | 45–57 |
| §17.4 Exchange MSA Edit Modal | 58–78 |
| §17.5 Custom MSA Backend Helpers | 79–90 |
| §17.6 smtpSettings.json Verification | 91–108 |
| §17.7 Source IP Allow List | 109–129 |
| §17.8 Domain Validation Rules | 130–140 |
| §17.9 Record Subject Line Feature | 141–152 |
| §17.10 SkopeIT Subject Line Integration | 153–166 |
| §17.11 Accessibility Tests | 167–182 |
| §17.12 UI Fixtures (feature flag provisioner) | 183–195 |
| §17.13 UI Input Data Files | 196–203 |
| §17.14 Generate SMTP Traffic for UI Tests | 204–215 |
| §18 Alerts/Events/Incidents (all) | 216–341 |
| §18.1 Page URLs and Navigation | 220–235 |
| §18.2 Alerts Table Columns | 236–240 |
| §18.3 Application Events Table Columns | 241–245 |
| §18.4 Feature-Flag-Controlled Column Pattern | 246–251 |
| §18.5 Side Panel Structure | 252–276 |
| §18.6 Conditional Field Display Pattern | 277–288 |
| §18.7 Filter Pattern | 289–303 |
| §18.8 Column Customize Dialog | 304–311 |
| §18.9 Feature Flag Toggle for UI Column Visibility | 312–324 |
| §18.10 Accessing Side Panel Fields by Label | 325–341 |
| §23 Additional Page Objects & UI Test Areas (all) | 342–405 |
| §23.1 Custom Tenant Identification settings page | 357–368 |
| §23.2 DNS Domain Validation UI (two generations) | 369–376 |
| §23.3 Self-Addressed Email on the Incidents page | 377–382 |
| §23.4 Machine-Generated Email in SkopeIT | 383–388 |
| §23.5 RTP Balkan combined-scenario suite | 389–396 |
| §23.6 SMTP Settings page — old modal vs new modal | 397–405 |

> If the KB files are ever edited, these line numbers go stale — re-derive them (grep for
> `^##` / `^###` headers) before trusting this table again. **This has happened for real:**
> two prior commits added §16–§22 and §17.1/§18.1 to `smtp_proxy_kb_api.md` without this table
> being refreshed, drifting by up to 64 lines by §19–§22 before the 2026-08-04 fix — silently
> feeding wrong/partial KB content into targeted reads with no visible error. Whenever you add
> or resize a KB section (see the README's "How to add a new KB section"), re-derive this whole
> table in the same edit, not just the Step 2 section-guide row.

---

## Step 3 — Always read regardless of Design Spec type

- `~/.claude/skills/smtp-testplan-generator/templates/testrail_format_reference.md` — CSV column spec + placeholder variables (essential for output)
- `~/.claude/skills/smtp-testplan-generator/templates/confluence_template.md` — Confluence page structure (essential for output)

**Do NOT read files in `examples/` at runtime. Format is fully captured in testrail_format_reference.md.**
