---
name: Verify an inbound call and apply Call Validation Treatment
description: >-
  Use TransUnion TruContact Trusted Call Solutions' Verification Service (STI-VS) to validate
  the SHAKEN PASSporT on an inbound call, and use the CVT operation to get calling name and
  robocall analytics alongside the verdict.
api: openapi/transunion-trucontact-tcs-shaken-openapi.yml
operations:
  - verify/identityj-j
  - verify/identitys-j
  - verify/identitys-s
  - verify/identitycvt
generated: '2026-08-13'
method: generated
source: openapi/transunion-trucontact-tcs-shaken-openapi.yml
---

# Verify an inbound call and apply Call Validation Treatment

The Verification Service (STI-VS) validates one or more `Identity` headers on an inbound
INVITE and tells you whether the calling number was legitimately attested. The CVT variant
adds the commercially distinctive part: calling name and robocall analytics in the same call.

## Before you start

Same two facts as the signing skill: the host is your own deployment (`servers[]` is
templated), and auth is an `apiKey` **query** parameter with client-IP allowlist as the
fallback. Neither is declared as a securityScheme, so add the key yourself.

## Step 1 — pick the operation

| Your input | You want back | Operation |
|---|---|---|
| JSON verification request | JSON | `verify/identityj-j` |
| Raw SIP INVITE as `text/plain` | JSON | `verify/identitys-j` |
| Raw SIP INVITE as `text/plain` | The INVITE, `text/plain` | `verify/identitys-s` |
| JSON, **with CNAM + robocall analytics** | JSON | `verify/identitycvt` |

Use `verify/identitycvt` when you intend to do anything with the result beyond pass/fail —
display a name, block, or route to voicemail. The other three give you the verdict only.

## Step 2 — send the verification request

`POST /verify/v2/identityCVT?apiKey=…`

Accepted query parameters: `apiKey`, `status`, `origcc`, `destcc`, `cnam`, `robocall`,
`cnamPrefix`, `verstat`, `identity`.

- `identity` — the `Identity` header(s) from the inbound INVITE. Schema `VerifyIdentity`
  accepts **more than one**; a call can carry several PASSporTs (shaken plus div, for
  example) and each is verified independently.
- `cnam` / `cnamPrefix` — request calling-name lookup and control the prefix rendered on the
  display name. In the spec's own example the verified response comes back with a check
  mark prefixed to the display name.
- `robocall` — request robocall analytics treatment.
- `verstat` — control whether the service writes `verstat` onto the `From` URI of the
  returned INVITE (`TN-Validation-Passed` / `TN-Validation-Failed`).

## Step 3 — read the verdict

`200` returns `VerifyResponse`. On the `-s` variant you get the INVITE back with `verstat`
set on the `From` URI, which is what most SBCs want. **A 200 does not mean the call passed** —
it means verification ran. Read the status field, not the HTTP code, before deciding call
treatment.

## Step 4 — handle failure, and note the different envelope

The Verification Service does **not** return the same error shape as the signing service.
`VerifyErrorResponse` wraps an **array**:

```
{ "status": "error", "error": [ {error_id, http_status_code, sip_code, timestamp, reason}, … ] }
```

That is deliberate — a call carrying several PASSporTs can fail on several of them at once,
and you must iterate. Code that assumes a single error object will silently drop failures.

Verification-side classes to expect (full list in `errors/transunion-problem-types.yml`):

- `AtisVerificationRequestInvalid` / `AtisVerificationRequestMissing` (400) — malformed request.
- `CertificateFetchError`, `CertificateConversionError` (400, `sip_code` 436) — the `x5u` in
  the PASSporT could not be fetched or parsed. This is the *caller's* certificate, not yours;
  it is a verification failure, not an outage on your side.
- `CertificateExpired`, `CertificateNotFound`, `CertificateNotSupported` (403, `sip_code`
  437/438) — the signing credential is not acceptable.
- `CnamDataNotFound`, `CnamNotAuthorized` — CVT-specific. `CnamNotAuthorized` means your
  agreement does not include calling-name; it will not resolve by retrying.
- `ApiKeyRequired` / `ApiKeyInvalid` (400).

Always propagate `sip_code` into your SIP response rather than inventing one.

## Retry rules

No idempotency contract, no published rate limits, no `429`. Verification is a read-shaped
operation, so retrying a `5xx`/`503` with backoff is safe. Never retry a `4xx` unchanged, and
never let a verification failure become a retry storm against the caller's certificate host.
