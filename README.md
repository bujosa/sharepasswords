# SharePasswords

> Secure, zero-knowledge password sharing platform with encrypted one-time links.

SharePasswords enables users to share sensitive information (passwords, API keys, credentials) via encrypted one-time links. The platform uses **client-side encryption**, ensuring the server **never** has access to plaintext content.

## Key Features

- **Zero-Knowledge Architecture** - Encryption happens 100% client-side
- **AES-256-GCM Encryption** - Industry-standard symmetric encryption
- **One-Time Links** - Auto-destruct after viewing
- **Configurable Expiration** - 1 hour, 24 hours, 7 days, or 30 days
- **View Limits** - Set maximum number of views (1, 5, 10, or unlimited)
- **No Account Required** - Anonymous secret sharing
- **QR Code Generation** - Easy mobile sharing

## How It Works

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   SENDER    │     │   SERVER    │     │  RECIPIENT  │
└─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │
      │ 1. Generate key   │                   │
      │    & encrypt      │                   │
      │    (client-side)  │                   │
      │                   │                   │
      │ 2. Send encrypted │                   │
      │    blob ─────────►│ Store blob only   │
      │                   │                   │
      │ 3. Create URL:    │                   │
      │    /s/{id}#{key}  │                   │
      │                   │                   │
      │ 4. Share URL ─────┼───────────────────┼──►
      │                   │                   │
      │                   │◄──────────────────┤ 5. Request (id only)
      │                   │                   │
      │                   │  Return encrypted ├──►
      │                   │  blob             │
      │                   │                   │ 6. Decrypt with
      │                   │                   │    key from URL
```

The encryption key is stored in the URL fragment (`#`), which is **never sent to the server** - providing true zero-knowledge architecture.

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture.md) | System architecture and design decisions |
| [API Reference](docs/api-reference.md) | Complete API documentation |
| [Security](docs/security.md) | Encryption details and security model |
| [Examples](docs/examples/) | Code examples in multiple languages |

## Quick Example

### Create a Secret (JavaScript)

```javascript
// 1. Generate encryption key
const key = await crypto.subtle.generateKey(
  { name: 'AES-GCM', length: 256 },
  true,
  ['encrypt', 'decrypt']
);

// 2. Encrypt your secret
const iv = crypto.getRandomValues(new Uint8Array(12));
const encoded = new TextEncoder().encode('my-secret-password');
const ciphertext = await crypto.subtle.encrypt(
  { name: 'AES-GCM', iv },
  key,
  encoded
);

// 3. Combine IV + ciphertext and encode
const combined = new Uint8Array(iv.length + ciphertext.byteLength);
combined.set(iv, 0);
combined.set(new Uint8Array(ciphertext), iv.length);
const encryptedContent = btoa(String.fromCharCode(...combined));

// 4. Send to API
const response = await fetch('https://api.sharepasswords.com/api/v1/secrets', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    secretId: crypto.randomUUID(),
    encryptedContent,
    expiresIn: '24h',
    maxViews: 1
  })
});

// 5. Generate shareable URL with key in fragment
const keyBytes = await crypto.subtle.exportKey('raw', key);
const keyBase64 = btoa(String.fromCharCode(...new Uint8Array(keyBytes)));
const shareUrl = `https://sharepasswords.com/s/${secretId}#${keyBase64}`;
```

### Retrieve a Secret (cURL)

```bash
curl -X GET https://api.sharepasswords.com/api/v1/secrets/{secretId}
```

## Community

### Suggestions & Feature Requests

We welcome community input! See [SUGGESTIONS.md](SUGGESTIONS.md) for how to submit feature requests and suggestions via Pull Requests.

### Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

## Technology Stack

| Component | Technology |
|-----------|------------|
| Backend | NestJS 11 (Node.js) |
| Database | MongoDB |
| Frontend | Svelte 5 |
| Encryption | Web Crypto API (AES-256-GCM) |

