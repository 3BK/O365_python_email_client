# SECURITY.md

# Security Policy

## Overview

`O365_python_email_client` is a Python command-line utility that sends email through Microsoft Graph using Microsoft Entra ID / Azure AD application authentication and certificate-based credentials.

Because this utility handles authentication material, mailbox targeting, and outbound email transmission, it should be treated as a security-relevant component. This document describes the current security posture, supported reporting process, operational expectations, and key hardening recommendations.

---

## Supported Versions

At present, no formal version support matrix has been published for this project.

Recommended current support posture:

- support the latest maintained branch only
- treat older copies and local forks as unsupported unless explicitly reviewed
- revalidate security posture after any authentication, HTTP, packaging, or dependency changes

If this project is deployed in production, maintainers should publish an explicit support table such as:

```text
Version    Supported
main       Yes
<older>    No
```

---

## Security Contact / Reporting a Vulnerability

If you discover a security issue in this project, do **not** disclose it publicly until the issue has been reviewed and remediated.

Recommended reporting process:

1. Share a private report with the maintainer / repository owner.
2. Include:
   - affected version / commit
   - reproduction steps
   - proof of impact
   - logs or stack traces with secrets removed
   - whether the issue enables credential disclosure, unauthorized mail send, privilege escalation, or denial of service
3. Allow reasonable time for validation and remediation before public disclosure.

If this repository is used within an enterprise or regulated environment, follow the organization’s coordinated vulnerability disclosure and incident reporting process.

---

## Security Scope

Security-relevant assets in this project include:

- Microsoft Entra / Azure AD application identifiers
- certificate-backed client credentials
- private key material
- private key passphrase files
- mailbox identity passed via `--from_user`
- email recipient data
- message content and attachments
- access tokens returned by MSAL
- debug output that may expose token metadata

Compromise of any of the above may enable unauthorized email transmission, exposure of sensitive data, or abuse of mailbox-related permissions.

---

## Current Security Characteristics

The current implementation, based on the reviewed Python source, has the following positive security properties:

- uses Microsoft Graph rather than legacy basic-auth SMTP
- uses application authentication via MSAL confidential client flow
- supports certificate-based authentication instead of shared client secrets alone
- supports encrypted private keys through an optional passphrase file
- sends mail over HTTPS to Microsoft Graph

These are meaningful strengths, but they do **not** by themselves make the tool production-hard.

---

## Known Security Risks and Limitations

The following risks should be assumed until explicitly remediated.

### 1. Endpoint / service trust validation is incomplete

The source contains an explicit TODO indicating the author intended to improve validation of whether the client is communicating with Microsoft.

**Risk:** trust assumptions are implicit rather than explicitly documented and enforced at the application level.

### 2. No explicit HTTP timeout

The outbound `requests.post()` call does not set a timeout.

**Risk:** the process may hang indefinitely under adverse network conditions, which can create operational and availability risk.

### 3. No retry / backoff strategy

Transient Microsoft Graph, DNS, TLS, proxy, or network failures are not retried.

**Risk:** intermittent failures may cause unreliable delivery and push operators toward unsafe ad-hoc reruns.

### 4. Debug mode may expose sensitive information

Debug mode prints:

- raw token acquisition output
- decoded JWT contents
- generated message payload

**Risk:** logs, consoles, CI/CD output, or shell history may expose sensitive operational metadata.

### 5. JWT inspection disables signature verification

The debug path decodes the access token with signature verification disabled for inspection.

**Risk:** acceptable for local diagnostics if carefully controlled, but this output must not be treated as a trust decision.

### 6. Attachment content type is hard-coded

Attachments are emitted with `contentType: text/plain` regardless of actual file type.

**Risk:** this may produce incorrect content handling, downstream filtering issues, or inconsistent recipient experience.

### 7. No secret manager integration

The implementation expects local files for:

- Azure application metadata
- private key
- public certificate
- optional passphrase

**Risk:** local secret sprawl, poor lifecycle control, accidental inclusion in backups, and weaker auditability.

### 8. No explicit permission guardrails in code

The tool assumes the Entra application and Exchange / Graph permissions are appropriately constrained.

**Risk:** if the application is over-permissioned, misuse of the tool may enable broader-than-intended send capability.

### 9. Limited input validation

The script performs only basic operational checks such as whether files exist.

**Risk:** malformed inputs may lead to partial failures, confusing operator behavior, or weak diagnostics.

### 10. No defensive logging model

The current implementation uses `print()` statements instead of structured, redacted logging.

**Risk:** sensitive data may be written to logs, terminal captures, or centralized collectors without sanitization.

---

## Threat Considerations

This project should be evaluated against at least the following threat scenarios:

- theft of private key or passphrase file
- unauthorized local execution by a user with access to credential files
- over-permissioned Entra application capable of sending as unintended mailboxes
- accidental leakage of access token metadata through debug output
- misuse of the tool in automation pipelines without change control
- denial of service caused by network hangs or lack of timeout controls
- sensitive data exposure through attachment mishandling or broad recipient entry

---

## Recommended Hardening Controls

The following improvements are strongly recommended before production deployment.

### Authentication and Secrets

- store private keys in a managed secret or key system where feasible
- avoid keeping long-lived credential material in broadly readable filesystem paths
- protect passphrase files with least-privilege file permissions
- rotate certificates on a defined schedule
- document the certificate lifecycle, expiry monitoring, and replacement procedure
- restrict Entra application permissions to the minimum required scope

### Transport and Network Behavior

- set explicit HTTP connect and read timeouts
- implement controlled retry with backoff for transient failures
- record request correlation information safely for troubleshooting
- document expected proxy, egress, and TLS behavior for enterprise environments

### Logging and Diagnostics

- replace `print()` with structured logging
- redact tokens, credentials, and mailbox-sensitive fields from logs
- disable or tightly control debug mode in shared environments
- treat debug output as sensitive

### Input and Payload Validation

- validate email addresses and required arguments more strictly
- detect attachment MIME types rather than forcing `text/plain`
- validate message size and attachment size constraints
- add safer error handling around file reads and network failures

### Software Assurance

- pin and review dependencies
- add unit tests for token acquisition path and payload generation
- add integration tests for Graph send behavior in a test tenant
- add static analysis and linting in CI
- publish a release and support policy

---

## Operational Security Guidance

### Minimum Safe Usage Expectations

- run the tool only on trusted hosts
- restrict filesystem access to key material
- avoid using `--debug` in production
- protect shell history where command-line parameters may reveal operational details
- use a dedicated application registration for this tool
- use a dedicated sending identity where feasible
- monitor Graph / Exchange send activity for anomalies

### Recommended File Permission Model

Credential-related files should be readable only by the account executing the tool.

Examples of sensitive files:

- `Azure_AD.json`
- private key PEM
- passphrase file

Do not commit any of these files to source control.

---

## Secure Development Expectations

Contributors should follow these principles:

- never commit secrets, private keys, tokens, or passphrase files
- do not add debug output that prints credentials or bearer tokens
- prefer explicit exception handling over silent failure
- document all new permissions, endpoints, and dependencies
- re-review the security posture whenever authentication logic changes

---

## Deployment Suitability

### Suitable use cases

This project may be appropriate for:

- controlled administrative automation
- low-volume integration tasks
- lab or proof-of-concept Graph-based send workflows
- internal tools where certificate handling is already governed

### Use with caution

Additional hardening is recommended before use in:

- unattended production automation
- shared jump hosts
- CI/CD agents with broad access
- regulated environments requiring audit-grade secret management and logging controls

---

## Immediate Improvement Priorities

If only a few issues can be addressed first, prioritize these:

1. add request timeouts
2. remove or heavily restrict sensitive debug output
3. introduce structured redacted logging
4. integrate secret handling with a managed store where feasible
5. validate and document the minimum required Graph / Exchange permissions
6. add MIME type detection for attachments
7. add retry and exception handling around outbound HTTP calls

---

## Non-Goals / Current Limitations

This project does not currently claim to provide:

- hardware-backed key protection
- formal secure secret storage
- formal FIPS validation
- tamper-resistant logging
- delivery guarantees
- workflow approval controls
- built-in policy enforcement for allowed senders or recipients

If those properties are required, they must be implemented outside of or in addition to this tool.

---

## Disclosure and Change Management

Security-impacting changes should include:

- a description of the issue addressed
- affected threat scenario(s)
- operational impact
- config or permission changes required
- rollback considerations

When possible, maintainers should track:

- dependency updates
- certificate handling changes
- Graph permission changes
- logging behavior changes
- CLI changes affecting secret exposure

---

## Acknowledgements

Responsible disclosure is appreciated. Reports that are clear, reproducible, and scoped to actual impact are most helpful.
