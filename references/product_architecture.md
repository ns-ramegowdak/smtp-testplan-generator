# SMTP Proxy — Product Architecture

Source: docs.netskope.com/en/netskope-smtp-proxy and its sub-pages, cross-checked and extended
against the real automation code in `nsproxy/tests/api/SMTP/smtp_proxy` and `nsproxy/tests/ui/SMTP`
(the code confirmed real header names, ports, and endpoints where the docs were silent or vague).
Use this together with `feature_matrix.md` in Phase 1.5 to reason about where a new feature sits
in the product and what it touches. For test-writing mechanics (page objects, smtplib patterns,
log formats) see `../kb/` instead — this file is about the product, not test code.

## What it is

Netskope SMTP Proxy is a DLP scanning service that intercepts outbound email traffic in transit
between a customer's cloud email service and the upstream mail transfer agent (MTA), evaluates it
against Real-time Protection (RT) policies, and takes enforcement action before the message is
forwarded on for delivery.

## End-to-end data flow

```
Cloud email service (O365 Exchange / Gmail / Custom MSA)
        │  outbound "Send" activity
        ▼
Netskope SMTP Proxy
   1. Tenant identification (SNI / tenant verification attributes / source IP)
   2. Real-time Protection policy match (source, destination, activity=Send)
   3. DLP scan — metadata (to/from/cc/attachment) + content (subject/body)
   4. Action: Allow / Alert / Add SMTP Header / Remove Recipients / Add Traffic Action
   5. Tag message x-netskope-inspected (loop-prevention on re-ingest)
        │
        ▼
Upstream MTA (customer's own mail server / relay)
        │
        ▼
Recipient mailbox
```

Multiple messages can be processed within a single SMTP session (connection reuse —
front-conn/back-conn persistence, see `../kb/smtp_proxy_kb_api.md` §3). The proxy handles
connection lifecycle edge cases explicitly: DNS resolution happens per back-connection, and
RSET/QUIT/HELO/EHLO sent in quick succession mid-session must be handled gracefully (NPLAN-5173).

## Alternate ingestion path — Outlook Plugin (REST API)

Not all mail enters via SMTP. The Outlook Plugin submits mail via a **REST API**, not the SMTP
protocol — a structurally different ingestion path that still goes through the same DLP/policy
pipeline (NPLAN-6393):

```
Outlook Plugin
   │  (via a separate "epdlp-mp-gateway" pod, namespace epdlp-mp — not the customer's own MTA)
   │  POST https://smtpproxy.smtpproxyv2:8000/api/v1/email/send
   │  body: {sender, recipients[], headers: {Subject, Date, Message-ID, To, From, ...}, body}
   │  headers: x-ns-tenant-id, x-ns-app-name, x-plugin-version (all required)
   ▼
SMTP Proxy REST listener (port 8000)
   → same RT policy / DLP scan / action pipeline as SMTP-ingested mail
   → works with both sync and async DLP (see below)
   → ALWAYS responds HTTP 200 with a structured JSON body: {status, scan_result: {status, ...},
     message, request_id} — even for a blocking policy match, the proxy itself still returns 200
     with a scan_result describing the outcome; it does not reject the HTTP call
   ▼
epdlp-mp-gateway (the actual caller)
   → interprets scan_result and, for a blocking/justify-required outcome, is the one that renders
     the inline response page to the end user inside Outlook — epdlp_content_block_page.html /
     epdlp_content_useralert_justify.html. The SMTP Proxy pod does not render these pages itself.
```

**RT policies must specifically target "Outlook Plugin" as a destination.** A policy configured
for an MSA (Exchange/Gmail/Custom, "inline-only") does **not** automatically apply to
plugin-submitted traffic and vice versa — confirmed by an explicit test where an inline-only
policy is configured and plugin traffic still gets "no policy matched" (still HTTP 200 + an empty
`scan_result`, not an error). Any feature spec involving policy targeting must consider both
destinations as separate, not one implying the other.

**This is a genuinely different action UX from SMTP-transport mail.** Because the actual
block/justify rendering happens in the gateway (not the proxy), a test plan for a
plugin-visible feature should assert against the proxy's `scan_result` JSON at the SMTP Proxy
layer, and treat the gateway's page-rendering behavior as a separate, EPDLP-owned concern — don't
conflate "proxy returned the right verdict" with "user saw the right page," they're different
systems.

Confirmed ports (from the pod's internal endpoints, not customer-facing):

| Port | Path | Purpose |
|---|---|---|
| `7999` | `/status` | Legacy health endpoint, must stay unchanged (see "Health check service") |
| `3200` | `/health`, `/stats` | New connectivity-aware health + stats endpoints |
| `8000` | `/api/v1/email/send` | Outlook Plugin REST ingestion |
| `9000` | — | Must not be externally exposed (role not confirmed beyond that) |

**Test implication:** any feature that changes tenant identification, header handling, or the RT
policy/DLP pipeline should be checked against this path too, not just SMTP ingestion — it shares
the pipeline but not the transport, so transport-level assumptions (e.g. "every message came in
over an SMTP session", "actions only ever change headers") don't hold here.

## Mail Sending Application (MSA) types

| MSA type | Config path | Identity/limits |
|---|---|---|
| Microsoft O365 Exchange | Settings > Security Cloud Platform > SMTP > Exchange card | Built-in, ID 0/1/2 reserved. Tenant ID (Azure AD Directory ID) + Next Hop required. Connector configured in Exchange admin center (Mail Flow > Connectors), direction "O365 to Partner Org", triggered by a transport rule, bypassed via `x-netskope-inspected: true` exception |
| Google Gmail | Settings > Security Cloud Platform > SMTP > Gmail card | Built-in. Next Hop = `smtp-relay.gmail.com:587`. Google Admin Console: Hosts route + Content Compliance rule matching absence of `x-netskope-inspected` header |
| Custom MSA | Settings > Security Cloud Platform > SMTP > + > Create Custom MSA | Up to 3 per tenant, custom IDs 3+, highest-ID evicted when full. Optional Tenant Verification key-value pairs, optional Source IP allow list (individual IPs or /24–/32 CIDR, up to 12 entries) |

Every MSA type shares: domain config (domain/subdomain/wildcard, e.g. `*.abc.com`; public domains
like `*.gmail.com`/`*.onmicrosoft.com` rejected), and the rule "configure every MAIL FROM domain,
or mail from an unconfigured domain is rejected."

### Tenant identification methods

The proxy must determine which tenant an inbound connection belongs to before anything else can
happen. Multiple methods exist and can be mixed across tenants on the same domain (NPLAN-3336,
NPLAN-6151):

| Method | How it works | Notes |
|---|---|---|
| SNI-based | TLS Server Name Indication at connection time | Default/baseline method |
| Domain-based | MAIL FROM domain matched against configured tenant domains | Falls back here when source-IP identification is disabled |
| Source-IP-based | `srcip-identification` feature flag; source IP mapped to tenant | Falls back to domain-based (historically "r106") when the feature is disabled or traffic doesn't match |
| Custom-header-based | Tenant Verification key-value header pairs configured per MSA, matched against incoming message headers | Supports "same domain, different tenant" scenarios (multiple tenants sharing one domain, disambiguated by header); mismatched/missing header value is a distinct negative case from a matching one |

**Test implication:** a change to any one identification method must be checked for interaction
with the others — the matrix of "2 tenants, same domain, different identification methods enabled"
is a real, previously-tested scenario (NPLAN-6151), not a hypothetical edge case.

### Domain validation via DNS

Domains configured on any MSA type go through DNS-based ownership verification, not just format
validation (Nplan-6653): the proxy generates a verification token per domain (POST request per
app type — MS-O365/Gmail/Custom), which the customer publishes in DNS; a follow-up token query
confirms ownership. This enforces domain **uniqueness** across a tenant's MSAs and is where the
"max 3 custom MSAs per tenant" limit is also enforced (a 4th custom MSA's domain-verification
request itself fails, not just UI creation).

## Real-time Protection (RT) policy for Email Outbound

Configured at Policies > Real-time Protection > new Email Outbound policy.

- **Source**: users / user groups / OUs. Blank = all users.
- **Destination**: Microsoft O365 Exchange and/or Google Gmail.
- **Activity**: Send (with optional constraints via Activities & Constraints dialog).
- **DLP profile(s)**: one or more, each can carry its own action ("Set action for each profile")
  or a shared action.
- **Actions**:
  - `Allow` — permit
  - `Alert` — log/notify only, feeds Incidents (requires a Forensics Profile too)
  - `Add SMTP Header` — inject a custom header on match (relies on an upstream MTA rule, see below)
  - `Remove Recipients` (R128+) — strip only the restricted recipients from the RCPT list, deliver
    to the rest
  - `Add Traffic Action` — secondary action fired when the selected profiles do NOT match
- Email notifications to senders are configurable (frequency + recipients) independent of the
  in-UI alert.

## DLP scanning

Two independent inspection surfaces (intentionally separated to avoid duplicate matches):

- **Metadata inspection**: SMTP header fields — to, from, cc, attachment (presence/name, not body
  content)
- **Content inspection**: subject line + email body

Because email addresses/names can appear in body text of a thread, profiles like
GDPR/GDPR-narrow/DLP-PII can match on **both** surfaces independently for the same message —
expect multiple violations per message when both metadata and content options are enabled on the
same custom rule (observed: 3 violations on one Exchange test message).

### DLP scanning execution modes — synchronous vs asynchronous ("SnowyOwl")

DLP scanning is not always inline/synchronous. A tenant can be configured for **asynchronous DLP**
(internal codename "SnowyOwl"), which changes the scanning architecture significantly:

```
SMTP Proxy
   │ payload < 1 MB (staged-config threshold) → sent directly to scan handler
   │ payload ≥ 1 MB → uploaded to Ceph (object storage), a contentLink is sent instead
   ▼
Async-DLP scan handler — POST /v1/inspections/jobs
   → returns a Job ID (request includes client_info: SMTP settings info, notification field,
     content field: file info + encryption details + is_inline_email flag)
   ▼
Result handler — GET /v1/inspections/jobs/{job_id}
   → polled/notified for verdict; event-generator publishes the result back to the proxy
   → GET /v1/health confirms scan-handler/result-handler/redis/event-generator liveness
```

Key behaviors:
- **Fallback header on any handler error or timeout** — every documented failure mode (413/400/404/409/500/503
  from either the scan handler or result handler, or scan-timeout) results in a fallback header being
  added to the email rather than the message being dropped or hung. This is the same "add a header
  instead of failing" pattern as the DLP Fallback action below, applied to the async path specifically.
  - Verdict semantics: even if the event-generator/result-handler goes down and comes back up,
    already-submitted jobs must still resolve when it recovers — email delivery is not blocked, and
    already-purged job IDs still need a deterministic outcome.
- **Multi-MSA and secondary-next-hop tenants** are explicitly supported — async DLP behavior must
  be verified per-MSA when a tenant has more than one configured.
- **Health of the pipeline is observable** via metrics: request failures (by error type), requests
  sent, jobs acknowledged, verdicts received, scan-timeouts, revisions encountered, and health
  checks of the scan-handler, result-handler, redis node, and event-generator node are each their
  own metric.
- **Policy semantics still apply the same way** on top of async DLP — "Continue Policy Evaluation
  After Match" (single policy and across multiple policies/policy groups), exclusion lists (user/
  group excluded from DLP entirely — proxy must skip the scan request, not just skip the action),
  and Alert-triggers-UI-alert all behave the same as the synchronous path.

**Test implication:** any DLP-scanning feature must be considered against *both* execution modes —
a behavior verified only under synchronous scanning is not verified for async tenants, and vice
versa. The 1 MB Ceph-upload threshold is itself a boundary worth testing directly.

### DLP Fallback action (error handling)

Separate from the async-specific fallback above, there's a general-purpose `dlp-fallback-add-header`
tenant feature flag governing what happens when DLP scanning itself cannot complete for any reason —
not just async-path errors. Documented failure strings: email larger than the size limit, DLP
scan timeout, DLP scan could not be performed (system error/process down), DLP profile can't be
loaded/missing. When enabled, these produce a fallback header instead of blocking; regression
checks explicitly exist for "no regression on smtpproxy for DLP profile actions in regular
scenarios" — i.e. this flag must not change behavior for the non-error happy path.

## Custom header injection & upstream MTA integration

Two distinct kinds of header get added, on two different triggers:

1. **`x-netskope-inspected: true` — baseline, always-on.** Added to **every** email SMTP Proxy
   inspects, regardless of whether any RT policy/DLP profile matches. Purpose: loop prevention —
   the customer's own MTA/Gmail/Exchange rule must be configured to skip re-routing a message back
   through the proxy when this header is already present. Both the Gmail and Exchange configuration
   guides call this out as a required exception rule.
2. **Customer-defined header(s) — only when Action = "Add SMTP Header".** These are added in
   addition to `x-netskope-inspected`, and only fire when the matching RT policy's action is
   specifically "Add SMTP Header" (not Allow/Alert/Remove Recipients/Add Traffic Action). The
   customer supplies their own header name/value pair(s) when creating the policy — there is no
   single canonical header name to test against. Example pair a customer might configure:
   - `X-Netskope-Action: Block` — a static string value, chosen by the customer
   - `X-Netskope-Policy: {{NS_DLP_PROFILE}}` — the value can be a **template placeholder** that
     Netskope substitutes at match-time; `{{NS_DLP_PROFILE}}` resolves to the name of the DLP
     profile that matched
   - Only `{{NS_DLP_PROFILE}}` is confirmed so far as a substitutable placeholder. Whether other
     placeholders exist (e.g. for action taken, policy name, tenant, matched rule) is not yet
     documented here — treat any other `{{...}}`-style value seen in a Design Spec as something to
     confirm with the user/Design Spec author rather than assume.
   - Any customer-defined header still requires **two-sided** configuration: (1) the header
     name/value defined on the RT policy in Netskope, and (2) a matching rule in the upstream MTA
     to *act on* that header (e.g. route/hold based on it). Configuring only the Netskope side has
     no downstream effect.
   - **Confirmed real header names in use, from the test suite's actual code** (not just the
     illustrative example above): `X-Netskope-Action`, `X-Netskope-Policy`, `X-Netskope-DLP-Action`,
     `X-Netskope-Notification`. These are examples of what customers configure, still free-form —
     don't treat this as an exhaustive or fixed list.
   - **Fallback header is a distinct, separate header**: `X-NETSKOPE-DLP-FALLBACK` /
     `x-netskope-fallback-action` — added by the DLP Fallback action (see below) on scan failure,
     not by a customer-configured "Add SMTP Header" policy. Don't confuse the two when writing
     assertions — a fallback-path test should check for this header, not a customer-defined one.
   - Other confirmed Netskope-internal headers seen in the pipeline: `x-netskope-pop`,
     `x-netskope-tenantid`, `x-netskope-tenant-home-pop`, `X-Netskope-Terminal` — these are
     infrastructure/routing headers, not customer-facing policy output; mention only if a Design
     Spec specifically touches routing/pop/tenant-header logic.

**Test implication:** a feature touching header injection must verify (a) `x-netskope-inspected`
is present on every inspected email regardless of policy match, (b) the customer-defined header
only appears when Action = "Add SMTP Header" and is absent for every other action, and (c) any
template placeholder in the customer-defined value resolves to the correct runtime value (e.g. the
actual matched profile name for `{{NS_DLP_PROFILE}}`), not the literal placeholder text.

## Where results surface (UI)

| Surface | Path | Notes |
|---|---|---|
| Alerts | Skope IT > Alerts | Filterable by app (O365/Gmail). `to_user` is a `like`-match field (SMTP header, may contain multiple recipients); `smtp_to` is the exact-match SMTP-envelope field. Side panel shows sender/recipient, app instance ID, matched DLP profiles, body content classification, Message Size (body+attachments), User (envelope sender), From User (header sender), To User (header recipients, ≤1KB), SMTP To (envelope recipients) |
| Application Events | Skope IT > Events and Alerts > Application Events | Same underlying event stream as Alerts; non-DLP-triggering events land here |
| DLP Incidents | Incidents > DLP | Requires: Forensics Profile configured + Email Outbound RT policy with a DLP profile set to `Alert`. Incident detail shows Activity, Violation, Action Taken, Sensitivity Label / Sensitivity Label Instance. "Acting User" and "From User" are always identical for SMTP proxy incidents |
| Subject Line (cross-cutting) | Alerts / Application Events / Incidents | See `feature_matrix.md` — "Subject Recording and Search" |

## Cross-cutting detection & enrichment features

These layer on top of the core scan/policy pipeline rather than replacing any part of it — each is
its own feature flag and each can interact with the others on the same message:

| Feature | What it does | Key config | Interaction notes |
|---|---|---|---|
| GeoIP enrichment | Adds source/destination geo info to SMTP events | `nsgeoip.cfg`, primary service + secondary fallback | Enriches both source (sender) and destination (including secondary next-hop destination) independently — a fallback from primary to secondary GeoIP service is itself a tested path |
| MIP/DRM sensitivity labels | Detects Microsoft Information Protection sensitivity labels (and sublabels) on the email or its attachment, encrypted or unencrypted, and applies the DLP policy's Alert action accordingly | Works across Exchange→Gmail routing and multiple languages | Email and its attachment can carry **different** sensitivity labels — both must be evaluated, not just one |
| Self-addressed email detection | Similarity score between `mail_from` and recipient (to/cc/bcc) name+address; flags likely self-exfiltration | `self-addressed-email-detection` flag, `threshold-similarity-score` (default 80%) in `feature_config_smtp.json`, `email_similarity_config.json` | Phase 2 adds this as a first-class SkopeIT filter/column (Alerts/Events), including Advanced Query suggestions — a UI surface, not just a backend score |
| Machine-generated email detection | Classifies outbound email as machine-generated via a BOT score (high/medium/low/below-low thresholds) | `smtp_machine_traffic_config.json` (`detection_confidence_threshold`, per-rule enable/disable), signals include RFC 3834 `Auto-Submitted` header and automation-pattern `Message-ID` values | Rules can be individually disabled — a disabled rule must not contribute to the BOT score at all, not just be down-weighted |

## Health check service

A dedicated `healthcheckd` container runs inside the `smtpproxy` pod (ENG-809208), separate from
the main proxy process:

| Endpoint/Port | Purpose |
|---|---|
| `/status` (legacy) | Must keep returning 200 OK unchanged — explicitly a no-regression requirement |
| `/health` (new, case-insensitive path) | Returns 200 only when `healthcheckd` confirms active internet connectivity to configured external sites; reflects failure state if any become unreachable |
| Port 9000, Port 3200 | Must NOT be exposed externally |
| Port 7999 | Health endpoint for the API gateway (Outlook Plugin path); rate-limited (>5 req/s triggers the Nginx rate-limiter) |
| Port 8000 | Outlook Plugin REST ingestion (see above) — not a health port, don't confuse with 7999 |

Config changes to health check behavior are staged: they are not live immediately after
deployment, requiring a config push **and** a pod restart to take effect.

## Known architectural gotchas (apply when reasoning about impact)

- Test-connection validation between O365 and Netskope is documented to "fail with an error even
  though the connection is successful" — expected, not a real failure signal. Don't flag this as a
  bug in generated tests.
- `to_user` vs `smtp_to` are genuinely different fields (header vs envelope) — any feature touching
  recipient display/filtering must treat them distinctly.
- Metadata vs content DLP inspection are separate match paths — a feature that touches one does not
  automatically affect the other, but a feature that touches "how a profile evaluates a message"
  likely affects both.
- **"Continue Policy Evaluation After Match" and "Traffic Action" are mutually exclusive** on a
  policy — a feature touching either must verify the other is correctly disabled/hidden, not just
  that its own option works.
- **Multiple headers configured with the same key get overwritten, not duplicated** — "no duplicate
  headers in email" and "header value can be overwritten if same header is passed in UI" are both
  existing, deliberate behaviors for the Add SMTP Header action.
- **Constraint Profiles can't be deleted while referenced** by an SMTP Proxy setting (e.g. Record
  Subject Line's sender constraint, or Remove Recipients' "To user" constraint) — any feature that
  introduces a new constraint-profile reference point must add this same delete-guard, or it's a
  regression against existing behavior.
- **"Remove Recipients" and "Delete Recipients"** are the same action referred to by two names
  across sources (public docs say "Remove Recipients", the internal test suite says "Delete
  Recipients action" / feature flag `smtp_remove_recipients_enabled`) — treat them as one feature;
  use whichever name the Design Spec at hand uses.
- **SMTP Settings page has coexisting "old modal" and "new modal" UI** for at least secondary
  next-hop configuration (both are actively tested: `TestSMTPProxySettingsPageOldModal` and
  `TestSMTPProxySettingsPageSecondaryNexthopNewModal` in the real test suite) — a mid-migration
  state. A UI feature touching SMTP Settings must clarify which modal version it targets, or
  verify both, rather than assuming only one exists.
- **Tenant identification via custom headers has a full WebUI component**, not just the backend
  API described above — a dedicated "Custom Tenant Identification" settings page exists with its
  own secondary-next-hop and disable/enable variants. A backend-only view of this feature misses
  real UI surface area.
