# O365_python_email_client

## Overview

`O365_python_email_client` is a Python command-line utility that sends email through **Microsoft Graph** on behalf of an Office 365 / Microsoft 365 mailbox.

Based on the code currently provided, this project:

- uses **MSAL** (`ConfidentialClientApplication`) to acquire an **app-only access token**
- authenticates with **certificate-based credentials** (private key + public certificate)
- reads Azure AD / Entra application metadata from an `Azure_AD.json` file
- sends mail by calling the Microsoft Graph endpoint:
  - `https://graph.microsoft.com/v1.0/users/{from_user}/SendMail`
- supports:
  - `To`
  - `Cc`
  - `Bcc`
  - `Reply-To`
  - subject
  - body text
  - body from file
  - basic file attachments
  - message importance

---

## Current Implementation Summary

The entry point is a Click-based CLI function:

- `send_email_via_O365()`

The authentication flow is:

1. Read `appclientID` and `tenantID` from `Azure_AD.json`
2. Load the private key PEM and public certificate PEM
3. Build certificate credentials for MSAL
4. Acquire a token for scope:
   - `https://graph.microsoft.com/.default`
5. Submit a `POST` request to Graph `users/{from_user}/SendMail`

---

## Features Implemented

### Authentication

- Microsoft Entra ID / Azure AD application configuration from JSON
- Certificate-based confidential client authentication
- Optional private key passphrase file
- Support for RSA certificate handling
- Support for ECDSA certificate handling

### Message Composition

- `From` mailbox selector via `--from_user`
- Multiple recipients via repeated `--to`
- Multiple CC recipients via repeated `--cc`
- Multiple BCC recipients via repeated `--bcc`
- Optional `--replyto`
- Subject line via `--subject_line` / `-s`
- Body via `--body` / `-b`
- Body loaded from file via `--body_file`
- Importance (`high`, `normal`, `low`)
- Optional attachments via repeated `--attachment_file` / `-a`

### Diagnostic Output

- Optional `--debug` mode prints:
  - the raw token response
  - a decoded JWT payload (without signature verification)
  - the generated Graph message payload

---

## Repository Layout

The repository appears to contain at least:

- `src/`
- `dist/`
- `pyproject.toml`
- `README.md`
- `LICENSE`

The provided code indicates the main behavior is a CLI email sender rather than an object-oriented SDK.

---

## Requirements

## Python

- Python 3.x

## Python packages used in the code

- `click`
- `msal`
- `requests`
- `PyJWT` (`jwt`)
- `asn1crypto`

Example install command:

```bash
pip install click msal requests pyjwt asn1crypto
```

If the project already defines dependencies in `pyproject.toml`, prefer installing from the project itself instead.

---

## Configuration Files

### 1. Azure_AD.json

The code expects a JSON file with the following schema:

```json
{
  "appclientID": "YOUR-APPLICATION-CLIENT-ID",
  "tenantID": "YOUR-TENANT-ID"
}
```

### 2. Certificate / Key Files

The code expects:

- `--keyfile`: PEM private key
- `--pubfile`: PEM public certificate
- `--passphrase_file`: optional file containing the private key passphrase

---

## Command-Line Usage

### Basic syntax

```bash
python your_script.py \
  --azure_ad_file Azure_AD.json \
  --keyfile private_key.pem \
  --pubfile public_cert.pem \
  --from_user sender@domain.com \
  --to recipient1@domain.com \
  --subject_line "Test message" \
  --body "Hello from Microsoft Graph"
```

### Example with multiple recipients and attachments

```bash
python your_script.py \
  --azure_ad_file Azure_AD.json \
  --keyfile private_key.pem \
  --pubfile public_cert.pem \
  --passphrase_file private_key.pass \
  --from_user sender@domain.com \
  --to user1@domain.com \
  --to user2@domain.com \
  --cc team@domain.com \
  --bcc audit@domain.com \
  --replyto noreply@domain.com \
  --subject_line "Quarterly status" \
  --body_file message.txt \
  --importance high \
  --attachment_file attachment1.txt \
  --attachment_file attachment2.txt
```

### HTML body example

```bash
python your_script.py \
  --azure_ad_file Azure_AD.json \
  --keyfile private_key.pem \
  --pubfile public_cert.pem \
  --from_user sender@domain.com \
  --to recipient@domain.com \
  --subject_line "HTML test" \
  --content_type HTML \
  --body "<html><body><h1>Hello</h1><p>Graph email test</p></body></html>"
```

---

## CLI Options

```text
--azure_ad_file         JSON file containing appclientID and tenantID
--passphrase_file       Private key passphrase file
--keyfile               PEM private key filename
--pubfile               PEM public certificate filename
--debug / --no-debug    Debug flag
--from_user             Sender mailbox / Graph user id
--to                    Recipient (repeatable)
--cc                    CC recipient (repeatable)
--bcc                   BCC recipient (repeatable)
--replyto               Reply-To address
--body_file             Message body filename
--body, -b              Message body text
--subject_line, -s      Subject line
--attachment_file, -a   Attachment filename (repeatable)
--content_type          Body content type (default: Text)
--importance, -i        high | normal | low (default: normal)
```

---

## How It Works

### Token acquisition

The script uses `msal.ConfidentialClientApplication` with certificate-backed client credentials:

- `client_id` comes from `Azure_AD.json`
- `tenant_id` comes from `Azure_AD.json`
- `client_credential` is built from:
  - private key
  - certificate thumbprint
  - public certificate
  - optional passphrase

Then it requests:

```python
app.acquire_token_for_client(scopes=["https://graph.microsoft.com/.default"])
```

### Message submission

The script constructs a Microsoft Graph message payload and sends it to:

```text
https://graph.microsoft.com/v1.0/users/{from_user}/SendMail
```

If the API call succeeds, it prints:

```text
Email sent successfully
```

Otherwise, it prints the JSON error response from Graph.

---

## Notes on Attachments

Attachments are currently added as Graph `fileAttachment` objects with:

- base64-encoded file content
- the original file name
- a hard-coded content type of `text/plain`

### Important limitation

The current implementation does **not** detect the MIME type of each attachment. As written, every attachment is sent with:

```text
contentType: text/plain
```

That may be acceptable for plain text files, but it is not ideal for PDFs, images, Office documents, or binary files.

---

## Security Notes

## What the code does well

- Uses Microsoft Graph rather than legacy basic-auth SMTP
- Uses application authentication via certificate credentials
- Supports encrypted private keys through an optional passphrase file

## Current security gaps / limitations in the provided source

- The code includes a TODO noting that endpoint / service validation should be improved:
  - `TODO Verify that we are communicating with Microsoft`
- No explicit HTTP timeout is set on the `requests.post()` call
- No retry / backoff behavior is implemented
- Debug mode prints token-related information and decoded JWT contents
- The JWT decode in debug mode explicitly disables signature verification for inspection output
- Secrets are file-based rather than integrated with a secrets manager

If you plan to use this in a production or regulated environment, consider strengthening those areas before deployment.

---

## Known Behavioral Characteristics

These observations are based directly on the provided code:

- This is a **CLI utility**, not a reusable typed SDK
- The script requires:
  - `--keyfile`
  - `--pubfile`
  - `--azure_ad_file`
- If those are missing, it prints the Click help and exits
- Message body may be supplied inline or from a file
- Newline and tab escape sequences in `--body` are normalized
- `saveToSentItems` is sent as the string `'true'`
- The helper function `base64UrlDecode()` is currently unused

---

## Example Azure AD / Entra App Expectations

To work correctly, the Microsoft Entra application typically needs:

- permission to send mail through Microsoft Graph
- an uploaded certificate that matches the private/public key pair used by this tool
- authorization to send as the specified mailbox / user context used in `--from_user`

Tenant-specific Exchange Online and Graph permission details depend on your environment and security model.

---

## Opportunities for Improvement

If you want to mature this project, high-value improvements would include:

1. Add request timeouts
2. Add retry / backoff logic for transient Graph failures
3. Add MIME type detection for attachments
4. Add structured logging instead of `print()`
5. Replace debug token printing with safer diagnostics
6. Validate Graph / Microsoft endpoints more explicitly
7. Add unit tests and integration tests
8. Add package entry points in `pyproject.toml`
9. Support large attachments if needed
10. Add a `--body_html_file` or clearer content-type validation
11. Improve error handling for file parsing, token failures, and HTTP exceptions
12. Document required Entra permissions and Exchange prerequisites

---

## Suggested Minimal Project Usage Pattern

A practical invocation pattern is:

1. Register an Entra application
2. Upload the certificate to the application
3. Create `Azure_AD.json`
4. Store the private key securely
5. Run the CLI with `--from_user`, `--to`, `--subject_line`, and `--body`
6. Use `--debug` only for troubleshooting in a controlled environment

---

## License

Refer to the repository `LICENSE` file.

---

## Disclaimer

This README was generated to match the provided Python source as closely as possible. It avoids assuming features that were not visible in the code snippet.

