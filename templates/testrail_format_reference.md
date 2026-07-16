# TestRail CSV Format Reference

Source: `/Users/ramegowdak/Downloads/SMTP_Proxy_Envelope_Testing_testrail.csv`
         `/Users/ramegowdak/Downloads/SMTP_Proxy_DKIM_Verification_testrail.csv`

---

## Column Order (exact, positional)

| # | Column Name | Description | Fixed Value? |
|---|---|---|---|
| 1 | Test Categories | Test type category | See allowed values below |
| 2 | Component | System under test | Always: `SMTP Proxy` |
| 3 | Test Summary | One-line test title (imperative, starts with "Verify…") | Dynamic |
| 4 | Steps | Numbered step list. Start with `Automation Steps:` for automatable tests, `MANUAL STEPS:` for manual | Dynamic |
| 5 | Expected Result | Bullet list of assertions, one per line | Dynamic |
| 6 | Priority (P0/P1/P2/P3) | Risk-based priority | Dynamic |
| 7 | Automatable | `Yes` or `No` | Dynamic |
| 8 | Automated | Always `No` for new tests (not yet implemented) | Always: `No` |
| 9 | UI Case | Always `No` for SMTP tests | Always: `No` |
| 10 | QE Owner | Test owner name | Dynamic (user-provided, default: `Ramegowda K`) |
| 11 | Suggested by Dev | Always `No` unless Design Spec specifies otherwise | Always: `No` |
| 12 | Result | Empty for new tests | Always: `` (empty) |
| 13 | Label | Tag for source tracking | Always: `ai_generated` |

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

## Priority Definitions

| Value | Meaning |
|---|---|
| `P0` | Blocking — core flow broken means feature is unusable |
| `P1` | High — significant impact on feature correctness or security |
| `P2` | Medium — edge case or secondary flow, noticeable but not blocking |
| `P3` | Low — cosmetic, rare edge case, or very low failure probability |

---

## Steps Format Rules

### Automatable test
```
Automation Steps:
1. <action using API/CLI/smtplib — include exact method names>
2. <next action>
3. <assertion — include exact code pattern, e.g. assert code == 250>
N. Teardown: <cleanup actions>
```

### Manual test
```
MANUAL STEPS:
1. <prerequisite setup>
2. <manual action>
3. <what to observe>
4. <assertion criterion>
5. Restore: <cleanup>
```

### Step writing rules
- Use placeholder variables for environment-specific values (see below)
- Reference specific libraries: `smtplib.SMTP`, `kubectl exec`, `docker exec`, `ssh`
- Include teardown as the last numbered step
- If test is parametrized, note: `Test is parametrized: runs once with X and once with Y`

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
