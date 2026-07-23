# SMTP Proxy — Existing Feature Matrix

Use this alongside `product_architecture.md` in Phase 1.5 to identify which existing features a
new Design Spec's feature is likely to interact with. This is deliberately about **product surface
area**, not test mechanics (see `../kb/` for that).

Source: the team's real TestRail suite (`SMTP Proxy`, suite ID S926 — 1,718 cases across ~40
feature areas as of this writing), the actual automation code in
`nsproxy/tests/api/SMTP/smtp_proxy` and `nsproxy/tests/ui/SMTP` (confirms/corrects real header
names, endpoints, ports, and config paths against what TestRail titles imply), plus public docs
(`docs.netskope.com/en/netskope-smtp-proxy`) for baseline architecture framing. Ticket IDs in
feature names (NPLAN-/ENG-) are the internal reference — use them to look up the original Design
Spec if you need more than this summary.

Some very small or purely-internal areas (Watson debug logging, generic bug-tracking entries) are
folded into notes on a related row rather than given their own row — they're not independent
product surfaces.

---

## A. Core Policy & Access Control

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 1 | RT policy for Email Outbound (core) | Source/destination/activity/DLP-profile → action policy engine; users/groups can be added, edited, removed from a policy | MSA/relay config (destination) | Every action below, DLP scanning, Alerts/Incidents, Constraint Profiles | Changing user/group evaluation semantics affects every policy already relying on source targeting |
| 2 | WebUI — RT Policy page behavior | UI-level validation: action dropdown options, DLP profile add/delete reflected in config-service DB, SMTP header text box + "Enter valid key value pair" validation, Save/Cancel button state, constraint display rules for Send/All activity | Core RT policy | Add SMTP Header action, Constraint Profiles, config-service sync | UI validation regressions here surface as silent policy-save failures, not crashes — easy to miss without a config-service DB check |
| 3 | Constraint Profiles | Reusable named profiles (e.g. "To user", sender constraints) referenced by policies and by other SMTP settings (Record Subject Line, Remove Recipients) | RT policy, Subject Recording, Remove Recipients | Any feature that references a constraint profile | **Deletion must be blocked while referenced anywhere** — any new feature that adds a constraint-profile reference point must add this same delete-guard, or it's a regression |
| 4 | Feature flag provisioning (Provisioner + Feature Enablement API, NPLAN-4724) | Two paths to toggle tenant-scoped feature flags: generic provisioner API (global-flag-gated) and the SMTP-specific `SMTPFeatureEnablementAPI` (get/set per flag, schema validation, add-to-empty-config, edit-and-add) | — | Every feature gated by a flag in `../kb/smtp_proxy_kb_api.md` §3 | A provisioning-path bug silently affects every flag-gated feature at once — test both paths independently for any new flag |
| 5 | Role-Based Access Control for SMTP pages (`Moved_From_SWG`: Email Notification, SMTP settings, Email DLP, Mail Relay, Cloud App, Other policies) | None/View/Manage/Manage+Apply role permissions gate visibility and edit rights across every SMTP-related page and the Policy Groups feature that scopes them | Netskope's general RBAC/Policy Groups system | Every UI surface listed elsewhere in this matrix | A new SMTP UI surface that doesn't respect the existing role matrix (None/View/Manage) is a regression against an established, heavily-tested pattern (76+ cases just for Email DLP RBAC) |

## B. MSA & Domain Management

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 6 | MSA configuration & landing page (Onprem MSA support) | Unified landing page for Exchange/Gmail/Custom MSA create/edit/delete; blocks delete while an RT policy references the MSA; reserved custom-key names (`mailfrom`, `guid`) rejected | Domain validation, RT policy | Alerts/Events/Incidents "Application" field (must show Custom MSA name), Source IP allow list | Any MSA-page change must re-verify the delete-guard against RT-policy references and the reserved-key rule |
| 7 | Wildcard domain support | Domain field accepts exact/subdomain/wildcard (`*.abc.com`) across all MSA types, with a defined precedence rule when both a wildcard and a more specific non-wildcard domain are configured | MSA config | Domain validation via DNS, tenant identification (domain-based) | Precedence-rule regressions are subtle — "more specific wins" must hold under every MSA type, not just the one being changed |
| 8 | Domain validation via DNS (Nplan-6653) | DNS-based ownership verification: token generated per domain per app type (O365/Gmail/Custom), customer publishes it, proxy confirms via token query; also where the 3-custom-MSA limit and cross-tenant domain-uniqueness are enforced | MSA config | Wildcard domain support, tenant identification, Custom MSA limit | A 4th custom MSA's *verification request itself* must fail, not just block at the UI-creation step — don't only test the UI-level limit |
| 9 | Multiple next hops / secondary next hop (NPLAN-195) | Optional secondary Next Hop + port per domain, independently configurable per MSA type; auto-defaults port to 25 if only next-hop is given; supports both per-domain and shared secondary config; UI has a coexisting "old modal" and "new modal" (mid-migration — see product_architecture.md gotchas) | MSA config | GeoIP (secondary-next-hop destination enrichment), async DLP (secondary-MSA behavior) | Secondary-hop failover interacts with GeoIP destination enrichment and async-DLP per-MSA behavior — test together, not in isolation; a UI change must be checked against both modal versions |
| 10 | Custom MSA source IP allow list | Restricts which mail-server egress IPs are trusted per Custom MSA; synced to load balancer + `smtp-pyipfirewall` container as an ipset | Custom MSA config | ipset propagation (`../kb/smtp_proxy_kb_ui.md` §17.7), tenant/domain uniqueness | Allow-list changes risk silently blocking legitimate mail if propagation timing regresses |

## C. Tenant Identification

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 11 | Source-IP-based tenant identification (Nplan-3336) | `srcip-identification` flag; maps source IP to tenant; falls back to domain-based identification when disabled or non-matching | MSA/domain config | SNI-based and domain-based identification (product_architecture.md), custom-header identification | Falls back silently to a different method when disabled — a regression here can misroute mail to the wrong tenant without an obvious error |
| 12 | Tenant identification via custom headers (NPLAN-6151) | Tenant Verification key-value header pairs per MSA disambiguate multiple tenants sharing one domain; staged-config-gated (`allow-tenant-matching-with-header`); has its own dedicated WebUI page ("Custom Tenant Identification" settings), not just a backend API | Custom MSA config, staged config flags | Source-IP identification, wildcard domains (tested together explicitly), its own WebUI settings page (secondary-next-hop + enable/disable variants) | The largest single test area (111 backend cases + 67 dedicated WebUI cases) — any tenant-identification change must be re-checked against the "same domain, multiple tenants, mixed flag states" matrix already covered here, on both backend and UI |
| 13 | GeoIP integration | Enriches SMTP events with source and destination geo info independently; primary GeoIP service with secondary fallback | `nsgeoip.cfg` | Secondary next-hop (destination enrichment), Alerts/Events fields | Primary→secondary fallback is itself a tested path — don't assume enrichment always uses the primary service |

## D. DLP Scanning & Policy Actions

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 14 | Flexible email scan — metadata vs content selection (nplan-2068) | Lets a DLP rule choose which surface(s) to inspect (metadata-only, content-only, or both) rather than always scanning everything | DLP scanning (product_architecture.md) | Subject Recording (content path also feeds it), metadata/content double-fire behavior | A rule that appears to "not match" may simply be scanning the wrong surface — verify the selection itself, not just the match outcome |
| 15 | Alert & Continue policy evaluation (NPLAN-534) | "Continue policy evaluation after match" and multi-policy/multi-profile match ordering (R1/R2, P1/P2 combinations) | Core RT policy | Add Traffic Action (mutually exclusive with Continue), Alerts | "Continue" and "Traffic Action" are mutually exclusive on a policy — verify the other is correctly disabled, not just that the enabled one works |
| 16 | Add SMTP Header action — customer-defined headers | Customer supplies header name/value pairs at policy-creation time, only when Action = "Add SMTP Header"; values can use the `{{NS_DLP_PROFILE}}` template placeholder. Confirmed real header names in use: `X-Netskope-Action`, `X-Netskope-Policy`, `X-Netskope-DLP-Action`, `X-Netskope-Notification` | RT policy match, upstream MTA rule for that header | `x-netskope-inspected` (always-on, separate trigger — see product_architecture.md), DLP profile matching, DLP Fallback action's `X-NETSKOPE-DLP-FALLBACK` (separate header, don't confuse) | Same-key headers overwrite rather than duplicate — a change here must preserve that, plus resolve `{{NS_DLP_PROFILE}}` to the real profile name, never the literal placeholder |
| 17 | Delete/Remove Recipients action (NPLAN-5753) | Strips only restricted recipients (via a "To user" Constraint Profile), delivers to the rest; hidden from Traffic Action policies; requires a DLP profile + Email Outbound policy to appear at all | RT policy, Constraint Profiles, DLP profile selection | Multi-recipient handling (`../kb/smtp_proxy_kb_api.md` §7.6, error 558), Constraint Profile delete-guard | Removing/editing the Constraint Profile must also remove the action from the policy — this cascade is existing, tested behavior |
| 18 | DLP Fallback action (`dlp-fallback-add-header`) | On DLP-scan failure (oversized email, timeout, system error, missing/unloadable profile), adds the `X-NETSKOPE-DLP-FALLBACK` / `x-netskope-fallback-action` header instead of blocking or hanging | DLP scanning (sync or async) | Async-DLP fallback-on-error (same pattern, different trigger), SkopeIT `Dlp_scan_failed` filter | Explicit no-regression requirement: this flag must not change behavior for the *non-error* happy path; verify the fallback header name specifically, not just "a header was added" |
| 19 | Async DLP scanning ("SnowyOwl" integration) | Job-based async scanning pipeline (scan handler → Ceph for large payloads → result handler → verdict); see product_architecture.md for the full flow | RT policy, DLP profiles, Ceph object storage | DLP Fallback action, multi-MSA tenants, secondary next-hop, Outlook Plugin ingestion | Largest and newest architecture change in the suite (141 cases across 4 sub-areas) — any DLP-pipeline feature must be verified under async as well as sync scanning |
| 20 | Large File Support / LFS (NPLAN-1163) | Enforces a configurable max email size (default 32MB, up to 64MB), with DLP scan timeout (90s) handling | `dlp-settings` feature flag | DLP Fallback action (oversized-email fallback), async DLP (Ceph upload threshold is a separate, smaller 1MB boundary — don't conflate the two) | LFS's 32/64MB boundary and async DLP's 1MB Ceph-upload boundary are two different limits on two different paths — a size-related change must be tested against both |
| 21 | DKIM verification | Signing mode combinations (simple/simple, relaxed/simple, simple/relaxed, relaxed/relaxed) crossed with header-whitespace preservation on/off and presence/absence of spaces around header colons | `../kb/smtp_proxy_kb_api.md` §3.4, §6.7, §7.4 | Header injection/preservation logic generally | Whitespace-preservation is a generic header-handling flag — a header-processing change anywhere risks silently breaking DKIM canonicalization even if DKIM itself wasn't touched |
| 22 | Data type & format edge cases | Adversarial/malformed payloads: 10K+ char subjects, header spoofing, homoglyph/Unicode tricks, HTML obfuscation, multi-layer encoding, RFC822 nested mail, bad MIME boundaries, null-byte/control-char injection | Core parsing pipeline | Every feature that reads subject/header/body content (Subject Recording, DLP content inspection, Machine-Generated detection) | These are parser-hardening regressions, not feature regressions — any change to header/body/subject parsing should be re-run against this whole suite, not just the feature's own new cases |
| 23 | MIP/DRM sensitivity label integration (NPLAN-4004) | Detects Microsoft Information Protection labels/sublabels (encrypted or not) on the email or its attachment and applies the Alert action; works across Exchange→Gmail routing and multiple languages | DLP profile Alert action | Async DLP (labels must be evaluated under both scan modes), attachment handling | Email and attachment can carry *different* labels — a fix that only checks one is incomplete |

## E. Detection & Enrichment

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 24 | Self-addressed email detection (NPLAN6261, + Phase 2) | Similarity score between `mail_from` and recipient name+address flags likely self-exfiltration; default threshold 80%, configurable; Phase 2 adds a first-class SkopeIT filter/column including Advanced Query | `feature_config_smtp.json`, `email_similarity_config.json` | SkopeIT filter pattern (`../kb/smtp_proxy_kb_ui.md` §18.7), Data exfiltration scenarios (SMTP Solution testing row) | Setting the threshold to 0 must be rejected, not silently accepted — an existing, explicit negative case |
| 25 | Machine-generated email detection (NPLAN-4895) | BOT score classification (high/medium/low/below-low) using RFC 3834 `Auto-Submitted` header and automation-pattern `Message-ID` values; per-rule enable/disable in `smtp_machine_traffic_config.json` | Feature flag + config file | Data type/format edge cases (header parsing), Subject/header inspection generally | A disabled rule must contribute **nothing** to the BOT score, not just be down-weighted — existing explicit test for this |

## F. Session, Protocol & Resiliency

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 26 | Connection reuse / multi-email-per-TCP (NPLAN-5173) | Multiple emails processed per TCP session; DNS resolution per back-connection; graceful handling of RSET/QUIT/HELO/EHLO sent in quick succession; behavior under back-connection network/DNS errors | `../kb/smtp_proxy_kb_api.md` §3.1/§3.2 (front/back-conn persistence flags) | DLP + LFS + Subject Recording all tested together per-connection (i.e. "1 email/TCP with DLP+LFS+record subject search" is a real combined case) | New per-message features must be verified to still work correctly when multiple messages share one TCP session, not just in isolation |
| 27 | TLS version interoperability | SMTP Proxy accepts TLS 1.2/1.3 and rejects TLS 1.1/SSLv3 via STARTTLS, independently on both the client-facing and next-hop-facing sides | — | Connection reuse, MSA next-hop config | Client-side and next-hop-side TLS enforcement are tested and must be verified separately — a fix on one side doesn't imply the other |
| 28 | SMTP Proxy service resiliency (protocol-level) | Oversized headers (557), oversized payloads (552), >1000-recipient fan-out (550/452), TCP half-close, out-of-order commands (503), premature dot-terminator injection, multilingual/multi-byte payloads, SMTPUTF8, null-sender bounce (`<>`), duplicate MAIL FROM | Core SMTP protocol handling | Data type/format edge cases, connection reuse | These are protocol-conformance regressions — treat as a standing regression suite to re-run whenever the SMTP command/response handling layer changes at all |
| 29 | Health check service (ENG-809208) | Separate `healthcheckd` container; `/status` (legacy, must stay unchanged) vs `/health` (new, connectivity-aware) endpoints; ports 9000/3200 must stay unexposed; port 7999 rate-limited for the API gateway | See product_architecture.md "Health check service" | Outlook Plugin (shares the API-gateway path), deployment/staged-config rollout | Health-check config changes are staged and need a pod restart — a test that doesn't restart the pod will falsely pass/fail |

## G. Observability

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 30 | SMTP DEM + Metrics Pusher (NPLAN-2991, Phase 1 & 2) | Per-tenant per-metric-type counters pushed to GEF at a configurable publish interval; resets to 0 only on successful send; separate pub-sub-lite pipeline aggregates metrics by [metric name + POP] for synthetic monitoring | Staged config, `eventforwarder-mp-grpc` endpoint | Async-DLP metrics (separate metric set, same observability philosophy) | Metrics must NOT reset to 0 on a failed send — an existing, explicit invariant |

## H. Alternate Ingestion & Cross-Feature UI Surfaces

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 31 | Outlook Plugin support — REST ingestion (NPLAN-6393) | Mail submitted via REST API (`POST /api/v1/email/send`, port 8000) instead of SMTP, arriving via a separate `epdlp-mp-gateway` pod (not the customer's own MTA); requires `x-ns-tenant-id`/`x-ns-app-name`/`x-plugin-version` headers; goes through the same policy/DLP pipeline (sync and async); always returns HTTP 200 with a `scan_result` JSON — the gateway, not the proxy, renders any block/justify page to the end user. RT policies must explicitly target "Outlook Plugin" as a destination — an MSA-targeted policy does not automatically apply | Core RT policy/DLP pipeline, health-check ports 7999/3200 | Async DLP (explicitly tested together), tenant identification (via required headers, not SNI/domain/IP), EPDLP gateway pod reachability, RT policy destination targeting | The single largest feature area (240+ cases across 4 files, ~172 test functions in code) — any pipeline change (policy, DLP, header handling, action semantics) must be re-verified on this ingestion path; verify `scan_result` at the proxy layer, don't conflate it with gateway-side page rendering |
| 32 | Subject Recording and Search (NPLAN-1160) | Persists up to 1000 chars (UTF-8) of subject line for Email-Outbound-scanned messages, searchable in Alerts/Events/Incidents; optional sender-only Constraint Profile scoping | Email Outbound RT policy, Constraint Profiles | Content DLP inspection (reads the same subject field), Alerts/Events/Incidents Subject Line column | Constraint Profile delete-guard applies here too; subject/header parsing changes risk corrupting recorded subjects even when scanning itself is unaffected |
| 33 | Alerts / Application Events / DLP Incidents views | Lists policy-match events and forensic detail; Incidents additionally now carries a validated `incident_id` field (ENG-577204) | RT policy + DLP scan producing a match; Incidents also needs a Forensics Profile | `to_user`/`smtp_to` field semantics, Message Size, Subject Line, matched DLP profiles, Self-Addressed/Machine-Generated filters | UI column/field changes risk breaking existing filters (`like` vs `=` semantics on `to_user` vs `smtp_to`); Acting-User/From-User equality is a regression signal specific to SMTP proxy incidents |

## I. End-to-End / Solution-Level Scenarios

| # | Feature | What it does | Depends on | Shares architecture with (interaction candidates) | Regression risk if changed |
|---|---|---|---|---|---|
| 34 | SMTP Solution / integration scenarios | Cross-feature scenarios exercised together: full customer onboarding (domain validation → DKIM → TLS → DLP all active), data-exfiltration patterns (self-addressed, MIP-labeled, base64-encoded PII, large attachment), multi-domain/multi-policy enterprise setups, domain-removed-mid-test, tenant-identification-via-source-IP with multiple tenants (no context bleed), overlapping org+department policy hierarchies | Nearly everything above | Everything above | This row is a reminder, not a dependency map: **when a new feature could plausibly appear in one of these combined scenarios, add a scenario-level regression case, not just a unit-level one** — most real regressions in this suite show up at the combination level |

---

## How to use this in Phase 1.5

1. From the Design Spec, identify which architecture component(s) (`product_architecture.md`) the
   new feature is changing.
2. Scan the "Shares architecture with" and "Regression risk if changed" columns above for every row
   whose component overlaps. Use the category headers (A–I) to narrow down quickly before scanning
   row-by-row.
3. Anything found is a candidate for the Confluence "Dependency features and Regression Impact"
   section and, if the user opts in, a `[REG]`-tagged regression test case.
4. If the new feature doesn't clearly map to any row here, say so plainly rather than forcing a
   match — not every feature has a predecessor to regress against.
5. If the new feature looks like it could show up in a combined/end-to-end scenario (row 34), flag
   that explicitly — combination-level regressions are where most real issues in this product surface.

## Keeping this current

This matrix reflects the team's real TestRail suite (`SMTP Proxy`, S926) as pulled and summarized
on 2026-07-23, plus the public docs. When a new feature ships (i.e. every time this skill is used
end-to-end for a new Design Spec), add a row for it here once its test plan is done — that's how
future features get correctly flagged as regression candidates. If the underlying TestRail suite
changes significantly, re-pull and re-summarize rather than hand-patching this file indefinitely.
