# Test Plan: SMTP Proxy — Machine Generated Emails

**Feature:** machine_generated_emails
**Component:** SMTP Proxy
**Author:** Ramegowda K
**Date:** 2026-06-30
**PRD Source:** Confluence
**Status:** Draft

---

## 1. Overview

The Machine Generated Emails feature adds two new fields to the Alerts, Application Events, DLP Incidents, and Policy Alerts pages to indicate whether an email was machine-generated and the detection confidence level. The SMTP Proxy calculates a BOT score for each email and sets `Machine Email Detected` (Yes/No) and `Machine Email Confidence` (Low/Medium/High) accordingly. The feature is controlled by a tenant-level feature flag `machine-traffic-detection.enabled` with no staged config equivalent.

---

## 2. Scope

### In Scope

- Presence and correctness of `Machine Email Detected` (Yes/No) table column on Alerts page when feature flag is enabled
- Presence and correctness of `Machine Email Detected` (Yes/No) table column on Application Events page when feature flag is enabled
- Side panel display of `Machine Email Detected` and `Machine Email Confidence` under "Other Category:" in the APPLICATION section (Alerts and Application Events)
- Conditional display: `Machine Email Confidence` shown only when `Machine Email Detected = Yes`
- Feature flag disabled: column hidden from both tables; previously-recorded side panel values remain
- All three confidence values: Low, Medium, High
- Filter support for `Machine Email Detected` on Alerts and Application Events pages

### Out of Scope

- API-level BOT score calculation and SMTP Proxy logging (not in scope — UI-only test plan)
- DLP Incidents and Policy Alerts pages (no screenshots provided; covered as open question)
- Performance testing with large volumes of machine-generated emails
- Backend feature flag API correctness testing
- Subject Line feature (already covered separately)

---

## 3. Test Strategy

### Test Types Applied

| Test Type | Rationale |
|---|---|
| Functional | Verify each PRD requirement produces the correct UI field/column behavior |
| Negative | Verify Confidence field is absent when Detected = No |
| Boundary | Verify all three Confidence values (Low, Medium, High) are rendered correctly |
| Integration | Verify end-to-end: email sent through SMTP proxy → event recorded → field visible in UI |

### Automation Approach

| Category | Tool / Method |
|---|---|
| Feature flag toggle | `SMTPFeatureEnablementAPI` → `POST /internal/v1/featureconfigs` |
| Page navigation | `NSSkopeITAlertsPage`, `NSSkopeITApplicationEventsPage` page objects |
| Column assertion | `page.page_pagination_table.get_all_customize_columns()` |
| Filter assertion | `page.multilevel_filter.get_all_first_level_options_text_list()` |
| Side panel assertion | `page.assert_side_panel_key_exist(key=...)` + text parsing under Other Category |
| Traffic generation | `EmailBuilderSmtp.send_email()` via `<SMTP_PROXY_VIP>:25` |

### Test Environment Requirements

| Item | Value |
|---|---|
| SMTP Proxy VIP | `<SMTP_PROXY_VIP>` |
| SMTP Proxy Port | 25 |
| Kubernetes Namespace | `smtpproxyv2` |
| Tenant ID | `<TENANT_ID>` |
| Alerts page URL | `/ns#/alerts` |
| Application Events URL | `/ns#/skopeIT` |

---

## 4. Coverage Matrix

| PRD Requirement / Feature Area | Functional | Negative | Boundary | Integration |
|---|---|---|---|---|
| Machine Email Detected column — Alerts table | ✅ | ➖ | ➖ | ✅ |
| Machine Email Detected column — App Events table | ✅ | ➖ | ➖ | ✅ |
| Side panel: Detected=Yes shows Confidence | ✅ | ➖ | ✅ | ✅ |
| Side panel: Detected=No hides Confidence | ➖ | ✅ | ➖ | ➖ |
| Feature flag disabled hides column | ✅ | ➖ | ➖ | ➖ |
| Previously-recorded values visible after flag off | ✅ | ➖ | ➖ | ➖ |
| Confidence values Low / Medium / High | ➖ | ➖ | ✅ | ➖ |
| Filter support — Alerts | ✅ | ➖ | ➖ | ➖ |
| Filter support — App Events | ✅ | ➖ | ➖ | ➖ |
| Export button — Alerts | ✅ | ➖ | ➖ | ➖ |
| Export button — App Events | ✅ | ➖ | ➖ | ➖ |
| Non-ASCII characters in email | ✅ | ➖ | ➖ | ✅ |

**Coverage totals:**
- P0 tests: 7
- P1 tests: 8
- P2 tests: 0
- Total tests: 15
- Automatable: 15 / 15

---

## 5. Test Cases

> Full detail (Steps + Expected Results) is in the TestRail CSV:
> `machine_generated_emails_testrail.csv`

| # | Test Summary | Category | Priority | Automatable |
|---|---|---|---|---|
| 1 | Verify Machine Email Detected column present on Alerts table (flag enabled) | Functional | P0 | Yes |
| 2 | Verify Machine Email Detected column present on Application Events table (flag enabled) | Functional | P0 | Yes |
| 3 | Verify Alerts side panel: Detected=Yes shows Confidence field | Functional | P0 | Yes |
| 4 | Verify Alerts side panel: Detected=No has no Confidence field | Functional | P0 | Yes |
| 5 | Verify App Events side panel: Detected=Yes shows Confidence field | Functional | P0 | Yes |
| 6 | Verify App Events side panel: Detected=No has no Confidence field | Functional | P0 | Yes |
| 7 | Verify feature flag disabled hides column on both tables | Functional | P0 | Yes |
| 8 | Verify previously-recorded side panel values visible after flag disabled | Functional | P1 | Yes |
| 9 | Verify Confidence shows all three values: Low, Medium, High | Boundary | P1 | Yes |
| 10 | Verify Machine Email Detected filter option on Alerts page | Functional | P1 | Yes |
| 11 | Verify Machine Email Detected filter option on Application Events page | Functional | P1 | Yes |
| 12 | Verify filtering by Machine Email Detected=Yes on Alerts returns correct rows | Functional | P1 | Yes |
| 13 | Verify Export button is present and enabled on Alerts page when Machine Email Detected events exist | Functional | P1 | Yes |
| 14 | Verify Export button is present and enabled on Application Events page when Machine Email Detected events exist | Functional | P1 | Yes |
| 15 | Verify Machine Email Detected and Confidence fields display correctly for email with non-ASCII characters in subject and body | Functional | P1 | Yes |

---

## 6. Test Data

| Data Item | Source | Notes |
|---|---|---|
| Machine-generated email | Sent via `EmailBuilderSmtp` to `<SMTP_PROXY_VIP>:25` | Must trigger BOT score > low threshold for Detected=Yes |
| Non-machine email | Sent via `EmailBuilderSmtp` to `<SMTP_PROXY_VIP>:25` | Must produce Detected=No |
| Low / Medium / High confidence emails | Coordinate with Dev on content/headers | Required for TC-9 |
| Feature flag state | Toggled via `POST /internal/v1/featureconfigs` | Restored to disabled in teardown |
| Tenant ID | `<TENANT_ID>` | Used in `x-netskope-tenantid` header |

---

## 7. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| No test data available that triggers all three confidence levels (Low/Medium/High) | High — TC-9 cannot execute | Coordinate with Dev team to identify email headers/content that deterministically produce each confidence level |
| DLP Incidents and Policy Alerts pages not covered | Medium — PRD lists 4 pages; only 2 tested | Add tests once page objects and field XPaths are confirmed (see Q-2) |
| Feature flag is tenant-specific with no staged config — may not be toggleable in all test environments | High — all tests blocked | Confirm test tenant has flag access before running suite |

---

## 8. Entry / Exit Criteria

### Entry Criteria
- SMTP Proxy deployed and reachable at `<SMTP_PROXY_VIP>:25`
- Feature flag `machine-traffic-detection` toggleable via `POST /internal/v1/featureconfigs`
- At least one SMTP Proxy event with `Machine Email Detected = Yes` and one with `= No` available or generatable

### Exit Criteria
- All P0 tests pass
- All P1 tests pass or have documented known issues with bug tickets
- Test results uploaded to TestRail

---

## 9. Open Questions

- [ ] Q-2 (partial): Are the new fields also visible as table columns or only in side panel on DLP Incidents and Policy Alerts pages? No screenshots provided for those pages.
- [ ] Q-3 resolved: Filter support confirmed in PRD.
- [ ] Q-4 resolved: Flag disabled → column hidden, side panel values retained.
- [ ] Q-5 resolved: Policy Alerts = same `NSSkopeITAlertsPage`.
- [ ] What email headers or body content deterministically produces Low / Medium / High BOT scores? Required for TC-9.
- [ ] Is `Machine Email Confidence` field visible in the DLP Incidents detail view the same way it is in Alerts/App Events side panel (under Other Category)?

---

*Generated by `smtp-testplan-generator` skill from PRD: Confluence*
