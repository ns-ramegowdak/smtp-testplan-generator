---
name: smtp-testplan-generator
description: >
  Generate a complete test plan for SMTP Proxy features from a PRD (Jira or Confluence content).
  Produces a TestRail-ready CSV and a Confluence-ready Markdown page.
  Strictly PRD-driven — only covers what the PRD describes, nothing more.
  Use when: generating SMTP test plan, creating test cases from PRD, smtp testplan, smtp test coverage,
  smtp test cases from requirements, testrail csv smtp, smtp proxy testing plan.
triggers:
  - smtp-testplan-generator
  - smtp testplan
  - generate smtp test plan
  - smtp test cases from prd
  - smtp testrail csv
  - smtp proxy test coverage
  - create test plan smtp
---

# SMTP Proxy Test Plan Generator

## Purpose

Generate a complete, ready-to-import test plan for any SMTP Proxy feature given its PRD content.

**Two outputs every run:**
1. TestRail CSV — `smtp_testplan/{TICKET_ID}_{feature_name}_testrail.csv`
2. Confluence Markdown page — `smtp_testplan/{TICKET_ID}_{feature_name}_confluence.md`

**Core constraint:** Test cases are generated STRICTLY from the PRD content provided. Do not add
test cases for behaviors, configurations, or edge cases not described in the PRD. If a behavior
seems implied but is not stated, call it out in Section 9 (Open Questions) instead.

---

## Phase 0 — Input Collection

Before doing anything else, ask the user for the following. All fields are required except item 6.

```
smtp-testplan-generator needs a few details:

1. Jira ticket ID  (e.g.: QE-83414, NPLAN-4895)

2. Feature name (snake_case, used in output filenames)
   e.g.: dkim_verification, envelope_testing, lfs_size_enforcement

3. Test scope:
   a) UI only  (Alerts/Events/Incidents pages, SMTP Settings page)
   b) API only  (SMTP protocol, feature flags, relay config, logs, smtplib)
   c) Both UI and API

4. PRD content — paste the full text from your Jira ticket or Confluence page.
   Include: feature description, acceptance criteria, error handling requirements,
   security requirements, any listed edge cases or notes.

5. QE Owner name (your name — used in TestRail CSV)

6. (Optional) Test focus — any areas to emphasize or deprioritize?
   e.g.: "Emphasize boundary cases for the size limit", "Deprioritize export tests"
   Press Enter to skip.
```

Wait for all inputs before proceeding. Do not begin analysis until the user has provided the PRD content.

After receiving inputs:
- Set `TICKET_ID` = Jira ticket ID provided
- Set `FEATURE_NAME` = feature name provided
- Set `TEST_SCOPE` = `UI` / `API` / `Both` based on answer to item 3
- Set `QE_OWNER` = name provided
- Set `PRD_SOURCE` = `Jira` / `Confluence` / `Manual` based on the pasted content origin (ask if unclear)
- Set `TEST_FOCUS` = emphasis/deprioritize notes from item 6, or empty if skipped

---

## Phase 1 — PRD Analysis

Read the PRD content and extract the following. Work entirely from what is stated — no inference.

### 1.1 Requirements Extraction

Extract into these categories (build internally — do NOT print the full map):

**A. Behaviors** `[B-N]` — What the feature must do (positive requirements)
**B. Error conditions** `[E-N]` — What must be rejected/blocked + expected response/action
**C. Security requirements** `[S-N]` — TLS, DKIM, auth, injection prevention
**D. Constraints** `[C-N]` — Limits, timeouts, size thresholds, count limits (with exact values)
**E. Feature flags** `[F-N]` — Named flags that gate the feature + default state
**F. Ambiguities** `[Q-N]` — Unclear items or missing details needed for testing

Print a **compact summary** to the conversation (not the full map):

```
Requirements summary — {FEATURE_NAME}
Behaviors: N  |  Errors: N  |  Constraints: N  |  Flags: N  |  Questions: N

Key behaviors: [B-1] ..., [B-2] ..., [B-3] ...
Key constraints: [C-1] ..., [C-2] ...
Open questions: [Q-1] ..., [Q-2] ...
```

Ask: "Does this look right? Any additions or corrections before I generate tests?"
Wait for confirmation, then proceed.

---

## Phase 2 — Test Design

Apply ISTQB techniques to each requirement.

### 2.1 Technique Mapping Rules

| Technique | Apply to requirement types | Category tag |
|---|---|---|
| **Equivalence Partitioning (EP)** | Behaviors with valid/invalid input ranges | `[POS]` valid partition / `[NEG]` invalid partition |
| **Boundary Value Analysis (BVA)** | Constraints [C-N]: test AT the limit, one below, one above | `[BND]` |
| **Decision Table (DT)** | Feature flag combinations, mode × behavior combinations | `[POS]` happy-path rows / `[NEG]` error rows |
| **State Transition (ST)** | SMTP session states, connection lifecycle | `[POS]` success transitions / `[NEG]` invalid transitions |
| **Use Case Testing (UC)** | End-to-end happy-path flows from acceptance criteria | `[POS]` |
| **Error Guessing (EG)** | Error conditions [E-N] | `[NEG]` |
| **Error Guessing (EG)** | Security requirements [S-N] | `[SEC]` |

### 2.2 Priority Assignment Rules

| Priority | Assign when |
|---|---|
| P0 | Core feature requirement; failure means feature is broken or insecure |
| P1 | Important but failure degrades correctness without breaking the whole feature |
| P2 | Edge case, secondary flow, or low-probability failure mode |
| P3 | Cosmetic or near-zero impact — usually omit unless PRD explicitly requires |

**Default:** Explicit PRD acceptance criterion → P0. Described behavior without AC → P1. Edge case/note → P2.

**If `TEST_FOCUS` is set:** Apply it as a weighting override after initial priority assignment.
- "Emphasize X" → bump requirement IDs related to X up one priority level (P1 → P0, P2 → P1) and generate additional test cases for those requirements if coverage is thin.
- "Deprioritize X" or "Skip X" → drop related cases to P2 or move them to Open Questions if they are not PRD acceptance criteria.

### 2.3 Automatable Decision Rules

**Yes** when ALL true: driven via smtplib/kubectl/SSH/REST or Selenium page objects; no human judgment; no tcpdump; no external DNS control.

**No** when ANY true: external DNS server control; iptables injection on pod; tcpdump/pcap; /etc/resolv.conf modification as test mechanism; manual timing coordination.

### 2.4 Test Case Generation Rules

For each requirement: apply technique → determine test count → prune duplicates.
- If two tests verify the same assertion, keep the higher-priority one.
- If a behavior is a subset of another test's steps, absorb it.
- Never generate a test that is purely "feature X exists" without verifiable assertions.

Assign a category tag to each test based on the technique and outcome:
- `[POS]` — UC, DT/ST happy-path, EP valid partition
- `[NEG]` — EG errors [E-N], EP invalid partition, DT/ST error rows
- `[BND]` — any BVA test (at/below/above limit)
- `[SEC]` — EG from security requirements [S-N]

Prepend the tag to the test Summary: `[POS] Verify ...`, `[NEG] Verify ...`.

---

## Phase 3 — Draft Test Cases

Build each test case internally with category tags assigned per §2.4.

### 3.1 Self-check (run silently before printing the draft)

**Step A — Dedup check (run first):** Identify test cases that differ only by a parameter value — e.g. TC-A tests "Confidence=Low", TC-B tests "Confidence=Medium", TC-C tests "Confidence=High" with identical steps otherwise. Merge them into a single parameterized case that covers all variants in the steps. In the draft table, note merged cases with `(merged: N variants)` after the summary.

**Step B — Coverage gap check (run after dedup):** For every item in the requirements map:
- `[B-N]`, `[E-N]`, `[C-N]`, `[S-N]` — verify at least one test case covers it. If any ID has zero coverage: add a test, or add it to Open Questions with a one-line justification.
- `[F-N]` (feature flags) — coverage is satisfied by Decision Table tests on the behaviors the flag gates. If no DT test covers a flag's on/off states, add one. No standalone flag test is needed if DT coverage exists.

**Step C — Quality score (0–100):**

| Criterion | Points |
|---|---|
| Every [B-N] has ≥ 1 test | 25 |
| Every [C-N] has a BVA test (at/below/above limit). If no [C-N] items exist, criterion passes (25 pts). | 25 |
| Every [E-N] has a [NEG] test | 25 |
| No test has vague steps (e.g. "verify behavior" without an assertion) | 25 |

Compute the score. If score < 75: fix gaps silently (add missing tests, sharpen vague steps), then re-score. Do not print intermediate scoring attempts.

### 3.2 Draft table

Print compact summary with tags and final score:

```
Test Plan Draft — {FEATURE_NAME}  (Quality score: N/100)
TC-1  P0  Functional   [POS] Verify ...
TC-2  P0  Security     [SEC] Verify ...
TC-3  P1  Negative     [NEG] Verify ...
TC-4  P1  Boundary     [BND] Verify ...
...
Total: N  |  Auto: X/N  |  P0: A  P1: B  P2: C
Coverage: [POS]: A  [NEG]: B  [BND]: C  [SEC]: D
```

Ask: "Does this list look complete? Add, remove, or reprioritize anything?"
Wait for confirmation before generating output files.

---

## Phase 4 — Output Generation

### Step 1: Load reference files (index-first, targeted)

**1a. Read the KB index first:**
`~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_index.md`

**1b. Based on `TEST_SCOPE`, read only the relevant KB file(s):**
- `TEST_SCOPE = UI`   → read `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_ui.md`
- `TEST_SCOPE = API`  → read `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_api.md`
- `TEST_SCOPE = Both` → read both files

Within each KB file, read only the sections the index identifies as relevant for your PRD's feature areas. Skip sections that have no bearing on the PRD.

**1c. Always read these two files regardless of scope:**
- `~/.claude/skills/smtp-testplan-generator/templates/testrail_format_reference.md`
- `~/.claude/skills/smtp-testplan-generator/templates/confluence_template.md`

**Do NOT read files in `examples/` at runtime** — format is fully captured in `testrail_format_reference.md`.

Use the KB to write concrete, executable steps — exact Python method names, exact kubectl command strings, exact smtplib call patterns, exact log grep strings. Replace all `<PLACEHOLDER>` tokens with KB-sourced values where the value is constant (e.g. port 25, namespace `smtpproxyv2`). Keep placeholders only for values that vary per deployment (VIP, tenant ID, pod name).

### Step 2: Generate TestRail CSV

Before writing any file, create the output directory if it does not exist:
`mkdir -p smtp_testplan`

File path: `smtp_testplan/{TICKET_ID}_{FEATURE_NAME}_testrail.csv`

Rules:
- Row 1: exact header from `testrail_format_reference.md` (13 columns)
- One row per test case
- Use placeholder variables from `testrail_format_reference.md` for all environment values
- Steps column: multi-line, wrapped in double quotes, numbered list starting with `Automation Steps:` or `MANUAL STEPS:`
- Expected Result column: multi-line, wrapped in double quotes, bullet list starting with `-`
- Fixed columns: `Component=SMTP Proxy`, `Automated=No`, `UI Case=No`, `Result=`, `Label=ai_generated`
- `Suggested by Dev=No` unless PRD explicitly attributes a test to a dev suggestion
- `QE Owner={QE_OWNER}`

### Step 3: Generate Confluence Markdown

File path: `smtp_testplan/{TICKET_ID}_{FEATURE_NAME}_confluence.md`

Use `confluence_template.md` as the structural skeleton. Fill in every `{placeholder}`:

| Placeholder | Source |
|---|---|
| `{FEATURE_NAME}` | User input |
| `{QE_OWNER}` | User input |
| `{DATE}` | Current date |
| `{PRD_SOURCE}` | Jira / Confluence / Manual |
| Overview paragraph | Summarize PRD in 2–4 sentences |
| In Scope bullets | Behaviors [B-N] + Error conditions [E-N] from requirements map |
| Out of Scope bullets | Adjacent SMTP features not in PRD + explicit PRD exclusions |
| Coverage Matrix rows | One row per PRD requirement area; mark ✅ / ➖ / ❌ per test type |
| Coverage totals | Count from Phase 3 draft |
| Test Cases summary table | One row per test case: TC-N, Summary, Category, Priority, Automatable |
| Risks table | Derive from ambiguities [Q-N] and any stated constraints |
| Open Questions | List [Q-N] items from requirements map |

Do NOT invent content for any section — if a section has no PRD basis, write `(not specified in PRD)`.

### Step 4: Confirm output

After both files are written, print:

```
Output files written:
  CSV:        smtp_testplan/{TICKET_ID}_{FEATURE_NAME}_testrail.csv
  Confluence: smtp_testplan/{TICKET_ID}_{FEATURE_NAME}_confluence.md

Summary:
  Total test cases: N
  P0: A  |  P1: B  |  P2: C  |  P3: D
  Automatable: X/N

Next steps:
  1. Review the Confluence page and paste into your Confluence space
  2. Import the CSV into TestRail (use "Import Test Cases" → CSV)
  3. Review Open Questions in Section 9 with Dev/PM before finalizing
```

---

## Quality Rules (apply throughout all phases)

1. **PRD-only scope** — If a behavior is not in the PRD, do not test it. If it seems important, add it to Open Questions.
2. **No duplicate assertions** — Two tests that assert the same outcome for the same condition are one test.
3. **Teardown in every test** — Every test that creates a relay config, RT policy, or feature flag change must include teardown in the last step(s).
4. **Placeholder variables only** — Never hardcode IP addresses, pod names, tenant IDs, or email addresses. Use placeholders from `testrail_format_reference.md`.
5. **Exact SMTP codes** — When the PRD specifies a response code, use it exactly. When the PRD says "error", use the correct RFC 5321 code and note the basis.
6. **Steps must be executable** — Each step must be a concrete action an automation engineer can implement directly. Avoid vague steps like "verify behavior" — say `assert response_code == 250`.
   - **Forbidden words in steps:** "correctly", "properly", "as expected", "should work", "the button", "equivalent", "appropriate", "or similar".
   - **UI element references** must use the exact visible label, placeholder, or aria-label as it appears in the product — never a description like "the save button".
   - **API steps** must spell out the exact endpoint path from the PRD — never a guessed or constructed path. If a path is absent from the PRD, write `<path not in PRD>` and flag it in Open Questions.
   - **Forbidden words in expected results:** "correctly", "properly", "as expected", "should work". Every assertion must be independently verifiable: HTTP status code, exact UI text, element visibility state, API response field + value, log entry content.
7. **Confirmation before output** — Always get user confirmation on the requirements summary (Phase 1) and test list (Phase 3) before writing files.

---

## Reference Files

| File | Purpose |
|---|---|
| `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_index.md` | **Read first** — decision table for which KB file/sections to load |
| `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_ui.md` | UI layer KB: SMTP Settings page, Alerts/Events/Incidents patterns |
| `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_api.md` | API/backend KB: smtplib, kubectl, feature flags, logs, relay config |
| `~/.claude/skills/smtp-testplan-generator/templates/testrail_format_reference.md` | CSV column spec, placeholder vars, step/expected format rules |
| `~/.claude/skills/smtp-testplan-generator/templates/confluence_template.md` | Confluence page section skeleton |
| `~/.claude/skills/smtp-testplan-generator/examples/` | Sample outputs for reference only — do NOT read at runtime |

> Always read the KB index first. Never read example files at runtime.

---

## Skill Layout

```
~/.claude/skills/smtp-testplan-generator/
├── SKILL.md                          ← this file
├── kb/
│   ├── smtp_proxy_kb_index.md        ← read first every run
│   ├── smtp_proxy_kb_api.md          ← API/backend patterns
│   └── smtp_proxy_kb_ui.md           ← UI layer patterns
├── templates/
│   ├── testrail_format_reference.md  ← always read
│   └── confluence_template.md        ← always read
└── examples/
    ├── machine_generated_emails_testrail.csv   ← reference only
    └── machine_generated_emails_confluence.md  ← reference only
```

Output files are written to `smtp_testplan/` inside the current working directory. The directory is created with `mkdir -p smtp_testplan` before the first write if it does not exist. Files are named `{TICKET_ID}_{FEATURE_NAME}_testrail.csv` and `{TICKET_ID}_{FEATURE_NAME}_confluence.md`.

---

## Example Invocation

**User types:**
```
/smtp-testplan-generator
```

**Skill asks:**
```
smtp-testplan-generator needs a few details:

1. Jira ticket ID (e.g.: QE-83414): _
2. Feature name (snake_case): _
3. Test scope: a) UI only  b) API only  c) Both
4. PRD content (paste from Jira or Confluence): _
5. QE Owner (your name): _
6. (Optional) Test focus — areas to emphasize or deprioritize? (Enter to skip): _
```

**User provides inputs → skill prints compact requirements summary → user confirms →
skill builds test cases internally → runs silent coverage gap check + quality score (auto-fixes if < 75) →
skill prints compact test list with tags + score → user confirms → skill reads KB index + targeted sections →
mkdir -p smtp_testplan → writes CSV + Confluence .md → prints output summary.**
