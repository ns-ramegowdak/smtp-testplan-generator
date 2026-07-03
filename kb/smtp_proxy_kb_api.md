# SMTP Proxy KB — API / Backend Layer

Sections: Infrastructure, Feature Flags, Relay Config, RT Policy, smtplib, Logs, Setup/Teardown, Test Files, Response Codes, Placeholders
For UI patterns → see smtp_proxy_kb_ui.md

---

## 1. Infrastructure Overview

### Kubernetes

| Item | Value |
|---|---|
| Namespace | `smtpproxyv2` |
| Container name | `smtpproxy` |
| Process name (for restart) | `nssmtpproxy` |
| Restart command | `Kube.restart_smtp_container(k8s_operations)` → `pkill -f nssmtpproxy` + `time.sleep(100)` |

### Config Files (on pod)

| File | Purpose |
|---|---|
| `/opt/ns/tenant/{tenant_id}/feature_config_smtp.json` | Per-tenant feature flags |
| `/opt/ns/tenant/{tenant_id}/smtpSettings.json` | Per-tenant email relay/domain config |
| `/opt/ns/cfg/staged_config_smtp.json` | Global staged config |

### Shell Commands on Pod

```python
Kube.exec_smtpproxy_shell_command(k8s_operations, command="cat /opt/ns/tenant/{tenant}/feature_config_smtp.json")
k8s_operations.exec_pod_shell_commands(pod_name, container="smtpproxy", command)
```

---

## 2. SMTP Proxy VIP

```python
smtp_server_vip = input_json["smtp_server_vip"].get(stork_name, "163.116.158.109")
```
Placeholder: `<SMTP_PROXY_VIP>`. Default port: `25`.

---

## 3. Feature Flags

### 3.1 Tenant-Scoped Feature Flags

**API:** `SMTPFeatureEnablementAPI` (from `routine.py`)
**Endpoint:** `POST http://{smtp_feature_enablement_host}/internal/v1/featureconfigs`
**Header:** `x-netskope-tenantid: {tenant_id}`

```python
headers = {"x-netskope-tenantid": f"{tenant_id}"}
resp = smtp_feature_api_client.update_feature_flags(headers=headers, feature_flag_payload={...})
assert resp.status_code in [200, 201]

# GET current flags
resp = smtp_feature_api_client.get_feature_flags(headers=headers, params={})

# Sync to pod
resp = smtp_feature_api_client.sync_to_smtp_proxy(headers=headers)
```

**Helper functions:**
```python
enable_disable_front_conn(tenant_id, smtp_feature_api_client, flag=True/False)
# Payload: {"front-conn-persistence": {"enabled": bool}}

enable_disable_back_conn(tenant_id, smtp_feature_api_client, flag=True/False)
# Payload: {"back-conn-persistence": {"enabled": bool}}

enable_disable_lfs(tenant_id, smtp_feature_api_client, flag=True/False, max_file_size=64)
# Payload: {"dlp-settings": {"enabled": bool, "max-file-size": int_MB, "timeout": 90}}

enable_disable_source_ip_identification(tenant_id, smtp_feature_api_client, flag)
# Payload: {"srcip-identification": {"enabled": bool, "headers": [{"key": str, "value": str}]}}

enable_disable_multi_hop(tenant_id, smtp_feature_api_client, flag=True/False)
# Payload: {"use-multiple-next-hops": bool}
# Self-addressed: {"self-addressed-email-detection": {"enabled": bool, "threshold-similarity-score": 80}}
```

**Sync validation (poll loop):**
```python
for _ in range(10):
    json_data = get_featureConfigjson_kube(tenant_id, k8s_operations)
    if json_data["core"]["front-conn-persistence"]["enabled"] == expected:
        break
    time.sleep(1)
else:
    raise Exception("Feature flag sync timed out")
```

**Full idempotent helper:** `update_smtp_feature_flag_status(tenant, k8s_operations, smtp_feature_api_client, feature_flag, status=True)`

### 3.2 Feature Flag Names (feature_config_smtp.json → "core" key)

| Flag Name | Type | Description |
|---|---|---|
| `front-conn-persistence` | `{"enabled": bool}` | Reuse client-facing TCP connections |
| `back-conn-persistence` | `{"enabled": bool}` | Reuse server-facing TCP connections |
| `dlp-settings` | `{"enabled": bool, "max-file-size": int, "timeout": int}` | Large file support |
| `srcip-identification` | `{"enabled": bool, "headers": [...]}` | Source IP tenant ID |
| `use-multiple-next-hops` | `bool` | Multi-hop relay failover |
| `self-addressed-email-detection` | `{"enabled": bool, "threshold-similarity-score": int}` | Self-addressed detection |

### 3.3 Staged Config Flags

**File:** `/opt/ns/cfg/staged_config_smtp.json`
```python
update_smtp_staged_config(k8s_operations, feature_flag_name, payload=None, status=True)
```

Known entries:
```python
{"name": "back-persistence-settings", "data": {"enabled": True, "idle-timer-enabled": True,
 "idle-timer-interval-secs": 60, "send-noop-enabled": True, "send-noop-interval-secs": 10,
 "max-messages-per-connection": 30, "close-front-if-back-closed": True}, "apply-globally": True}

{"name": "smtpproxy-header-settings", "data": {"preserve-header-whitespace": True/False}, "apply-globally": True}
```

### 3.4 DKIM Feature Flag (nswatson.sh)

```bash
nswatson.sh -p 3214 -c 'set feature verify-dkim true'
nswatson.sh -p 3214 -c 'set feature verify-dkim false'
nswatson.sh -p 3214 -c 'set log level all 0 smtpheader 2 smtpreq 2 smtpres 1 dnslkup 3'
nswatson.sh -p 3214 -c 'set log level all 0'
```
```python
Kube.exec_smtpproxy_shell_command(k8s_operations, command="nswatson.sh -p 3214 -c '...'")
```

---

## 4. Email Relay Config (SMTP Settings / Domain Config)

**API:** `webapi.settings.security_cloud_platform.mail_relay.smtp.Smtp`

```python
smtpsetting_obj = Smtp(webui)
response = smtpsetting_obj.get_email_relay_settings()         # {"data": [...]}
response = smtpsetting_obj.create_email_relay_from_hash(domain_hash=domain_cfg)
assert response.get("code") == 200
response = smtpsetting_obj.delete_email_relay_custom_msa(id)
assert response == 200
```

**Domain config structure:**
```python
domain_cfg = {
    "id": "3", "is_custom": True, "name": "<sender_domain>",
    "data": {
        "app": "<sender_domain>",
        "TenantIdentification": {"customVerifiedDomain": [{
            "domain": "<sender_domain>", "domainVerified": "1",
            "DestinationIP": [{"host": "<MAILSERVER_HOST>", "port": 25}],
            "tenantKey": "", "tenantValue": "",
        }]},
        "individuallySet": "1", "individuallySetKeyValue": "1", "sourceIp": [],
    }
}
```

**ID rules:** 0/1/2 reserved (built-ins). Custom: ID 3+. Max 3 custom MSAs per tenant.

**Backend verification:**
```python
verify_domain_hash_in_file(k8s_operations, tenant_id, expected_domain_hash)
# Reads /opt/ns/tenant/{tenant_id}/smtpSettings.json
```

---

## 5. Real-Time Policy

```python
policy_obj = policy["RTPolicy"]
policy_obj.create_policy(name=policy_name, policy_hash=email_policy_cfg)
policy_obj.delete_policy(policy_name)
# email_policy_cfg from: common.input_json_data_parse() → input_json["dtelu-email-dlp-test"]
```

---

## 6. Sending Emails (smtplib)

### 6.1 Standard Send
```python
import smtplib, ssl
context = ssl.create_default_context()
context.check_hostname = False
context.verify_mode = ssl.CERT_NONE
server = smtplib.SMTP(smtp_server_vip, port=25)
server.starttls(context=context)
server.sendmail(_from, _to_list, msg.as_string())
server.quit()
```

### 6.2 EHLO / HELO
```python
code, message = server.ehlo(sender_domain)   # expect 250
code, message = server.helo(sender_domain)   # expect 250
```

### 6.3 Raw SMTP Commands
```python
server.sock.sendall(b"MAIL FROM:<user@domain.com>\r\n"); code, msg = server.getreply()
server.sock.sendall(b"RCPT TO:<recipient@domain.com>\r\n"); code, msg = server.getreply()
server.sock.sendall(b"DATA\r\n"); code, msg = server.getreply()          # expect 354
server.sock.sendall(b"STARTTLS\r\n"); code, msg = server.getreply()
server.sock.sendall(b"VRFY user@domain.com\r\n"); code, msg = server.getreply()  # expect 502
server.sock.sendall(b"QUIT\r\n"); code, msg = server.getreply()          # expect 221
```

### 6.4 RSET / NOOP
```python
code, message = server.rset()   # expect 250
code, message = server.noop()   # expect 250
```

### 6.5 SMTP Exceptions
```python
smtplib.SMTPDataError           # DATA phase rejection (e.g. 552 LFS)
smtplib.SMTPServerDisconnected  # server closed connection
smtplib.SMTPSenderRefused       # MAIL FROM rejected
smtplib.SMTPRecipientsRefused   # all RCPT TO rejected
smtplib.SMTPResponseException   # generic — e.smtp_code, e.smtp_error
```

### 6.6 EmailBuilder
```python
from nsproxy.tests.api.SMTP.smtp_proxy.Email_builder_smtplib import EmailBuilderSmtp
builder = EmailBuilderSmtp(smtp_server=smtp_server_vip, smtp_port=25)
code, msg, recipients = builder.send_email(
    from_email="user@domain.com", to_email="recipient@domain.com",
    subject="Test Subject", body="body", attachment="dlp_pci.txt",
    custom_headers={"x-netskope-pop": "qa01_test"}, bcc_email="bcc@domain.com",
)
```

### 6.7 DKIM-Signed Email
```python
# CLI
python send_dkim_email.py --host <SMTP_PROXY_VIP> --port 25 --sender user@<DKIM_DOMAIN> \
  --recipient recipient@<DKIM_DOMAIN> --dkim-private-key <DKIM_PRIVATE_KEY> \
  --dkim-selector <DKIM_SELECTOR> --dkim-domain <DKIM_DOMAIN> --dkim-mode simple/simple

# Python API
sender = DKIMEmailSender(script_path="send_dkim_email.py", stork_name=stork_name)
sender.send_email(host=smtp_server_vip, port=25, from_addr=f"user@{dkim_domain}",
    to_addr=f"recipient@{dkim_domain}", dkim_private_key=dkim_keys.private,
    dkim_selector=dkim_selector, dkim_domain=dkim_domain, dkim_mode="simple/simple",
    body_file=None, skip_space=False)
```

---

## 7. Log Verification

### 7.1 Fetching Logs
```python
pod_logs = Kube.get_smtpproxy_pods_log(k8s_operations, since_seconds=60)  # {pod_name: log_string}
pod_logs_tailer = Kube.tail_smtpproxy_pods_log(k8s_operations, since_seconds=60)
```

### 7.2 SMTPREQ Pattern
```
SMTPREQ N: SmtpRequestEngine.cpp:NNNNN
  trid={trid} rqid={rqid} tenantid={tenant_id} user='{sender_email}'
  src-ip=[{ip}:{port}] front-conn-reuse=[true/false]
  messageId=[<{message_id}>]
  [MAIL FROM={sender_email}] [RCPT TO={recipient_email}] rcpt-count=[{N}]
```

### 7.3 SMTPRES Pattern
```
SMTPRES N: SmtpResponseEngine.cpp:NNNNN
  trid={trid} rqid={rqid} tenantid={tenant_id} user='{sender_email}'
  Done sending content to {relay_host}:25; got reply code={smtp_code};
  [MAIL FROM={sender_email}]; messageId=[<{message_id}>];
  rcpt-count=[{N}]; send {code} OK to front
```

### 7.4 DKIM Log Patterns
```
Ingress DKIM Check: Pass    domain=<dkim_domain> selector=<selector> algo=rsa-sha256 canonicalization=<mode>
Ingress DKIM Check: Fail    detail=body hash mismatch
Ingress DKIM Check: TempError  selector=<selector> detail=NXDOMAIN
Ingress DKIM Check: None
Egress DKIM Check: Pass / Fail  detail=<reason>
DKIM degradation detected: signature was valid at ingress but failed at egress
```

### 7.5 Log Parser / Verifier
```python
log_parser(log_line, rqid=rqid, signature="SMTPREQ")
verifier = SMTPLogVerifier(k8s_operations)
success, message = verifier.verify_recipient_removal(
    expected_removed=["user@domain.com"], expected_kept=["other@domain.com"],
    message_id=None, wait_time=10, since_seconds=60)
```

### 7.6 Error Code 558
`558 User <email> is restricted and has been deleted from the recipient list due to policy`

---

## 8. Test Setup Pattern (Standard)

```
1. Read domain config template from smtpSettings.json
2. Generate random sender_domain (lorem.words)
3. Toggle feature flags via smtp_feature_api_client.update_feature_flags()
4. Poll get_featureConfigjson_kube() until synced (10 retries, 1s sleep)
5. Create email relay config via smtpsetting_obj.create_email_relay_from_hash()
6. Wait 60s for relay config propagation
7. Create RT policy via policy_obj.create_policy()
8. time.sleep(5) for policy propagation
9. Connect to SMTP proxy VIP and send email(s)
10. Verify (log-based or delivery-based)
```

## 9. Test Teardown Pattern (always in `finally` block)

```python
finally:
    if is_policy_created:
        policy_obj.delete_policy(policy_name)
    if is_email_relay_created:
        smtpsetting_obj.delete_email_relay_custom_msa(domain_cfg["id"])
    enable_disable_front_conn(tenant_id, smtp_feature_api_client, False)
    enable_disable_back_conn(tenant_id, smtp_feature_api_client, False)
```

---

## 10. Test File Organization

| Test File | Feature Area |
|---|---|
| `test_smtp_connection_reuse.py` | Front/back connection persistence |
| `test_smtp_dkim_functionality.py` | DKIM signing/verification |
| `test_large_file_support_smtp.py` | LFS: 552 rejection |
| `test_regression_smtp_proxy.py` | Core regression |
| `test_remove_recipients.py` | Remove recipients, 558 error |
| `test_self_addressed_email_detection.py` | Self-addressed detection |
| `test_tenant_identification_source_ip.py` | Source IP tenant ID |
| `test_tenant_identification_custom_header.py` | Custom header tenant ID |
| `test_secondary_next_hop.py` | Secondary relay routing |
| `test_smtp_front_back_tls_support.py` | TLS support |
| `test_smtp_proxy_resilency.py` | Pod restart, resilience |
| `test_pdv_smtp_proxy.py` | PDV (Postfix delivery) |
| `test_feature_enablement_api.py` | Feature flag API |
| `test_recording_subject_search.py` | Subject recording |

---

## 11. Test Markers

```python
@pytest.mark.feature_smtp       # all SMTP proxy tests
@pytest.mark.docker             # requires Postfix Docker container
@pytest.mark.stack_agnostic     # no proxy restart needed
@pytest.mark.nondestructive     # no permanent tenant state change
@flaky(max_runs=2)
@pyrobin.case("C2400744")       # TestRail case ID
```

---

## 12. Test Data Files

`nsproxy/tests/api/data/SMTP/smtp_proxy/`: `smtpSettings.json`, `input_json.json`, `input_email_args.json`
`nsproxy/tests/api/data_file/`: `dlp_pci.txt`, `PCI_4MB.pptx`, `test_lfs_payload.dat`, DKIM key pair

---

## 13. Downstream Mail Server / Delivery Verification

```python
# Gmail IMAP
with gmail_mailbox(common) as mailbox:
    msg = get_email_notification(mailbox, from_email, subject_pattern, timeout=100)
    assert msg is not None

# Postfix Docker
ssh <MAILSERVER_USER>@<MAILSERVER_HOST> "ls Maildir/new/"
```
Wait times: standard `time.sleep(15)`, DLP content `time.sleep(15–60)`.

---

## 14. File Generator

```python
from nsproxy.tests.api.common.file_generator.file_generator import generate_file
file_path = generate_file(32, 'MB', file_type="txt", file_complaint='dlp-pci')
file_path = generate_file(1, 'MB', file_type="txt")
```

---

## 15. Known SMTP Response Codes

| Code | Meaning |
|---|---|
| 220 | Service ready |
| 221 | Goodbye (QUIT) |
| 250 | OK |
| 354 | Start mail input (DATA) |
| 452 | Too many recipients |
| 501 | Syntax error in parameters |
| 502 | Command not implemented (VRFY) |
| 503 | Bad sequence of commands |
| 521 | Host does not accept mail |
| 552 | Exceeded storage allocation (LFS) |
| 555 | MAIL FROM/RCPT TO rejected |
| 558 | No recipients remaining (policy) |
| 5xx | Permanent failure (no relay config) |

---

## 16. Common Placeholder Variables

| Placeholder | Value / Source |
|---|---|
| `<SMTP_PROXY_VIP>` | `input_json["smtp_server_vip"][stork_name]` |
| `<SMTP_PROXY_PORT>` | `25` |
| `<SMTP_PROXY_NS>` | `smtpproxyv2` |
| `<SMTP_PROXY_POD>` | `Kube.get_smtp_pods(k8s_operations)` |
| `<TENANT_ID>` | `nsconfig[-1]` |
| `<SENDER_DOMAIN>` | randomly generated |
| `<SENDER_EMAIL>` | `userN@<SENDER_DOMAIN>` |
| `<RECIPIENT_EMAIL>` | `userN@swg.geskope.com` |
| `<MAILSERVER_HOST>` | `domain_cfg["DestinationIP"][0]["host"]` |
| `<RELAY_CONFIG_ID>` | Integer 3+ |
| `<RT_POLICY_NAME>` | timestamped |
| `<DKIM_DOMAIN>` | `dkim-postfix.swg.geskope.com` |
| `<DKIM_SELECTOR>` | `default` |
| `<DKIM_PRIVATE_KEY>` | path in test data directory |
| `<FEATURE_FLAG>` | one of the names in §3.2 |
