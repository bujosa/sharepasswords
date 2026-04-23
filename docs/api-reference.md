# API Reference

Complete REST API documentation for SharePassword.

## Base URL

```
Production: https://api.sharepasswords.com/v1
Development: http://localhost:3000/v1
```

## Authentication

One-time secret endpoints are public and do not require authentication. Vault and invite management endpoints require a bearer token returned by `/auth/login` or `/auth/register`.

```http
Authorization: Bearer <token>
```

## Response Format

All responses follow a consistent format:

### Success Response

```json
{
  "status": "ok",
  "data": { ... }
}
```

### Error Response

```json
{
  "status": "failed",
  "code": "ERROR_CODE",
  "message": "Human-readable error message",
  "errors": [
    {
      "field": "fieldName",
      "detail": "Specific validation error"
    }
  ]
}
```

## Endpoints

### Create Secret

Create a new encrypted secret.

```
POST /secrets
```

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `secretId` | string | Yes | Unique identifier (client-generated UUID) |
| `encryptedContent` | string | Yes | Base64-encoded encrypted content |
| `expiresIn` | enum | Yes | Expiration time: `1h`, `24h`, `7d`, `30d` |
| `maxViews` | number \| null | Yes | Maximum views allowed, or `null` for unlimited |

#### Example Request

```bash
curl -X POST https://api.sharepasswords.com/v1/secrets \
  -H "Content-Type: application/json" \
  -d '{
    "secretId": "a1b2c3d4",
    "encryptedContent": "dGVzdCBlbmNyeXB0ZWQgY29udGVudA==",
    "expiresIn": "24h",
    "maxViews": 1
  }'
```

#### Success Response (201)

```json
{
  "status": "ok",
  "data": {
    "secretId": "a1b2c3d4",
    "expiresAt": "2026-01-17T12:00:00.000Z"
  }
}
```

#### Error Responses

**400 Bad Request** - Invalid input

```json
{
  "status": "failed",
  "code": "VALIDATION_ERROR",
  "message": "Validation failed",
  "errors": [
    {
      "field": "encryptedContent",
      "detail": "Required"
    }
  ]
}
```

**409 Conflict** - Secret ID already exists

```json
{
  "status": "failed",
  "code": "CONFLICT",
  "message": "Secret with this ID already exists"
}
```

---

### Get Secret

Retrieve and consume an encrypted secret. This operation increments the view count and may trigger deletion if max views is reached.

```
GET /secrets/:secretId
```

#### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `secretId` | string | The secret's unique identifier |

#### Example Request

```bash
curl https://api.sharepasswords.com/v1/secrets/a1b2c3d4
```

#### Success Response (200)

```json
{
  "status": "ok",
  "data": {
    "encryptedContent": "dGVzdCBlbmNyeXB0ZWQgY29udGVudA==",
    "remainingViews": 0
  }
}
```

**Note:** `remainingViews` is `null` if the secret has unlimited views.

#### Error Responses

**404 Not Found** - Secret doesn't exist or has expired

```json
{
  "status": "failed",
  "code": "NOT_FOUND",
  "message": "Secret not found or expired"
}
```

---

### Check Secret Exists

Check if a secret exists without consuming a view. Useful for UI validation before showing the reveal button.

```
GET /secrets/:secretId/exists
```

#### Path Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `secretId` | string | The secret's unique identifier |

#### Example Request

```bash
curl https://api.sharepasswords.com/v1/secrets/a1b2c3d4/exists
```

#### Success Response (200)

```json
{
  "status": "ok",
  "data": {
    "exists": true
  }
}
```

---

### Health Check

Check API health status.

```
GET /health
```

#### Example Request

```bash
curl https://api.sharepasswords.com/health
```

#### Success Response (200)

```json
{
  "status": "ok"
}
```

---

## Auth Endpoints

### Register

```
POST /auth/register
```

Creates a user, an organization, an admin membership, and a personal vault.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | Yes | User email |
| `name` | string | Yes | User display name |
| `password` | string | Yes | Minimum 12 characters |
| `organizationName` | string | No | Organization name |

### Login

```
POST /auth/login
```

Returns an auth token and active memberships.

### Current User

```
GET /auth/me
```

Requires bearer auth.

### Invites

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/invites` | Create an organization invite |
| GET | `/auth/invites/:token` | Preview an invite |
| POST | `/auth/invites/:token/accept` | Accept an invite |

Admins can create invite links for `admin` or `user` roles.

---

## Vault Endpoints

Vault routes require bearer auth. Access is scoped by active organization membership.

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/vaults` | List accessible vaults |
| GET | `/vaults/:vaultId/items` | List vault items |
| POST | `/vaults/:vaultId/items` | Create encrypted vault item |
| GET | `/vaults/:vaultId/items/:itemId/versions` | List item versions |
| POST | `/vaults/:vaultId/items/:itemId/versions` | Create a new encrypted version |

### Create Vault Item

```json
{
  "type": "file",
  "encryptedName": "base64-url-safe-ciphertext",
  "encryptedPayload": "base64-url-safe-ciphertext",
  "encryptedSizeBytes": 58212
}
```

Current limits:

- Raw file upload limit in UI: `40KB`
- Encrypted payload API limit: `96KB`
- Encrypted file payloads are stored in private GCS

---

## Data Types

### SecretId

A unique identifier for secrets. Should be generated client-side to avoid the server knowing which secrets are related.

**Recommended format:** 8+ character alphanumeric string or UUID

```javascript
// Option 1: Short ID
const secretId = crypto.randomUUID().slice(0, 8);

// Option 2: Full UUID
const secretId = crypto.randomUUID();
```

### EncryptedContent

Base64-encoded encrypted data. The format is:

```
| IV (12 bytes) | Ciphertext (variable) | Auth Tag (16 bytes) |
```

All concatenated and encoded as base64.

### ExpiresIn

Enum with the following values:

| Value | Duration |
|-------|----------|
| `1h` | 1 hour |
| `24h` | 24 hours |
| `7d` | 7 days |
| `30d` | 30 days |

### MaxViews

Either a positive integer or `null`:

| Value | Meaning |
|-------|---------|
| `1` | One-time secret (deleted after first view) |
| `5` | Can be viewed up to 5 times |
| `10` | Can be viewed up to 10 times |
| `null` | Unlimited views until expiration |

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400 | Request body failed validation |
| `NOT_FOUND` | 404 | Secret not found or expired |
| `CONFLICT` | 409 | Secret ID already exists |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## Rate Limiting

The API may implement rate limiting to prevent abuse:

| Limit | Value |
|-------|-------|
| Requests per minute | 60 |
| Secrets per hour per IP | 100 |

When rate limited, you'll receive:

```json
{
  "status": "failed",
  "code": "RATE_LIMITED",
  "message": "Rate limit exceeded"
}
```

**HTTP Status:** 429 Too Many Requests

---

## CORS

The API allows cross-origin requests from any origin:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Allow-Headers: Content-Type
```

---

## Best Practices

### 1. Generate Secret IDs Client-Side

Always generate the `secretId` on the client to prevent the server from correlating requests:

```javascript
const secretId = crypto.randomUUID().slice(0, 8);
```

### 2. Use HTTPS

Always use HTTPS in production to protect the encrypted content in transit.

### 3. Validate Before Reveal

Use the `/exists` endpoint before showing the "Reveal" button to provide a better user experience:

```javascript
const response = await fetch(`https://api.sharepasswords.com/v1/secrets/${secretId}/exists`);
const { data } = await response.json();
if (!data.exists) {
  showError('This secret has expired or been viewed already');
}
```

### 4. Handle Expiration Gracefully

Secrets can expire at any time. Always handle 404 errors:

```javascript
try {
  const response = await fetch(`https://api.sharepasswords.com/v1/secrets/${secretId}`);
  if (!response.ok) {
    if (response.status === 404) {
      showError('This secret has expired or been viewed already');
      return;
    }
  }
  const { data } = await response.json();
  // Decrypt and display
} catch (error) {
  showError('Failed to retrieve secret');
}
```

### 5. Don't Store Keys Server-Side

Never send the encryption key to the server. Keep it in the URL fragment:

```javascript
// GOOD: Key in fragment (never sent to server)
const url = `https://sharepasswords.com/s/${secretId}#${key}`;

// BAD: Key in query string (sent to server)
const url = `https://sharepasswords.com/s/${secretId}?key=${key}`;
```
