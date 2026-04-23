# SharePassword

> Secure, zero-knowledge password sharing platform with encrypted one-time links and organization vaults.

SharePassword enables users to share sensitive information (passwords, API keys, credentials) via encrypted one-time links. The platform uses **client-side encryption**, ensuring the server **never** has access to plaintext content.

SharePassword also includes an early encrypted team vault: users can create an account, get an organization automatically, invite teammates, and store encrypted notes or small files with version history.

## Key Features

- **Zero-Knowledge Architecture** - Encryption happens 100% client-side
- **AES-256-GCM Encryption** - Industry-standard symmetric encryption
- **One-Time Links** - Auto-destruct after viewing
- **Configurable Expiration** - 1 hour, 24 hours, 7 days, or 30 days
- **View Limits** - Set maximum number of views (1, 5, 10, or unlimited)
- **No Account Required** - Anonymous secret sharing
- **QR Code Generation** - Easy mobile sharing
- **Accounts & Organizations** - Users get an organization and admin membership on registration
- **Encrypted Team Vault** - Store encrypted notes and files in an organization vault
- **Invite Links** - Organization admins can invite admins or normal users
- **Version History** - Vault item updates create new encrypted versions
- **Private File Storage** - Encrypted file payloads are stored in Google Cloud Storage, while MongoDB stores metadata
- **Backoffice Dashboard** - Dedicated admin UI for stats, users, waitlist, organizations, registration gates, and maintenance mode

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

## Vault Security Model

The vault keeps the same core principle as one-time links: plaintext should not be sent to the backend.

```
Browser
  ├─ Loads a local vault key
  ├─ Encrypts note or file payload with AES-GCM
  └─ Sends ciphertext to API

API / MongoDB
  ├─ Stores users, organizations, memberships, invites
  ├─ Stores vault item metadata and encrypted names
  ├─ Stores version records
  └─ Stores GCS object paths for encrypted files

Google Cloud Storage
  └─ Stores encrypted file payloads only
```

Current vault file limits:

- Raw file upload limit: `40KB`
- Encrypted payload API limit: `96KB`
- Storage path format: `vaults/{organizationId}/{vaultId}/{itemId}/versions/{versionId}.bin`

Important security note: the first vault release uses a browser-managed vault key. This keeps plaintext away from the server, but approved-device key transfer, mandatory MFA, and recovery-key based vault unlock are planned hardening layers before positioning the vault as a complete 1Password-style replacement.

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
const secretId = crypto.randomUUID();
const response = await fetch('https://api.sharepasswords.com/v1/secrets', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    secretId,
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
curl https://api.sharepasswords.com/v1/secrets/{secretId}
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
| Backoffice | Svelte 5 + Cloud Run |
| Encryption | Web Crypto API (AES-256-GCM) |
| File Storage | Google Cloud Storage |
