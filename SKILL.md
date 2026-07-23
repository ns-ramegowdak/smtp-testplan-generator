---
name: smtp-testplan-generator
description: >
  Generate a complete test plan for SMTP Proxy features from a Design Spec (Jira or Confluence content).
  Produces a TestRail-ready CSV and a QE test-plan page in the team's real Confluence template —
  either published live to Confluence via MCP or written locally as a .md file, your choice.
  Strictly Design Spec-driven — only covers what the Design Spec describes, nothing more.
  Fetches the Design Spec directly from a Jira issue or Confluence page URL via the Atlassian MCP connection
  when available — no copy-pasting required.
  Use when: generating SMTP test plan, creating test cases from Design Spec, smtp testplan, smtp test coverage,
  smtp test cases from requirements, testrail csv smtp, smtp proxy testing plan.
triggers:
  - smtp-testplan-generator
  - smtp testplan
  - generate smtp test plan
  - smtp test cases from prd
  - smtp test cases from design spec
  - smtp testrail csv
  - smtp proxy test coverage
  - create test plan smtp
---

# SMTP Proxy Test Plan Generator

## Purpose

Generate a complete, ready-to-import test plan for any SMTP Proxy feature given its Design Spec content.

**Two outputs every run:**
1. TestRail CSV — `smtp_testplan/{TICKET_ID}_{feature_name}_testrail.csv`
2. QE test-plan page in the team's real Confluence template — either:
   - **Published live** to a Confluence space via the Atlassian MCP connection, or
   - Written locally to `smtp_testplan/{TICKET_ID}_{feature_name}_confluence.md`
   The user picks which in Phase 0.

**Core constraint:** Test cases are generated STRICTLY from the Design Spec content provided. Do not add
test cases for behaviors, configurations, or edge cases not described in the Design Spec. If a behavior
seems implied but is not stated, call it out in Section 9 (Open Questions) instead.

---

## Phase 0 — Input Collection

Before doing anything else, ask the user for the following. All fields are required except item 6.

```
smtp-testplan-generator needs a few details:

1. Jira ticket ID  (e.g.: QE-83414, NPLAN-4895)
   If item 4 is a Jira URL for this same ticket, you can skip this — it will be inferred.

2. Feature name (snake_case, used in output filenames)
   e.g.: dkim_verification, envelope_testing, lfs_size_enforcement

3. Test scope:
   a) UI only  (Alerts/Events/Incidents pages, SMTP Settings page)
   b) API only  (SMTP protocol, feature flags, relay config, logs, smtplib)
   c) Both UI and API

4. Design Spec source — one of:
   a) A Jira issue URL or key (e.g. https://netskope.atlassian.net/browse/NPLAN-4895, or just NPLAN-4895)
      — fetched automatically via the Atlassian MCP connection.
   b) A Confluence page URL (e.g. https://netskope.atlassian.net/wiki/spaces/SMTPDLP/pages/5229281503/Design+Spec+-+...)
      — fetched automatically via the Atlassian MCP connection.
   c) Pasted Design Spec content, if you'd rather not use MCP or the page isn't in Atlassian.
      Include: feature description, acceptance criteria, error handling requirements,
      security requirements, any listed edge cases or notes.

5. QE Owner name (your name — used in TestRail CSV)

6. (Optional) Test focus — any areas to emphasize or deprioritize?
   e.g.: "Emphasize boundary cases for the size limit", "Deprioritize export tests"
   Press Enter to skip.

7. Where should the Confluence-formatted test plan go?
   a) Create it as a live Confluence page via the Atlassian MCP connection (no copy-paste)
   b) Write it locally as a .md file for you to paste in yourself

8. (Only if 7a) Confluence publish details:
   a) Space to create the page in (key or ID, e.g. ENG)
   b) Parent page to nest under (optional — URL or ID, e.g. the ticket's Epic or the Feature Design doc page). Press Enter to place it at the space root.
   c) QE Epic Link (optional — Jira epic URL for QE tracking). Press Enter to skip.
   d) Release Schedule (optional, e.g. R140). Press Enter to skip.
   e) Dev Members (optional, e.g. "Backend: Jane Doe   WebUI: John Smith"). Press Enter to skip.
   f) Test Rail Link (optional — paste once the TestRail run exists). Press Enter to skip.
   g) Testing Estimate (optional, e.g. "Total: 2w   Manual: 8d   Automation: 1w"). Press Enter to skip.
```

Wait for all inputs before proceeding. Do not begin analysis until the Design Spec source (URL or pasted content) has been provided.

After receiving inputs:
- Set `FEATURE_NAME` = feature name provided
- Set `TEST_SCOPE` = `UI` / `API` / `Both` based on answer to item 3
- Set `QE_OWNER` = name provided
- Set `TEST_FOCUS` = emphasis/deprioritize notes from item 6, or empty if skipped
- Set `CONFLUENCE_OUTPUT_MODE` = `Publish` (item 7a) or `Local` (item 7b)
- If `Publish`: set `CONFLUENCE_SPACE`, `CONFLUENCE_PARENT_ID` (or empty), `QE_EPIC_LINK`, `RELEASE_SCHEDULE`, `DEV_MEMBERS`, `TEST_RAIL_LINK`, `TEST_ESTIMATE` from item 8 (empty string for any field the user skipped — never invent a value for these)
- Set `DESIGN_SPEC_INPUT` = the raw value given for item 4 (a URL, a bare Jira key, or pasted text)
- Determine `DESIGN_SPEC_SOURCE`:
  - If `DESIGN_SPEC_INPUT` looks like a Jira URL (`/browse/<KEY>`) or a bare key (`^[A-Z][A-Z0-9]+-\d+$`) → `DESIGN_SPEC_SOURCE = Jira-MCP`
  - If `DESIGN_SPEC_INPUT` looks like a Confluence URL (`/wiki/spaces/...` or `/wiki/x/...` tiny link) → `DESIGN_SPEC_SOURCE = Confluence-MCP`
  - Otherwise (free text pasted in) → `DESIGN_SPEC_SOURCE = Manual`
- Set `TICKET_ID`:
  - If item 1 was given, use it as-is.
  - Else if `DESIGN_SPEC_SOURCE = Jira-MCP`, infer it from the issue key in `DESIGN_SPEC_INPUT`.
  - Else (Confluence Design Spec with no ticket ID given), ask the user for the ticket ID before proceeding — it's required for output filenames.

If `DESIGN_SPEC_SOURCE` is `Jira-MCP` or `Confluence-MCP`, proceed to **Phase 0.5** to fetch the content before Phase 1. If `Manual`, skip Phase 0.5 and go straight to Phase 1 using the pasted text as the Design Spec content.

---

## Phase 0.5 — Fetch Design Spec via Atlassian MCP

Only runs when `DESIGN_SPEC_SOURCE` is `Jira-MCP` or `Confluence-MCP`. Goal: pull the Design Spec content directly instead of relying on copy-paste, while keeping the conversation output compact — don't dump the full fetched body into chat, just a short confirmation (see the Quality Rules / token-efficiency guidance for this skill).

### 0.5.1 Resolve the cloud ID

Extract the site hostname from the URL in `DESIGN_SPEC_INPUT` (e.g. `yourorg.atlassian.net`). Try it directly as `cloudId` on the first fetch call below. If that call fails with an auth/not-found error, call `mcp__atlassian__getAccessibleAtlassianResources` once, pick the matching site, and retry with its returned cloud ID. Reuse the resolved `cloudId` for any subsequent calls in the same run — don't re-resolve it.

If `DESIGN_SPEC_INPUT` was a bare Jira key with no URL (no hostname to try), call `mcp__atlassian__getAccessibleAtlassianResources` directly to get the cloud ID.

### 0.5.2 Fetch the content

**If `DESIGN_SPEC_SOURCE = Jira-MCP`:**
- Call `mcp__atlassian__getJiraIssue` with `issueIdOrKey = TICKET_ID`, `cloudId`, `fields: ["summary","description","comment","labels","components","issuelinks"]`, `responseContentFormat: "markdown"`.
- Jira tickets for this team are often thin customer-request narratives, not the real Design Spec — and the Confluence link isn't always where you'd expect. Search **all three** of `description`, every entry in `comment.comments[].body`, and `issuelinks` for a Confluence URL (`/wiki/spaces/...`) — on real tickets it has shown up buried in a mid-thread engineering comment, not the description:
  - If found, ask the user once: "This ticket links to a Confluence page — `{title/url}`. Fetch that as the Design Spec too? (y/n)". If yes, also run the Confluence fetch below against that page and treat the combined content (Jira description + comments + linked Confluence body) as the Design Spec.
  - If not found, use the Jira issue's `description` + `comment` fields as the Design Spec content.
- Also scan `issuelinks` for a linked issue whose key matches `QE-\d+` (commonly summarized "QE activities for ..."). If found, surface it as a suggested `{QE_EPIC_LINK}` value for Phase 0 item 8c and ask the user to confirm/override rather than silently filling it in — don't skip asking just because a candidate was found.

**If `DESIGN_SPEC_SOURCE = Confluence-MCP`:**
- Extract the page ID from the URL (the numeric ID in `/pages/<id>/`, or the tiny-link ID after `/wiki/x/`).
- Call `mcp__atlassian__getConfluencePage` with `pageId`, `cloudId`, `contentFormat: "markdown"`.

### 0.5.3 Confirm before analyzing

Print a compact confirmation — not the full fetched body:

```
Fetched Design Spec source:
  {Jira NPLAN-4895: "<issue summary>"} or {Confluence: "<page title>"}
  Content length: ~N words
  {If a linked Confluence Design Spec was also pulled in, name it here too}
```

Ask: "Is this the right Design Spec? Proceed with analysis, or should I fetch something else?"
Wait for confirmation. If the user says this is the wrong page/ticket, ask for the correct URL/key and re-run 0.5.1–0.5.3.

### 0.5.4 Fallback to manual paste

If any MCP call errors (no Atlassian MCP connection available, permission denied, page/issue not found), tell the user plainly what failed and fall back to Phase 0 item 4c: ask them to paste the Design Spec content directly. Do not retry the same failing call more than once.

---

## Phase 1 — Design Spec Analysis

Read the Design Spec content and extract the following. Work entirely from what is stated — no inference.

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

## Phase 1.5 — Feature Impact Analysis

Goal: use product-level knowledge of SMTP Proxy (not the test-mechanics KB in `kb/`) to work out
where the new feature sits in the architecture and what existing features it could affect —
so the Confluence "Dependency features and Regression Impact" section reflects real product
context instead of falling back to "(not specified)", and so regression coverage can be added
deliberately rather than left to chance.

### 1.5.1 Read the product references

Both files are small — read each in full, no targeted/offset reads needed here (unlike `kb/`):
- `~/.claude/skills/smtp-testplan-generator/references/product_architecture.md`
- `~/.claude/skills/smtp-testplan-generator/references/feature_matrix.md`

### 1.5.2 Map the new feature onto the architecture

From the Design Spec content and requirements map (§1.1), identify which architecture
component(s) in `product_architecture.md` the new feature changes (e.g. MSA/relay config, RT
policy engine, DLP scanning metadata/content path, header injection, a specific UI surface).

If the feature doesn't clearly map to any documented component, say so plainly — do not force a
match.

### 1.5.3 Find interaction/regression candidates

Using `feature_matrix.md`, find every existing feature row whose "Shares architecture with"
column overlaps the component(s) from §1.5.2. Pull that row's "Regression risk if changed" note.

Build:
- `IMPACTED_FEATURES` — list of existing feature names with a one-line reason each
- `REGRESSION_RISK_NOTES` — the specific risk statements pulled from the matrix

### 1.5.4 Confirm with the user

Print a compact summary:

```
Feature Impact Analysis — {FEATURE_NAME}
Touches: {architecture component(s)}
Related existing features: {feature name} — {one-line reason}, ...
Regression risk notes: {risk statement}, ...
```

Ask: "Does this impact analysis look right — anything to add or correct? And should I generate
`[REG]`-tagged regression test cases for the impacted areas above, or just note them in the
Dependency/Regression Impact section for now?"

Wait for confirmation. Record the answer as `GENERATE_REGRESSION_TESTS` = yes/no. If the user
corrects the impacted-features list, use their corrected version for the rest of this run.

**Note:** generating `[REG]` tests only when the user opts in here keeps Quality Rule 1
(Design Spec-only scope) intact by default — regression coverage of existing features is an
explicit, confirmed addition, not an inferred one.

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
| **Regression (REG)** | Existing features flagged in Phase 1.5 §1.5.3, only if `GENERATE_REGRESSION_TESTS = yes` | `[REG]` |

### 2.2 Priority Assignment Rules

| Priority | Assign when |
|---|---|
| P0 | Core feature requirement; failure means feature is broken or insecure |
| P1 | Important but failure degrades correctness without breaking the whole feature |
| P2 | Edge case, secondary flow, or low-probability failure mode |
| P3 | Cosmetic or near-zero impact — usually omit unless Design Spec explicitly requires |

**Default:** Explicit Design Spec acceptance criterion → P0. Described behavior without AC → P1. Edge case/note → P2.

**If `TEST_FOCUS` is set:** Apply it as a weighting override after initial priority assignment.
- "Emphasize X" → bump requirement IDs related to X up one priority level (P1 → P0, P2 → P1) and generate additional test cases for those requirements if coverage is thin.
- "Deprioritize X" or "Skip X" → drop related cases to P2 or move them to Open Questions if they are not Design Spec acceptance criteria.

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
- `[REG]` — regression test on an existing feature flagged in Phase 1.5, only when
  `GENERATE_REGRESSION_TESTS = yes`. Steps must verify the existing feature's documented
  behavior is unchanged, not test the new feature itself.

Prepend the tag to the test Summary: `[POS] Verify ...`, `[NEG] Verify ...`, `[REG] Verify ...`.

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
TC-5  P2  Regression   [REG] Verify ...
...
Total: N  |  Auto: X/N  |  P0: A  P1: B  P2: C
Coverage: [POS]: A  [NEG]: B  [BND]: C  [SEC]: D  [REG]: E
```

Ask: "Does this list look complete? Add, remove, or reprioritize anything?"
Wait for confirmation before generating output files.

---

## Phase 4 — Output Generation

### Step 1: Load reference files (index-first, targeted, ranged reads)

**1a. Read the KB index first:**
`~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_index.md`
(this now includes a Step 2b line-number map — read that table too, it's small)

**1b. Based on `TEST_SCOPE`, identify the relevant KB file(s):**
- `TEST_SCOPE = UI`   → `smtp_proxy_kb_ui.md`
- `TEST_SCOPE = API`  → `smtp_proxy_kb_api.md`
- `TEST_SCOPE = Both` → both files

**1c. Do NOT read a full KB file by default.** For each feature area your Design Spec touches,
look up its section(s) in the Step 1/Step 2 tables, then look up each section's line
range in the Step 2b map. Call `Read` with `offset`/`limit` set to that range (+~3 lines
buffer) — one targeted read per section (or per contiguous run of sections), not one
read of the entire file. Skip sections with no bearing on the Design Spec entirely.

**Exception — the 60% fallback:** if the sections you need cover more than ~60% of a
file's total lines (see the totals in Step 2b), just read the whole file in one call.
Past that threshold, multiple ranged reads cost more overhead than a single full read.

**Always read these two files regardless of scope:**
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
- `Suggested by Dev=No` unless Design Spec explicitly attributes a test to a dev suggestion
- `QE Owner={QE_OWNER}`
- `[REG]`-tagged test cases (§2.4) use `Test Categories=Regression` in Column 1

### Step 3: Build the Confluence page content

Use `confluence_template.md` as the structural skeleton — this mirrors the team's real QE test-plan pages in the `ENG` Confluence space. Fill in every `{placeholder}`. Build this content in memory first; Step 4 decides whether it gets published live or written to a local file.

| Placeholder | Source |
|---|---|
| `{TICKET_ID}`, `{FEATURE_TITLE}` | User input / Design Spec title |
| `{NPLAN_ENG_EPIC}` | The Design Spec URL if `DESIGN_SPEC_SOURCE = Confluence-MCP`; the linked Confluence page URL if a Jira ticket linked out to one (§0.5.2); otherwise `(not specified)` |
| `{QE_EPIC_LINK}` | Item 8c (the user's answer, including a `QE-\d+` link suggested and confirmed during §0.5.2), or `(not specified)` if skipped |
| `{PRODUCT_REQUIREMENT_DOC}` | Leave `(not specified)` unless the user separately supplied one — do not reuse the Design Spec link here, they're distinct fields on the real template |
| `{FEATURE_DESIGN_DOC}` | Same value as `{NPLAN_ENG_EPIC}` — the two rows point at the same doc on both reference pages |
| `{RELEASE_SCHEDULE}` | Item 8d, or `(not specified)` |
| `{QE_MEMBERS}` | `{QE_OWNER}` |
| `{DEV_MEMBERS}` | Item 8e, or `(not specified)` |
| `{TEST_RAIL_LINK}` | Item 8f, or `(not specified)` |
| `{TEST_ESTIMATE}` | Item 8g, or `(not specified)` |
| `{FEATURE_TOI_LINK}` | `(not specified)` — always filled in later by hand |
| `{VERSION}` | `1.0` |
| `{SCOPE_NARRATIVE}` | Short paragraph + bullets summarizing key feature changes/behaviors, drawn from Behaviors [B-N] — mirrors the "Feature changes:" block seen on real pages. Omit (empty string) if the Design Spec is too thin to summarize beyond the scope table itself. |
| `{IN_SCOPE_BULLETS}` / `{NOT_IN_SCOPE_BULLETS}` | Behaviors [B-N] + Error conditions [E-N] for in-scope; adjacent SMTP features not in the Design Spec + explicit exclusions for not-in-scope |
| `{SETUP_DIAGRAM}` | Simple `A → B → C` text arrow diagram of the component flow, only if the Design Spec describes one; else `(not specified in Design Spec)` |
| `{STACK_ACCESS_NOTES}` + `{TEST_REQUIREMENTS_TABLE_ROWS}` | Environment/stack requirements — reuse the placeholder variables from `testrail_format_reference.md` (VIP, namespace, tenant, etc.); mark availability `Yes` only if the KB confirms it, else leave blank |
| `{FEATURE_FLAGS_TABLE_ROWS}` | One row per flag `[F-N]` from the requirements map: Type / Flag Name / Default Status. If no flags exist, one row: `NA \| NA \| —` |
| `{DEPENDENCY_FEATURES_AND_REGRESSION_IMPACT}` | `IMPACTED_FEATURES` + `REGRESSION_RISK_NOTES` from Phase 1.5 (confirmed with the user), combined with anything the Design Spec itself states about dependencies. If Phase 1.5 found no clear mapping and the Design Spec states nothing, use `(not specified in Design Spec)` |
| `{ACCEPTANCE_CRITERIA_BULLETS}` | Explicit acceptance criteria from the Design Spec. If none are stated verbatim, use the three standard bullets seen on both reference pages (build valid + deployed, dev unit tests done, QE test cases pass) and note in Open Questions that Design-Spec-specific ACs were not found |
| `{RESILIENCY_SECTION}` | Only include when the Design Spec describes failure modes/edge cases worth calling out as "Potential Failures": render as `## Resiliency of Service/Feature\n\nPotential Failures :\n\n{bullets from [Q-N]/edge cases}`. Otherwise this placeholder resolves to an empty string (omit the whole section — do not print an empty heading) |
| Test Cases table rows | Group test cases under `Section` header rows you choose to fit the feature (e.g. `Backend API`, `WebUI`, `E2E`, `Negative`, `Security` — the real pages use feature-specific section names, not a fixed list). Within each section, one row per test case: S.No (`TC-N`), Section, Test Categories(Type) = the `[POS]/[NEG]/[BND]/[SEC]/[REG]` category, Service/Component, Test Summary, Steps, Expected Result, Priority, Automatable, `Automated=No`, `UI Case` (Yes only if it drives a UI page object), `Arrived by QE=No`, `Suggested by Dev` (same rule as the CSV), `Derived by AI=Yes` |
| `{DETAILED_TEST_CATEGORY_NOTES}` | One `###` subsection per Section used above, 2–4 bullets summarizing what that section validates (mirrors "WebUI Test" / "End to End Functional Test" / "Performance Test" subsections on the reference page) |
| `{ACTIONABLE_ITEMS_CHECKLIST}` | One checkbox item per Open Question `[Q-N]`, plus the standard `- [ ] Feature Automation needed?` item |
| `{TOI_LINKS}` | `To be Recorded.` |
| `{KEY_BUGS_TASKS}` | `(not specified)` |

Do NOT invent content for any section — if a section has no Design Spec basis, write `(not specified in Design Spec)` rather than fabricating specifics.

### Step 4: Deliver the Confluence content — Publish or Local

**If `CONFLUENCE_OUTPUT_MODE = Local`:**
- Write the filled-in content to `smtp_testplan/{TICKET_ID}_{FEATURE_NAME}_confluence.md`.

**If `CONFLUENCE_OUTPUT_MODE = Publish`:**
1. Resolve `cloudId` — reuse the one resolved in Phase 0.5 if this run already fetched from Atlassian MCP; otherwise resolve it now the same way (site hostname first, `mcp__atlassian__getAccessibleAtlassianResources` as fallback).
2. Print an explicit confirmation before creating anything — this writes to a shared system other people will see:
   ```
   About to create a Confluence page:
     Title:  {TICKET_ID} {FEATURE_TITLE}
     Space:  {CONFLUENCE_SPACE}
     Parent: {CONFLUENCE_PARENT_ID or "(space root)"}
   Proceed? (y/n)
   ```
   Wait for explicit confirmation. If the user declines, fall back to Local mode instead (write the file) rather than dropping the output.
3. On confirmation, call `mcp__atlassian__createConfluencePage` with `cloudId`, `spaceId = CONFLUENCE_SPACE`, `parentId = CONFLUENCE_PARENT_ID` (omit if empty), `title = "{TICKET_ID} {FEATURE_TITLE}"`, `body` = the filled-in content, `contentFormat = "markdown"`.
4. If the call fails for any reason (bad space key, permission denied, MCP unavailable), tell the user plainly what failed and fall back to writing the local `.md` file instead so the work isn't lost. Do not retry the same failing call more than once.
5. On success, note the returned page URL for the final output summary.

### Step 5: Confirm output

After the CSV is written and the Confluence content is delivered (published or written locally), print:

```
Output:
  CSV:        smtp_testplan/{TICKET_ID}_{FEATURE_NAME}_testrail.csv
  Confluence: {Published — <page URL>}  or  {smtp_testplan/{TICKET_ID}_{FEATURE_NAME}_confluence.md}

Summary:
  Total test cases: N
  P0: A  |  P1: B  |  P2: C  |  P3: D
  Automatable: X/N

Next steps:
  1. {If Local: "Review the Confluence page and paste into your Confluence space" / If Published: "Review the live Confluence page"}
  2. Import the CSV into TestRail (use "Import Test Cases" → CSV)
  3. Review Open Questions / Actionable Items with Dev/PM before finalizing
```

---

## Quality Rules (apply throughout all phases)

1. **Design Spec-only scope** — If a behavior is not in the Design Spec, do not test it. If it seems important, add it to Open Questions.
2. **No duplicate assertions** — Two tests that assert the same outcome for the same condition are one test.
3. **Teardown in every test** — Every test that creates a relay config, RT policy, or feature flag change must include teardown in the last step(s).
4. **Placeholder variables only** — Never hardcode IP addresses, pod names, tenant IDs, or email addresses. Use placeholders from `testrail_format_reference.md`.
5. **Exact SMTP codes** — When the Design Spec specifies a response code, use it exactly. When the Design Spec says "error", use the correct RFC 5321 code and note the basis.
6. **Steps must be executable** — Each step must be a concrete action an automation engineer can implement directly. Avoid vague steps like "verify behavior" — say `assert response_code == 250`.
   - **Forbidden words in steps:** "correctly", "properly", "as expected", "should work", "the button", "equivalent", "appropriate", "or similar".
   - **UI element references** must use the exact visible label, placeholder, or aria-label as it appears in the product — never a description like "the save button".
   - **API steps** must spell out the exact endpoint path from the Design Spec — never a guessed or constructed path. If a path is absent from the Design Spec, write `<path not in Design Spec>` and flag it in Open Questions.
   - **Forbidden words in expected results:** "correctly", "properly", "as expected", "should work". Every assertion must be independently verifiable: HTTP status code, exact UI text, element visibility state, API response field + value, log entry content.
7. **Confirmation before output** — Always get user confirmation on the requirements summary (Phase 1) and test list (Phase 3) before writing files.

---

## Reference Files

| File | Purpose |
|---|---|
| `~/.claude/skills/smtp-testplan-generator/references/product_architecture.md` | **Phase 1.5** — SMTP Proxy product architecture: data flow, MSA types, RT policy, DLP scanning, header injection, UI surfaces. Read in full — small, no targeted reads |
| `~/.claude/skills/smtp-testplan-generator/references/feature_matrix.md` | **Phase 1.5** — existing feature list with dependencies/interaction candidates/regression risk, used to identify impact of the new feature. Read in full |
| `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_index.md` | **Phase 4 — read first** — decision table for which KB file/sections to load, plus a line-number map for targeted offset/limit reads |
| `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_ui.md` | UI layer KB: SMTP Settings page, Alerts/Events/Incidents patterns |
| `~/.claude/skills/smtp-testplan-generator/kb/smtp_proxy_kb_api.md` | API/backend KB: smtplib, kubectl, feature flags, logs, relay config |
| `~/.claude/skills/smtp-testplan-generator/templates/testrail_format_reference.md` | CSV column spec, placeholder vars, step/expected format rules |
| `~/.claude/skills/smtp-testplan-generator/templates/confluence_template.md` | QE test-plan page skeleton, matching the team's real `ENG`-space pages |
| `~/.claude/skills/smtp-testplan-generator/examples/` | Sample outputs for reference only — do NOT read at runtime |

> `references/` (product knowledge — what the product does) is distinct from `kb/` (test mechanics
> — how to write the test code). Read both `references/` files in full during Phase 1.5. Always
> read the KB index first during Phase 4. Never read example files at runtime.

---

## Skill Layout

```
~/.claude/skills/smtp-testplan-generator/
├── SKILL.md                          ← this file
├── references/
│   ├── product_architecture.md       ← Phase 1.5: product architecture, read in full
│   └── feature_matrix.md             ← Phase 1.5: existing features + regression risk, read in full
├── kb/
│   ├── smtp_proxy_kb_index.md        ← Phase 4: read first every run
│   ├── smtp_proxy_kb_api.md          ← API/backend patterns
│   └── smtp_proxy_kb_ui.md           ← UI layer patterns
├── templates/
│   ├── testrail_format_reference.md  ← always read
│   └── confluence_template.md        ← always read
└── examples/
    ├── machine_generated_emails_testrail.csv   ← reference only
    └── machine_generated_emails_confluence.md  ← reference only
```

The CSV is always written to `smtp_testplan/` inside the current working directory (created with `mkdir -p smtp_testplan` before the first write). The Confluence output either goes to that same directory as a `.md` file, or is published live to Confluence — whichever the user chose in Phase 0 item 7. Files are named `{TICKET_ID}_{FEATURE_NAME}_testrail.csv` and `{TICKET_ID}_{FEATURE_NAME}_confluence.md`.

---

## Example Invocation

**User types:**
```
/smtp-testplan-generator
```

**Skill asks:**
```
smtp-testplan-generator needs a few details:

1. Jira ticket ID (e.g.: QE-83414): _  (skip if item 4 is a Jira URL for this ticket)
2. Feature name (snake_case): _
3. Test scope: a) UI only  b) API only  c) Both
4. Design Spec source: a) Jira URL/key  b) Confluence URL  c) pasted content: _
5. QE Owner (your name): _
6. (Optional) Test focus — areas to emphasize or deprioritize? (Enter to skip): _
7. Confluence output: a) Publish live via MCP  b) Write locally as .md: _
8. (Only if 7a) Space, parent page (optional), QE Epic Link (optional), Release Schedule (optional),
   Dev Members (optional), Test Rail Link (optional), Testing Estimate (optional): _
```

**User provides inputs → if item 4 is a Jira/Confluence URL or key, skill fetches it via Atlassian MCP
(resolves cloud ID, pulls issue/page content, follows a linked Confluence Design Spec if the Jira ticket points
to one, prints a one-line confirmation of what it fetched) → user confirms the fetched content is right
(or pastes manually if MCP fetch fails/unavailable) → skill prints compact requirements summary → user
confirms → skill reads `references/product_architecture.md` + `references/feature_matrix.md`, maps the
feature onto the architecture, and finds impacted existing features + regression risks → user confirms
the impact analysis and chooses whether to generate `[REG]` regression tests → skill builds test cases
internally → runs silent coverage gap check + quality score (auto-fixes if < 75) → skill prints compact
test list with tags + score → user confirms → skill reads KB index + targeted sections → mkdir -p
smtp_testplan → writes the TestRail CSV → builds the Confluence page content (Dependency/Regression
Impact section now grounded in the Phase 1.5 analysis) → either confirms space/parent and publishes live
via `createConfluencePage`, or writes the local .md file → prints output summary with the page URL (if
published) or file path.**
