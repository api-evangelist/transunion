---
name: Sign an outbound call with a SHAKEN PASSporT
description: >-
  Use TransUnion TruContact Trusted Call Solutions' Authentication Service (STI-AS) to mint
  a signed SIP Identity header for an outbound call, choosing the right media-type variant
  and handling the certificate and credential failures that dominate this API.
api: openapi/transunion-trucontact-tcs-shaken-openapi.yml
operations:
  - authn/identityj-j
  - authn/identitys-j
  - authn/identitys-s
  - authn/identitymps
  - authn/identitympj
generated: '2026-08-13'
method: generated
source: openapi/transunion-trucontact-tcs-shaken-openapi.yml
---

# Sign an outbound call with a SHAKEN PASSporT

The Authentication Service (STI-AS) takes an outbound INVITE and returns it with a signed
SIP `Identity` header attached. That header is the PASSporT the terminating carrier will
verify. Everything below is grounded in the published OpenAPI 3.1.0.

## Before you start

- **There is no TransUnion-hosted base URL.** `servers[]` is
  `https://{hostname}:{port}/{optionalRoutingPath}` because the STI-AS runs inside your own
  network under your TruContact agreement. Get the host from your deployment, never guess it.
- **Authentication is an `apiKey` QUERY parameter**, not a header, and it is *not* declared
  as a securityScheme — so your generated client will not add it for you. If you omit it,
  the service falls back to checking whether your source IP is on the pre-provisioned
  allowlist. Send the key over TLS only and keep it out of logs; a query-string key leaks
  into proxy logs and `Referer` headers.

## Step 1 — pick the operation by media type

The media types are encoded in the operation name as `<request>-<response>`. Pick once and
stay consistent; there is no content negotiation.

| Your input | You want back | Operation |
|---|---|---|
| JSON signing request | JSON | `authn/identityj-j` |
| Raw SIP INVITE as `text/plain` | JSON | `authn/identitys-j` |
| Raw SIP INVITE as `text/plain` | The INVITE, `text/plain` | `authn/identitys-s` |
| Multipart text/JSON | `text/plain` | `authn/identitymps` |
| Multipart text/JSON | JSON | `authn/identitympj` |

If you are sitting in the SIP path, `authn/identitys-s` is the natural fit — you hand it the
INVITE and get the same INVITE back with the `Identity` header inserted. If you are calling
from application code, use `authn/identityj-j`.

## Step 2 — build the signing request

`POST /authn/v2/identityj-j?apiKey=…`

Supply the attestation and the call identifiers. The query parameters the operation accepts
are `apiKey`, `ppt`, `attest`, `origid`, `compact`, `origcc`, `destcc`, `verstat` and
`identity`.

- `attest` — the SHAKEN attestation level, `A`, `B` or `C`. Assert `A` only when you both
  originated the call and can verify the calling number belongs to your subscriber.
- `origid` — a UUID you assign to this call. It is your only handle for traceback, so
  generate it per call and record it.
- `ppt` — the PASSporT type. Leave it at the SHAKEN default unless you are signing a
  diverted (`div`) or resource-priority (`rph`) call.
- `compact` — request the compact PASSporT form when header size matters on the wire.

## Step 3 — read the response

A `200` returns the `Identity` header (schema `Identity`, wrapped by `SuccessResponse`) — or,
on the `-s` variants, the whole INVITE with `Identity` inserted and `verstat` set on the
`From` URI. Insert it verbatim; do not re-encode the JWT.

## Step 4 — handle failure properly

**The declared responses are only `200`, `4xx` and `5xx`.** The real statuses live in the
spec's examples, and `errors/transunion-problem-types.yml` in this repo lists all 179
`error_id` values with the HTTP status and the SIP response code each maps to. Branch on
`error_id`, never on the HTTP status alone.

Every error carries `{error_id, http_status_code, sip_code, timestamp, reason}`. **`sip_code`
is the field that matters operationally** — it is what you propagate back into call
signalling (436 Bad Identity Info, 437 Unsupported Credential, 438 Invalid Identity Header).

The failures worth pre-handling, because certificate and credential classes make up more
than a third of the catalog:

- `ApiKeyRequired` / `ApiKeyInvalid` (400) — your key is missing or wrong, or you fell back
  to IP auth from an address that is not allowlisted.
- `CredentialNotFound`, `CredentialExpired`, `CredentialNotYetValid`, `CredentialKeyStoreError`
  (400/403) — a signing-credential problem on your side. These are operational alarms, not
  per-call retries: page someone.
- `CertificateExpired`, `CertificateNotFound`, `CertificateNotValidForOrig`,
  `CertificateTnAuthnListFetchError` (403) — the STI certificate or its TNAuthList does not
  cover the number you are trying to sign for.
- `AtisSigningRequestInvalid` / `AtisSigningRequestMissing` (400) — malformed or incomplete
  request. Never retry these unchanged.

## Retry rules

**There is no idempotency contract.** No `Idempotency-Key` parameter exists on any operation,
and no retry semantics are documented. Treat `5xx` and `503` as safe to retry with backoff —
signing the same call again produces another valid PASSporT rather than a duplicate side
effect — and treat every `4xx` as terminal until you change the request. There are also no
published rate limits and no `429`, so throttle yourself to your carrier capacity agreement;
the API will not tell you your budget.
