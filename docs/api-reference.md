# API Reference

Complete REST API documentation for SharePasswords.

## Base URL

```
Production: https://api.sharepasswords.com/api/v1
Development: http://localhost:3000/api/v1
```

## Authentication

The API is **public** and does not require authentication. Rate limiting may be applied to prevent abuse.

## Response Format

All responses follow a consistent format:

### Success Response

```json
{
  "success": true,
  "data": { ... }
}
```

### Error Response

```json
{
  "success": false,
  "error": {
    "message": "Human-readable error message",
    "code": "ERROR_CODE",
    "details": [
      {
        "field": "fieldName",
        "detail": "Specific validation error"
      }
    ]
  }
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
curl -X POST https://api.sharepasswords.com/api/v1/secrets \
  -H "Content-Type: application/json" \
  -d '{
    "secretId": "a1b2c3d4",
    "encryptedContent": "dGVzdCBlbmNyeXB0ZWQgY29udGVudA==",
    "expiresIn": "24h",
    "maxViews": 1
  }'
```

#### Success Response (200)

```json
{
  "success": true,
  "data": {
    "secretId": "a1b2c3d4",
    "expiresAt": "2024-01-16T12:00:00.000Z"
  }
}
```

#### Error Responses

**400 Bad Request** - Invalid input

```json
{
  "success": false,
  "error": {
    "message": "Validation failed",
    "code": "VALIDATION_ERROR",
    "details": [
      {
        "field": "encryptedContent",
        "detail": "Required"
      }
    ]
  }
}
```

**409 Conflict** - Secret ID already exists

```json
{
  "success": false,
  "error": {
    "message": "Secret with this ID already exists",
    "code": "CONFLICT"
  }
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
curl -X GET https://api.sharepasswords.com/api/v1/secrets/a1b2c3d4
```

#### Success Response (200)

```json
{
  "success": true,
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
  "success": false,
  "error": {
    "message": "Secret not found or expired",
    "code": "NOT_FOUND"
  }
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
curl -X GET https://api.sharepasswords.com/api/v1/secrets/a1b2c3d4/exists
```

#### Success Response (200)

```json
{
  "success": true,
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
curl -X GET https://api.sharepasswords.com/api/v1/health
```

#### Success Response (200)

```json
{
  "status": "ok"
}
```

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
  "success": false,
  "error": {
    "message": "Rate limit exceeded",
    "code": "RATE_LIMITED"
  }
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
const { data } = await api.checkSecretExists(secretId);
if (!data.exists) {
  showError('This secret has expired or been viewed already');
}
```

### 4. Handle Expiration Gracefully

Secrets can expire at any time. Always handle 404 errors:

```javascript
try {
  const secret = await api.getSecret(secretId);
  // Decrypt and display
} catch (error) {
  if (error.code === 'NOT_FOUND') {
    showError('This secret has expired or been viewed already');
  }
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
