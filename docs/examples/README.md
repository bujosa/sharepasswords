# Usage Examples

This directory contains code examples for integrating with the SharePasswords API in various programming languages.

## Available Examples

| Language | File | Description |
|----------|------|-------------|
| JavaScript/Node.js | [javascript.md](javascript.md) | Browser and Node.js examples |
| Python | [python.md](python.md) | Python with cryptography library |
| cURL | [curl.md](curl.md) | Command-line examples |

## Important Notes

### Client-Side Encryption

All examples follow the zero-knowledge architecture:

1. **Generate** encryption key locally
2. **Encrypt** secret locally
3. **Send** only encrypted data to API
4. **Never** send the encryption key to the server

### URL Fragment

The encryption key should always be placed in the URL **fragment** (after `#`):

```
https://sharepasswords.com/s/{secretId}#{encryptionKey}
```

The fragment is never sent to the server, maintaining zero-knowledge security.

## Quick Reference

### Create a Secret

```
POST /api/v1/secrets
Content-Type: application/json

{
  "secretId": "unique-id",
  "encryptedContent": "base64-encoded-encrypted-data",
  "expiresIn": "24h",
  "maxViews": 1
}
```

### Retrieve a Secret

```
GET /api/v1/secrets/{secretId}
```

### Check if Secret Exists

```
GET /api/v1/secrets/{secretId}/exists
```
