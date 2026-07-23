# SMTP Proxy KB — UI Layer

Sections: SMTP Settings Page, Alerts/Events/Incidents Page Patterns, Additional Page Objects & UI Test Areas (Tenant ID, Domain Validation, Self-Addressed, Machine-Generated, combined scenarios, Settings page migration state)
For API/backend patterns → see smtp_proxy_kb_api.md

---

## 17. UI Layer — SMTP Settings Page Tests

Sources: `nsproxy/tests/ui/SMTP/test_settings_smtp.py`, `test_settings_smtp_accessibility.py`

### 17.1 Page Object and Navigation

```python
from webui_libs.pages.settings.security_cloud_platform.smtp.settings_smtp_config import NSSMTPProxySettingPage

ns_settings_page = NSSMTPProxySettingPage(selenium, base_url)
_patch_smtp_page_open(ns_settings_page)   # always required — patches open() for MSA card wait

ns_settings_page.URL_TEMPLATE = "ns#/settings?view=email-relay"
ns_settings_page.open()

ns_settings_page.URL_TEMPLATE = "ns#/settings"
ns_settings_page.open()
```

**Always call `_patch_smtp_page_open(page)` before `page.open()` to avoid stale selectors.**

### 17.2 Nav Bar Assertions

```python
from webui_libs.components.ns_navbar import NsNavbar
Navbar = NsNavbar(driver)
Navbar.assert_item_in_sub_menu("Security Cloud Platform", "MAIL RELAY")
Navbar.assert_item_in_sub_menu("Security Cloud Platform", "SMTP")
Navbar.assert_item_not_in_sub_menu("Security Cloud Platform", "SMTP")
```

Feature visibility depends on `smtp_dlp_enabled` provisioner flag:
```python
provisioner.client_config_flag(feature_name="smtp_dlp_enabled", action=True/False)
provisioner.client_config_flag(feature_name="inline_policy_enhancements_enabled", action=True/False)
```

### 17.3 MSA Cards — Exchange, Gmail, Custom

```python
_click_exchange_icon(selenium)              # CSS: ns-app-card div.ns-app-card-container.ns-app-card-exchange
ns_settings_page.gmail_icon_btn.click()
ns_settings_page.custom_msa_1_btn.click()  # patched — uses JS mouseover + re-fetch
ns_settings_page.edit_icon_btn.click()
ns_settings_page.edit_cancel_btn.click()
open_existing_custom_msa_edit_dialog(selenium, ns_settings_page, msa_name="Test1")
```

Custom MSA cards: no `div.ns-app-card-exchange` or `div.ns-app-card-gmail` inside.

### 17.4 Exchange MSA Edit Modal

```python
_XP_PRI_NH = "(//ns-email-relay-modal//input[@placeholder='Enter IP address or FQDN'])[1]"
_XP_SEC_NH = "(//ns-email-relay-modal//input[@placeholder='Enter IP address or FQDN'])[2]"
_XP_PRI_PT = "(//ns-email-relay-modal//input[@placeholder='Port'])[1]"
_XP_SEC_PT = "(//ns-email-relay-modal//input[@placeholder='Port'])[2]"

ns_settings_page.exchange_second_guid_input.enter_text("<tenant_guid>")
# Invalid GUID error: "Enter a valid Tenant ID"

ns_settings_page.exchange_second_next_hop_input.enter_text("smtp-relay.gmail.com")
# Invalid next hop error: "Enter a valid IP or FQDN" (commas, CIDR, brackets, spaces, double-dots)

ns_settings_page.exchange_second_port_input.enter_text("25")
# Invalid ports: "0", "65536", "" → port error modal

ns_settings_page.edit_save_btn.click()
ns_settings_page.edit_cancel_btn.click()
```

### 17.5 Custom MSA Backend Helpers

```python
_create_custom_msa_via_backend(webui, name="Test1", next_hop="smtp-relay.gmail.com",
    port="25", domain="emailskope.com",
    extra_domains=[{"domain": "emailskope2.com", "next_hop": "smtp-relay2.gmail.com", "port": "25"}])

_delete_custom_msas_by_name_via_backend(webui, "Test1", "msa_1")
```

ID rules: 0/1/2 reserved (built-ins). Custom: ID 3+. Max 3 per tenant. Evict highest-ID if full.

### 17.6 smtpSettings.json Verification (Post UI Save)

Wait `time.sleep(120)` after saving, then:
```python
def get_smtpSettings_kube(nsconfig, k8s_operations):
    _, stack, tenant = nsconfig
    raw = Kube.exec_smtpproxy_shell_command(k8s_operations,
        command=f"cat /opt/ns/tenant/{tenant}/smtpSettings.json")
    return json.JSONDecoder().raw_decode(raw)[0]

resp_json = get_smtpSettings_kube(nsconfig, k8s_operations)
exchange_config = next((c for c in resp_json["configs"] if c.get("app") == "Microsoft Office 365 Exchange"), None)
assert exchange_config is not None
ti = {e["key"]: e["value"] for e in exchange_config.get("TenantIdentification", [])}
assert ti.get("guid") == expected_guid
assert expected_next_hop in json.dumps(resp_json)
```

### 17.7 Source IP Allow List (Custom MSA)

```python
ns_settings_page.custom_msa_allow_source_add_btn.click()
ns_settings_page.allow_ip_element_n_input(i + 1).enter_text("1.2.3.4/24")
# Invalid: /23, /33, 255.255.255.256, IPv6. Valid CIDR: /24–/32
# Error: "Enter valid IPv4 Address or CIDR (24-32)."
ns_settings_page.custom_msa_save_btn.click()
```

ipset verification (container: `smtp-pyipfirewall`, ~10s propagation):
```python
ipset_output = k8s_ops.exec_pod_shell_commands(pods[0], "smtp-pyipfirewall", "ipset -L")
network = ipaddress.ip_network("1.2.3.4/24", strict=False)
expected = str(network.network_address) if network.prefixlen == 32 else str(network)
assert expected in set(ipset_output.splitlines())
```

### 17.8 Domain Validation Rules (UI)

| Format | Result |
|---|---|
| `*.geskope.com`, `yahoo.com`, `*.abc.yahoo.com` | Valid |
| `..geskope.com`, `.geskope..com`, `geskope.com.` | Error: "Enter valid domain name" |
| `*.onmicrosoft.com`, `*.gmail.com` | Error (public domains not allowed) |
| FQDN > 255 chars | Error: "FQDN can be up to 255 characters" |

Tooltip: `"Email domain accepts domains, subdomains, and single wildcard expressions. Public domains are not allowed."`

### 17.9 Record Subject Line Feature

```python
ns_settings_page.edit_record_subject_status(enable=True/False)
ns_settings_page.edit_record_subject_status(enable=True, constraint_profile="profile_name")
ns_settings_page.record_subject_view_modal_status_enabled()
ns_settings_page.record_subject_view_modal_status_disabled()
_patch_record_subject_edit(ns_settings_page)   # required before click_record_subject_edit_btn()
```

Effect: enabled → Subject Line populated; disabled → Subject Line = `""` for SMTP events; non-SMTP always `"Unknown"`.

### 17.10 SkopeIT Subject Line Integration

```python
from webui_libs.components.ns_multilevel_filter import Filter, FilterType
page = NSSkopeITAlertsPage(selenium, base_url).open()
page.page_pagination_table.add_columns(column_names=["Subject Line", "Access Method"])
page.multilevel_filter.add_filter(Filter(first_level_value="Subject Line",
    type_of_multifilter=FilterType.SearchText, second_level_value="test"))
page.multilevel_filter.advanced_query_search("(access_method eq 'SMTP Proxy') and (subject like 'test')")
columns = page.page_pagination_table.get_all_customize_columns()
assert "Subject Line" in columns
page.assert_side_panel_key_exist(key="Subject Line")
```

### 17.11 Accessibility Tests (Keyboard Navigation)

```python
def _tab_until_focused(driver, xpath, max_tabs=150): ...    # TAB until element focused
def _shift_tab_until_focused(driver, xpath, max_tabs=150): ... # SHIFT+TAB backward
def _tab_if_present(driver, xpath, max_tabs=20): ...        # skip if element not in DOM

# Assert pattern
_tab_until_focused(driver, xpath)
assert _is_focused(driver, xpath)
_shift_tab_until_focused(driver, first_xpath)
assert _is_focused(driver, first_xpath)
```

All accessibility tests: `@pytest.mark.nondestructive` + `@pytest.mark.stack_agnostic`

### 17.12 UI Fixtures (feature flag provisioner)

```python
provisioner.client_config_flag(feature_name="smtp_dlp_enabled", action=False)  # disable, restore True on teardown
provisioner.client_config_flag(feature_name="inline_policy_enhancements_enabled", action=True/False)
# smtp_use_secondary_next_hop / enable_on_prem_msa: no-ops since R122/R123
```

**⚠ Possibly stale:** newer test classes (§23) parametrize explicitly on `enable_on_prem` (0/1)
crossed with the `nplan6151` (Tenant ID via custom headers) flag, which implies `enable_on_prem_msa`
is *not* purely a no-op in the current build. Re-verify against the live app before relying on the
"no-op since R122/R123" claim for any new test.

### 17.13 UI Input Data Files

```python
file_path = os.path.join(os.path.dirname(__file__), "../data/Feature_SMTP_Proxy/input.json")
smtp_msa = json.load(open(file_path, "r"))
# Keys: exchange_msa_tenant_1, exchange_msa_next_hop_1, email_body_traffic
```

### 17.14 Generate SMTP Traffic for UI Tests

```python
# session-scoped fixture: creates "Test1" custom MSA, sends 3 emails, waits 90s
builder = EmailBuilderSmtp(smtp_server=smtp_server_vip, smtp_port=25)
builder.send_email(from_email="user0@emailskope.com", to_email="fireeyesales2@gmail.com",
    subject=f"test subject line {unique_tag}", body=smtp_msa["email_body_traffic"],
    custom_headers={"x-custom-msa": "test"})
```

---

## 18. UI Layer — Alerts / Application Events / Incidents Page Patterns

Source: UI screenshots from `gurudattshenoy.qa.boomskope.com`

### 18.1 Page URLs and Navigation

| Page | URL | Nav path |
|---|---|---|
| Alerts | `/ns#/alerts` | Skope IT™ → Events & Alerts → Alerts |
| Application Events | `/ns#/skopeIT` | Skope IT™ → Events & Alerts → Application Events |
| DLP Incidents | `/ns#/incidents/dlp` | Incidents → DLP |

```python
from webui_libs.pages.skopeit.alerts import NSSkopeITAlertsPage
from webui_libs.pages.skopeit.application_events import NSSkopeITApplicationEventsPage
from webui_libs.pages.incidents.incidents_dlp import NSIncidentsDLPPage
alerts_page = NSSkopeITAlertsPage(selenium, base_url).open()
app_events_page = NSSkopeITApplicationEventsPage(selenium, base_url).open()
```

### 18.2 Alerts Table — Default Columns

`TIME | TIME (GMT) | NAME | ALERT TYPE | TRANSACTION ID | ACTION | POLICY NAME | SUBJECT LINE | CATEGORY | <feature columns>`
New SMTP feature columns added to the right. Column gear: `.ns-table-cell.column-settings > i`

### 18.3 Application Events Table — Default Columns

`TIME | ACTIVITY | ACCESS METHOD | USER | CATEGORY | OBJECT | SUBJECT LINE | WEBSITE | CCL | CCI | <feature columns> | FROM ISOLATION`
New SMTP feature columns inserted before `FROM ISOLATION`.

### 18.4 Feature-Flag-Controlled Column Pattern

- **Flag enabled** → column appears with correct values
- **Flag disabled** → column disappears entirely (no placeholder)
- **Previously-recorded events** → side panel values still shown after flag disabled (data persisted)

### 18.5 Side Panel Structure

Click zoom icon on row to open panel.

**Alert Details / Application Event Details sections (in order):**
```
User Key / Normalized
APPLICATION  → Application, App Category, Activity, Website, Object ID, Object Type,
               Subject Line, Message ID, SMTP Status, Tags, Other Category
DLP          → DLP Profile, DLP Rule, DLP Rule Count, DLP Severity, DLP File, DLP Incident ID
FILE         → File Type, Size, Language, SHA256, Encryption Status
SOURCE
SECURITY ASSESSMENT
DESTINATION  → IP, Location, GeoIP Source
SESSION      → # Total Events, Pop Name
```

**New SMTP feature fields land under "Other Category:" inside the APPLICATION section:**
```
Tags:
Other Category:
<FeatureField1>: <value>
<FeatureField2>: <value>    ← only shown when condition is met
```

### 18.6 Conditional Field Display Pattern

Secondary field shown **only when** primary meets a condition; **absent entirely** (not empty) otherwise.

```python
panel_text = get_side_panel_text(driver)
if "FeatureField1: Yes" in panel_text:
    assert "FeatureField2:" in panel_text
else:
    assert "FeatureField2:" not in panel_text
```

### 18.7 Filter Pattern

```python
from webui_libs.components.ns_multilevel_filter import Filter, FilterType
options = page.multilevel_filter.get_all_first_level_options_text_list()
assert "<Field Display Name>" in options

page.multilevel_filter.add_filter(Filter(
    first_level_value="<Field Display Name>",
    type_of_multifilter=FilterType.SearchText,
    second_level_value="Yes"))
page.wait_for_page_table_be_loaded()
assert page.page_pagination_table.get_row_count() > 1
```

### 18.8 Column Customize Dialog

```python
page.page_pagination_table.add_columns(column_names=["<Column Display Name>"])
columns = page.page_pagination_table.get_all_customize_columns()
assert "<Column Display Name>" in columns
```

### 18.9 Feature Flag Toggle for UI Column Visibility

```python
# Generic provisioner flag
provisioner.client_config_flag(feature_name="<flag_name>", action=True/False)

# SMTP-specific tenant-level flag
smtp_feature_api_client.update_feature_flags(
    headers={"x-netskope-tenantid": f"{tenant_id}"},
    feature_flag_payload={"<flag-name>": {"enabled": True}})
# Then reload page and assert column present/absent
```

### 18.10 Accessing Side Panel Fields by Label

```python
def get_side_panel_field_value(driver, field_label):
    panel = WebDriverWait(driver, 15).until(
        EC.presence_of_element_located((By.CSS_SELECTOR, ".side-panel, .alert-details, .event-details")))
    for line in panel.text.splitlines():
        if line.startswith(f"{field_label}:"):
            return line.split(":", 1)[1].strip()
    return None

assert get_side_panel_field_value(driver, "Machine Email Detected") == "Yes"
assert get_side_panel_field_value(driver, "Machine Email Confidence") is None  # field absent
```

---

## 23. Additional Page Objects & UI Test Areas (from live SMTP UI test folder)

Confirmed page objects beyond §17/§18, scoped strictly to `nsproxy/tests/ui/SMTP/` (the UI test
tree also contains dozens of unrelated-product folders — don't pull page objects from outside this
path for SMTP work):

```python
from webui_libs.pages.real_time_protection.real_time_policy_wrapper import RtProtectionPolicyWrapper
from webui_libs.pages.real_time_protection.real_time_protection_policy_page import (...)
from webui_libs.pages.SkopeIT_page_event import SkopeIT_Page_Event
from webui_libs.pages.email_notification_template import EmailNotificationPage
from webui_libs.pages.policies.profiles.constraint.users_profiles import (...)   # Constraint Profile page
from webui_libs.pages.incidents.incidents_dlp import NSIncidentsDLPPage          # already in §18
```

### 23.1 Custom Tenant Identification settings page (NPLAN-6151 WebUI)

Real test classes (`test_settings_smtp2.py`, `test_settings_smtp3.py`) parametrize this page along
three independent axes — combine all three when testing a change here, not just one:

- `enable_on_prem` (0/1) — On-Prem MSA enabled/disabled
- `nplan6151` (0/1) — Tenant Identification via custom headers flag
- Secondary next-hop configured or not

Class names to search for as a starting point: `TestSMTPProxySettingsPage_CustomTenantIdentification`,
`..._disable_onPremMSA`, `..._SecondaryHop`, `..._Disabled`.

### 23.2 DNS Domain Validation UI (two generations)

Two parallel test suites exist for domain validation UI — a "classic" flow and a newer "NGWEB"
flow (`test_domain_validation_classic_webui_cases.py`'s `TestSMTPProxy_DNSDomainValidation_classic`
vs. `test_ngweb_new_domain_validation.py`'s `TestNGWEBNewDomainValidation`). A domain-validation UI
change must be checked against both — don't assume the classic flow was removed just because a
newer one exists.

### 23.3 Self-Addressed Email on the Incidents page

`test_self_addressed_email_incidents_page.py` → `TestIncidentPageSelfAddressedEmail` — verifies
the self-addressed-email value/field specifically on the DLP Incidents page (`NSIncidentsDLPPage`),
distinct from the Alerts/Events SkopeIT filter coverage already in §18.7/§18.9.

### 23.4 Machine-Generated Email in SkopeIT

`test_skopeit_machine_generated_email.py` → `TestSkopeITMachineGeneratedEmail` — UI-side coverage
for the BOT-score feature (see `../references/feature_matrix.md` row 25 and
`smtp_proxy_kb_api.md` §21 for the backend/config side).

### 23.5 RTP Balkan combined-scenario suite

`rtp_balkan.py` (~1,300 lines, ~28 test functions) is a large combined-regression UI suite covering
RTP page access, MSA edit windows, Record Subject Line, DLP-flag interplay, Secondary Next Hop,
Custom MSA CRUD, and Tenant Identification (+ variants) together. Treat this as the UI equivalent
of `references/feature_matrix.md` row 34 (End-to-End / Solution-Level Scenarios) — a good place to
look for how a new feature might need a combined-scenario UI regression case, not just a unit one.

### 23.6 SMTP Settings page — old modal vs new modal

Confirmed in `test_settings_smtp.py`: `TestSMTPProxySettingsPageOldModal` and
`TestSMTPProxySettingsPageSecondaryNexthopNewModal` both exist and are both actively tested — the
settings page is mid-migration for at least secondary-next-hop configuration. A UI change here must
be verified against both modal versions; see `../references/product_architecture.md` "known
gotchas" for the same note. `TestCustomMSALimit` confirms the 3-custom-MSA limit is enforced at the
UI layer too, not just the backend DNS-validation layer (`smtp_proxy_kb_api.md` §18's domain
validation note).
