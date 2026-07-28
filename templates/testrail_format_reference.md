# TestRail CSV Format Reference

Source: `/Users/ramegowdak/Downloads/SMTP_Proxy_Envelope_Testing_testrail.csv`
         `/Users/ramegowdak/Downloads/SMTP_Proxy_DKIM_Verification_testrail.csv`

---

## Column Order (exact, positional)

| # | Column Name | Description | Fixed Value? |
|---|---|---|---|
| 1 | Test Categories | Test type category | See allowed values below |
| 2 | Sub-Type | ISTQB technique/outcome tag: `POS`, `NEG`, `BND`, `SEC`, `REG`, or `INTEG` (no brackets) | See §2.4 in SKILL.md |
| 3 | Component | System under test | Always: `SMTP Proxy` |
| 4 | Test Summary | One-line test title (imperative, starts with "Verify…") | Dynamic |
| 5 | Steps | Automation-oriented numbered step list. Start with `Automation Steps:`. Exact method/API/CLI names — see Steps Format Rules | Dynamic |
| 6 | Manual Execution Steps | Plain-language numbered step list a non-technical QE can follow by hand — no code/API jargon. Always populated, regardless of the Automatable value — see Manual Execution Steps Format Rules | Dynamic |
| 7 | Expected Result | Bullet list of assertions, one per line | Dynamic |
| 8 | Priority (P0/P1/P2/P3) | Risk-based priority | Dynamic |
| 9 | Automatable | `Yes` or `No` | Dynamic |
| 10 | Automated | Always `No` for new tests (not yet implemented) | Always: `No` |
| 11 | UI Case | Always `No` for SMTP tests | Always: `No` |
| 12 | QE Owner | Test owner name | Dynamic (user-provided, default: `Ramegowda K`) |
| 13 | Suggested by Dev | Always `No` unless Design Spec specifies otherwise | Always: `No` |
| 14 | Result | Empty for new tests | Always: `` (empty) |
| 15 | Label | Tag for source tracking | Always: `ai_generated` |

---

## Allowed Values — Test Categories

| Value | When to use |
|---|---|
| `Functional` | Core feature behavior, happy path, and failure modes |
| `Security` | Auth bypass, injection, TLS/DKIM/STARTTLS attacks, input validation |
| `Performance` | Throughput, latency, connection limits, timeouts under load |
| `Regression` | Re-test of a previously known bug fix |
| `Boundary` | Edge values: max recipients, max body size, max header length, etc. |
| `Negative` | Invalid inputs, bad sequences, protocol violations |
| `Integration` | End-to-end flows involving multiple systems (relay, DLP, policy engine) |

---

## Allowed Values — Sub-Type

| Value | When to use |
|---|---|
| `POS` | Positive/happy-path case: UC, DT/ST happy-path rows, EP valid partition |
| `NEG` | Negative case: EG errors, EP invalid partition, DT/ST error rows |
| `BND` | Boundary Value Analysis case (at/below/above a limit) |
| `SEC` | Security-derived Error Guessing case |
| `REG` | Regression case on an existing feature |
| `INTEG` | Phase 1 x Phase 2 integration case (Phase 2 runs only) |

---

## Priority Definitions

| Value | Meaning |
|---|---|
| `P0` | Blocking — core flow broken means feature is unusable |
| `P1` | High — significant impact on feature correctness or security |
| `P2` | Medium — edge case or secondary flow, noticeable but not blocking |
| `P3` | Low — cosmetic, rare edge case, or very low failure probability |

---

## Steps Format Rules

The **Steps** column is always automation-oriented — code/API/CLI level, whether or not the test is
currently Automatable. If `Automatable=No`, Steps still records the technical mechanism that *would*
drive it (or the reason none exists, e.g. requires tcpdump/manual DNS control) — the human-runnable
walkthrough for anyone executing it by hand lives in the separate **Manual Execution Steps** column
below, not here.

```
Automation Steps:
1. <action using API/CLI/smtplib — include exact method names>
2. <next action>
3. <assertion — include exact code pattern, e.g. assert code == 250>
N. Teardown: <cleanup actions>
```

### Step writing rules
- Use placeholder variables for environment-specific values (see below)
- Reference specific libraries: `smtplib.SMTP`, `kubectl exec`, `docker exec`, `ssh`
- Include teardown as the last numbered step
- If test is parametrized, note: `Test is parametrized: runs once with X and once with Y`

---

## Manual Execution Steps Format Rules

Every test case gets a Manual Execution Steps entry — this is the runbook a QE engineer without
coding context follows to execute the test by hand and judge pass/fail, independent of whether
automation exists yet.

```
MANUAL STEPS:
1. <plain-language prerequisite/setup action — product UI, email client, or a copy-pasteable command
   with no explanation of what the code does>
2. <plain-language action a human performs>
3. Observe: <exactly what to look at and where>
4. Confirm: <the pass/fail condition, in plain terms, matching Expected Result>
5. Restore: <cleanup, in plain language>
```

### Step writing rules
- No API method names, assertion code, or library references (`assert`, `smtplib.SMTP(...)`, etc.) —
  describe the equivalent human action instead (e.g. "send an email from `<SENDER_EMAIL>` to
  `<RECIPIENT_EMAIL>` using your mail client" rather than `EmailBuilderSmtp.send_email(...)`)
- Use placeholder variables from the table below the same way Steps does, but phrase each step as an
  instruction a first-time reader can follow without prior context
- Always include at least one `Observe:` step and one `Confirm:` step tied to the Expected Result
- Include a plain-language `Restore:`/cleanup step last
- **Never delegate a step to "Dev/SRE" or any other handoff** — this team has no separate Dev/SRE to
  hand off to; the QE running the test performs every step themselves. If the underlying action needs
  backend/pod access (tcpdump, config file checks, feature-flag state, kubectl exec), write it as a
  direct QE action using the access/tooling the KB documents (e.g. `kubectl exec` into
  `<SMTP_PROXY_POD>`, the feature-flag API, the relay-config API) rather than saying "ask" or "confirm
  with" anyone else:
  `4. Run kubectl exec -it <SMTP_PROXY_POD> -n <SMTP_PROXY_NS> -- tcpdump -i any port 25 -w /tmp/capture.pcap, then copy the file off the pod and inspect it and confirm <condition>`
  If genuinely no QE-accessible method exists for an action (rare), say so plainly as an Open Question
  rather than writing an unowned "ask someone" step.

---

## Placeholder Variables

Use these exact placeholder strings in Steps and Expected Result columns:

| Placeholder | Meaning |
|---|---|
| `<SMTP_PROXY_VIP>` | SMTP proxy virtual IP address |
| `<SMTP_PROXY_PORT>` | SMTP proxy port (usually 25) |
| `<SMTP_PROXY_POD>` | Kubernetes pod name in smtpproxy namespace |
| `<SMTP_PROXY_NS>` | Kubernetes namespace (usually `smtpproxy`) |
| `<SENDER_DOMAIN>` | Sender email domain used in test relay config |
| `<SENDER_EMAIL>` | Full sender email address |
| `<RECIPIENT_EMAIL>` | Full recipient email address |
| `<MAILSERVER_HOST>` | Docker/SSH hostname of the downstream mail server |
| `<MAILSERVER_USER>` | OS user on mail server for SSH access |
| `<TENANT_ID>` | Netskope tenant ID |
| `<RELAY_CONFIG_NAME>` | Name of the email relay config created in setup |
| `<RT_POLICY_NAME>` | Real-time policy name created in setup |
| `<FEATURE_FLAG>` | Name of the feature flag being toggled |
| `<DKIM_SELECTOR>` | DKIM selector string |
| `<DKIM_DOMAIN>` | DKIM signing domain |
| `<DKIM_PRIVATE_KEY>` | Path to DKIM private key PEM file |

---

## Expected Result Format Rules

- One bullet per assertion: `- <condition is true/false/value>`
- Lead with the primary success criterion
- Include negative assertions where relevant: `- Proxy does not crash`
- Include state-recovery assertions: `- Connection remains usable after rejection`
- Max ~6 bullets per test; consolidate related assertions

---

## CSV Encoding Rules

- File encoding: UTF-8
- Delimiter: `,` (comma)
- Multi-line cell values: wrap entire cell in double quotes `"`
- Embedded double quotes inside a cell: escape as `""`
- No trailing commas on rows
- First row is the header row (exact column names as listed above)
