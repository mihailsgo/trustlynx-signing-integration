# TrustLynx SignBox Integration API Guide

## Overview
This document defines a generic backend integration flow for any client system that needs SignBox web signing.

Integration outcome:

1. Upload a source document to SignBox Archive.
2. Generate a SignBox web signing session URL for that document.
3. Redirect user to SignBox signing page.
4. Receive user back in client application.
5. Download signed document from Archive.

This guide intentionally includes only APIs required for this flow.

## System Roles
- Client Backend: trusted server-side component in client landscape.
- Client Frontend: browser/UI used by end users.
- SignBox Internal Keycloak: OIDC token issuer for internal APIs.
- SignBox Archive Service: document storage and retrieval.
- SignBox Internal Gateway: API that creates signing session redirect URL.

## End-to-End Flow
```text
Client Backend
  -> POST /api/document/create (Archive)
  <- { id }
  -> POST /protocol/openid-connect/token (Internal Keycloak)
  <- { access_token }
  -> POST /api/auth/session/redirecturl?id={id} (Internal Gateway)
     Authorization: Bearer <access_token>
  <- <signingRedirectUrl>

Client Frontend
  -> browser redirect to <signingRedirectUrl>
User signs document in SignBox UI
SignBox redirects user back to client return URL

Client Backend
  -> GET /api/document/{id}/download (Archive)
  <- signed file bytes
```

## Critical Rule
`/api/auth/session/redirecturl` is a protected backend API.

Do not call it directly from browser code. Generate redirect URL on backend, then redirect browser to returned URL.

## Base URLs (placeholders)
Define environment-specific base URLs:

- `KEYCLOAK_BASE_URL`: `https://<internal-domain>/auth/realms/<realm>`
- `ARCHIVE_BASE_URL`: `https://<internal-domain>`
- `INTERNAL_GATEWAY_BASE_URL`: `https://<internal-domain>`

## API 1. Get OIDC Access Token
### Endpoint
`POST {KEYCLOAK_BASE_URL}/protocol/openid-connect/token`

### Recommended grant type
`client_credentials`

### Request
Content-Type: `application/x-www-form-urlencoded`

Required form fields:
- `grant_type=client_credentials`
- `client_id=<integration-client-id>`
- `client_secret=<integration-client-secret>`

### cURL
```bash
curl -X POST "${KEYCLOAK_BASE_URL}/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=${CLIENT_ID}" \
  -d "client_secret=${CLIENT_SECRET}"
```

### Typical response
```json
{
  "access_token": "eyJ...",
  "expires_in": 300,
  "refresh_expires_in": 0,
  "token_type": "Bearer",
  "scope": "openid profile"
}
```

## API 2. Create Document in Archive
### Endpoint
`POST {ARCHIVE_BASE_URL}/api/document/create?documentData=<URL_ENCODED_JSON>`

### Authentication
`Authorization: Bearer <access_token>`

### Request
- Method: `POST`
- Content-Type: `multipart/form-data`
- Multipart field:
  - `file` (required): source file to sign
- Query parameter:
  - `documentData` (required): metadata object serialized as JSON and URL-encoded

### `documentData` object (practical minimum)
- `documentFilename` (string): file name visible in signing flow
- `objectName` (string): business/system object reference
- `documentType` (string): document classification
- `contentType` (string): MIME type, for example `application/pdf`
- `documentData` (object/string): custom metadata payload

Example JSON before URL encoding:
```json
{
  "documentFilename": "contract-2026-001.pdf",
  "objectName": "contract-2026-001",
  "documentType": "Contract",
  "contentType": "application/pdf",
  "documentData": {
    "customerId": "CUST-001",
    "caseId": "CASE-8891"
  }
}
```

### cURL
```bash
DOC_META='{"documentFilename":"contract-2026-001.pdf","objectName":"contract-2026-001","documentType":"Contract","contentType":"application/pdf","documentData":{"customerId":"CUST-001","caseId":"CASE-8891"}}'

curl -X POST "${ARCHIVE_BASE_URL}/api/document/create?documentData=$(python -c 'import urllib.parse,sys;print(urllib.parse.quote(sys.argv[1]))' "$DOC_META")" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -F "file=@./contract-2026-001.pdf"
```

### Success response (example)
```json
{
  "id": "b1ca6de8-f3f1-4f6f-889f-0f0d2f8f4052",
  "externalid": "a45e3a95-0f94-4ef4-9543-50ab3671f52f",
  "archiveName": "FS-MAIN"
}
```

Store `id`. It is the primary identifier for redirect and download APIs.

## API 3. Create Signing Redirect URL
### Endpoint
`POST {INTERNAL_GATEWAY_BASE_URL}/api/auth/session/redirecturl?id=<DOCUMENT_ID>`

### Authentication
`Authorization: Bearer <access_token>`

### Request
- Method: `POST`
- Content-Type: `application/json`
- Query parameter:
  - `id` (required): archive document id from API 2
- Body:
  - JSON object. In standard integration, `{}` is sufficient.

### cURL
```bash
curl -X POST "${INTERNAL_GATEWAY_BASE_URL}/api/auth/session/redirecturl?id=${DOCUMENT_ID}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{}'
```

### Success response
A redirect payload (commonly URL string) that must be used by frontend for browser navigation.

## Frontend Redirect Step
After backend receives redirect payload:

- Return it to frontend (for example, as `{ "redirectUrl": "..." }`).
- Frontend performs browser redirect:
  - `window.location.href = redirectUrl`, or
  - HTTP `302` from backend controller.

Do not attach backend bearer token to browser redirect.

## API 4. Download Signed Document
### Endpoint
`GET {ARCHIVE_BASE_URL}/api/document/{id}/download`

### Authentication
`Authorization: Bearer <access_token>`

### Request parameters
- Path:
  - `id` (required): archive document id
- Query:
  - `version` (optional): specific version when versioning is enabled

### cURL
```bash
curl -L "${ARCHIVE_BASE_URL}/api/document/${DOCUMENT_ID}/download" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  --output signed-${DOCUMENT_ID}.bin
```

### Response
- HTTP `200` with binary file stream.
- `Content-Type` and filename headers depend on archive/storage implementation.

## Error Model and Handling
Implement these minimum rules:

- `400`: invalid parameters or malformed metadata.
- `401` / `403`: token missing, invalid, expired, or insufficient privileges.
- `404`: document id not found.
- `409`: create conflict (duplicate handling, if enabled).
- `5xx`: temporary server-side error.

Retry policy (recommended):
- Token API: up to 2 retries for network/5xx.
- Redirect URL API: up to 2 retries for network/5xx.
- Upload/download APIs: retry only on network/5xx, never blind-retry functional 4xx.

## Minimum Client-Side State to Persist
Persist at least:
- `documentId`
- `clientBusinessId` (your internal reference)
- `redirectUrl` generation timestamp
- signing completion status in your system

## Security Requirements
- Use confidential OIDC client for backend integration.
- Store secrets in secure vault/secret manager.
- Enforce HTTPS and TLS validation.
- Never expose integration client secret to frontend.
- Avoid logging full tokens or sensitive document content.

## Pre-Go-Live Checklist
- Token endpoint reachable from client backend.
- Upload API returns non-empty `id`.
- Redirect URL API returns non-empty redirect payload.
- Browser can open redirect URL and user completes signing.
- User returns to client application successfully.
- Download API returns signed artifact for same `documentId`.
- Expired token and invalid document id paths are tested.

## OpenAPI Discovery (recommended)
Before implementing, confirm contracts in your target SignBox environment:

- Internal Gateway OpenAPI: `GET /v3/api-docs`
- Archive Service OpenAPI: `GET /v3/api-docs`
- Keycloak OIDC discovery: `GET /auth/realms/<realm>/.well-known/openid-configuration`
