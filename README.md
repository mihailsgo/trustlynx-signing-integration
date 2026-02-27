# SignBox Integration Guide: Archive -> Internal Signing -> Download

## 1. Scope
This guide describes a production-grade integration pattern for client systems that need to:

1. Upload a document to SignBox archive.
2. Start an internal signing session.
3. Redirect the end user to SignBox signing UI.
4. Receive the user back in client system.
5. Download the signed document.

This document is written for backend and frontend developers and is safe to reuse for future clients.

## 2. What Was Validated
The following APIs were validated from running services in Evolution environment:

- Internal portal gateway:
  - `POST /api/auth/session/redirecturl`
  - `POST /api/auth/session/redirecturl/`
- Archive service:
  - `POST /api/document/create`
  - `GET /api/document/{id}/download`
  - `POST /api/document/find`
- Internal Keycloak OIDC discovery:
  - `/.well-known/openid-configuration` for realm `TrustLynx`

Important observed behavior:

- `POST /api/auth/session/redirecturl` returns `401` without bearer token.
- Browser redirect cannot include custom `Authorization` header.

Because of this, **redirect URL creation must happen server-to-server** (your backend -> SignBox API), then browser opens the returned URL.

## 3. High-Level Flow
```text
Client Backend
   -> (1) Upload document to SignBox Archive
   <- documentId
   -> (2) Get bearer token from Internal Keycloak (client credentials recommended)
   <- access_token
   -> (3) Call Internal Gateway /api/auth/session/redirecturl?id={documentId}
      with Authorization: Bearer {access_token}
   <- signingRedirectUrl

Client Frontend
   -> (4) HTTP 302 / window.location = signingRedirectUrl
User signs in SignBox
SignBox
   -> (5) Redirect user back to client return URL (configured in SignBox process/profile)

Client Backend
   -> (6) Download signed document from Archive by id
   <- signed file bytes
```

## 4. Endpoints and Contracts

## 4.1 Internal Keycloak token endpoint
Use OIDC discovery first, then token endpoint from discovery document.

- Discovery:
  - `https://<INTERNAL_DOMAIN>/auth/realms/<REALM>/.well-known/openid-configuration`
- Token endpoint (from discovery):
  - `.../protocol/openid-connect/token`

Recommended grant:

- `client_credentials` (service-to-service)

Avoid using user password grant for integrations.

## 4.2 Upload document to archive
`POST /api/document/create`

- Content type: `multipart/form-data`
- Part:
  - `file` (binary, required)
- Query parameter:
  - `documentData` (object metadata; service expects `DocumentData` model)

Response:

- `200` with created object containing:
  - `id` (this is the primary document identifier for next steps)

## 4.3 Create signing redirect URL
`POST /api/auth/session/redirecturl?id=<documentId>`

Notes:

- Current OpenAPI parameter name is `id` (query).
- Some prior communication may refer to `documentId`; use actual deployed contract (`id`) unless your build differs.
- Requires `Authorization: Bearer <token>`.
- Request body: JSON object (can be `{}` if no extra payload is required by your profile/build).

Response:

- `200` with redirect payload (treat as opaque value; in most builds it resolves to URL string).

## 4.4 Download signed file
`GET /api/document/{id}/download`

Query:

- `version` (optional)

Response:

- `200` file stream (signed version depending on process/status).

## 5. Critical Implementation Rule
Do **not** call `/api/auth/session/redirecturl` from browser code.

Reason:

- It needs bearer token in `Authorization` header.
- Browser navigation/redirect to final page does not preserve your custom API auth header.

Correct pattern:

1. Backend calls `/api/auth/session/redirecturl` with bearer.
2. Backend returns `signingRedirectUrl` to frontend.
3. Frontend navigates user to this URL.

## 6. Reference Backend Sequence (Pseudo-code)
```text
function startSigning(file, metadata):
  archiveResp = POST /api/document/create (multipart + documentData)
  documentId = archiveResp.id

  token = POST keycloak token endpoint (client_credentials)

  redirectResp = POST /api/auth/session/redirecturl?id=documentId
                 Authorization: Bearer token
                 Body: {}

  return { documentId, redirectUrl: redirectResp }
```

## 7. cURL Examples

## 7.1 Get token (client credentials)
```bash
curl -X POST "https://<INTERNAL_DOMAIN>/auth/realms/<REALM>/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=<CLIENT_ID>" \
  -d "client_secret=<CLIENT_SECRET>"
```

## 7.2 Upload document
```bash
curl -X POST "https://<INTERNAL_DOMAIN>/api/document/create?documentData=<URL_ENCODED_JSON>" \
  -H "Authorization: Bearer <TOKEN>" \
  -F "file=@./contract.pdf"
```

## 7.3 Create redirect URL
```bash
curl -X POST "https://<INTERNAL_DOMAIN>/api/auth/session/redirecturl?id=<DOCUMENT_ID>" \
  -H "Authorization: Bearer <TOKEN>" \
  -H "Content-Type: application/json" \
  -d "{}"
```

## 7.4 Download signed file
```bash
curl -L "https://<INTERNAL_DOMAIN>/api/document/<DOCUMENT_ID>/download" \
  -H "Authorization: Bearer <TOKEN>" \
  --output signed-document.bin
```

## 8. Error Handling
Handle at least:

- `401` / `403`: token missing/expired/wrong scope/client.
- `404`: unknown document id.
- `409`: duplicate create (for idempotency logic).
- `5xx`: transient; retry with exponential backoff for technical calls.

Recommended retries:

- Token call: retry up to 2 times on network/5xx.
- Redirect URL call: retry up to 2 times on network/5xx.
- Archive upload/download: retry on network/5xx only (not on functional 4xx).

## 9. Security Requirements

- Use confidential Keycloak client for integration backend.
- Keep client secret in secrets manager, never in frontend.
- Use HTTPS only.
- Log correlation IDs, not sensitive payload content.
- Rotate credentials regularly.

## 10. UAT Checklist

- Archive upload returns `id`.
- Redirect URL call returns `200` with non-empty payload.
- User can open redirect URL and complete signing.
- User is returned to client application.
- Download endpoint returns signed artifact.
- Expired token returns `401` (expected).
- Invalid document id returns `404` (expected).

## 11. Versioning Note
This guide reflects currently validated contracts in deployed Evolution environment.  
Before go-live, always confirm endpoint contract against your environment OpenAPI:

- Internal gateway: `/v3/api-docs`
- Archive service: `/v3/api-docs`
- Authentication service: `/v3/api-docs`
