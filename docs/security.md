# Security

This document details the security model and encryption implementation of SharePassword.

## Zero-Knowledge Architecture

SharePassword implements a **zero-knowledge** architecture, meaning the server **never** has access to your plaintext secrets.

This applies in two modes:

- **One-time links**: the encryption key lives in the URL fragment and is never sent to the server.
- **Vault items**: notes and files are encrypted in the browser before upload; the backend stores ciphertext, metadata, permissions, versions, and private storage paths.

### How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│                        YOUR BROWSER                               │
│                                                                   │
│   1. You enter your secret: "my-password-123"                    │
│                                                                   │
│   2. Browser generates encryption key (AES-256-GCM)              │
│      Key: "x7Kj9mPq..."                                          │
│                                                                   │
│   3. Browser encrypts your secret                                 │
│      Plaintext: "my-password-123"                                │
│      Ciphertext: "a8Xk2pL..." (unreadable without key)          │
│                                                                   │
│   4. Browser sends ONLY ciphertext to server                     │
│      (Key stays in browser)                                       │
└───────────────────────────┬──────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│                        OUR SERVER                                 │
│                                                                   │
│   Server receives: "a8Xk2pL..."                                  │
│                                                                   │
│   Server knows: ❌ Nothing about your secret                     │
│   Server can: ❌ NOT decrypt your data                           │
│   Server stores: ✅ Only encrypted blob                          │
└──────────────────────────────────────────────────────────────────┘
```

### URL Fragment Security

The encryption key is stored in the **URL fragment** (the part after `#`):

```
https://sharepasswords.com/s/abc123#x7Kj9mPqR2nL5vT8...
                                   │
                                   └── This NEVER goes to our server
```

**Why is this secure?**

Per the HTTP specification (RFC 3986), the fragment identifier is processed entirely client-side and is **never** included in HTTP requests. When you visit this URL:

1. Your browser sends a request for `/s/abc123`
2. The fragment `#x7Kj9mPq...` stays in the browser
3. JavaScript reads the fragment and decrypts locally

## Encryption Specifications

### Algorithm: AES-256-GCM

| Property | Value |
|----------|-------|
| Algorithm | AES (Advanced Encryption Standard) |
| Mode | GCM (Galois/Counter Mode) |
| Key Size | 256 bits |
| IV Size | 96 bits (12 bytes) |
| Auth Tag | 128 bits |

### Why AES-256-GCM?

- **Industry Standard**: Used by banks, governments, and security-conscious organizations worldwide
- **Authenticated Encryption**: GCM mode provides both confidentiality AND integrity
- **Tamper Detection**: Any modification to the ciphertext will cause decryption to fail
- **Performance**: Hardware-accelerated on modern CPUs (AES-NI)

### Implementation

We use the **Web Crypto API**, a native browser API for cryptographic operations:

```javascript
// Key Generation
const key = await crypto.subtle.generateKey(
  { name: 'AES-GCM', length: 256 },
  true,  // extractable (needed to share key)
  ['encrypt', 'decrypt']
);

// Encryption
const iv = crypto.getRandomValues(new Uint8Array(12));
const ciphertext = await crypto.subtle.encrypt(
  { name: 'AES-GCM', iv },
  key,
  plaintext
);

// Decryption
const plaintext = await crypto.subtle.decrypt(
  { name: 'AES-GCM', iv },
  key,
  ciphertext
);
```

## Data Format

### Encrypted Blob Structure

```
┌─────────────┬──────────────────────────┬─────────────┐
│   IV        │      Ciphertext          │  Auth Tag   │
│  12 bytes   │     Variable length      │  16 bytes   │
└─────────────┴──────────────────────────┴─────────────┘
         │
         └── All encoded together as Base64
```

### Key Export Format

The encryption key is exported as raw bytes and encoded as **URL-safe Base64**:

```javascript
const keyBytes = await crypto.subtle.exportKey('raw', key);
const keyBase64 = btoa(String.fromCharCode(...new Uint8Array(keyBytes)));
```

## Vault Security Model

The team vault adds accounts, organizations, invites, roles, file uploads, and version history while keeping sensitive contents encrypted before they leave the browser.

### Vault Data Flow

```
Browser
  ├─ Loads a local vault key
  ├─ Encrypts note/file payload using AES-GCM
  └─ Sends ciphertext to API

API
  ├─ Authenticates the user
  ├─ Verifies organization membership
  ├─ Stores metadata and version records in MongoDB
  └─ Writes encrypted file payloads to private GCS

Storage
  ├─ MongoDB: users, orgs, memberships, invites, encrypted names, version metadata
  └─ GCS: encrypted file payloads only
```

### Vault Access Control

| Control | Description |
|---------|-------------|
| Organization membership | Vault access is scoped by organization membership |
| Roles | Organization admins can create invite links and choose `admin` or `user` |
| Auth token | Authenticated vault routes require a bearer token |
| Private bucket | GCS bucket uses uniform bucket-level access and public access prevention |
| Version history | Updates create new encrypted versions instead of overwriting history |

### Current Device Model

The first vault release uses a browser-managed vault key. This keeps plaintext away from the server, but it means vault unlock is tied to the browser/device holding that key.

Planned hardening:

- Mandatory MFA for account access
- Approved-device key transfer
- Recovery-key based vault unlock
- Stronger account recovery flow

## Threat Model

### What We Protect Against

| Threat | Protection |
|--------|------------|
| **Server Compromise** | Server only has encrypted data; cannot decrypt without key |
| **Database Leak** | Encrypted blobs are useless without keys |
| **Man-in-the-Middle** | HTTPS encrypts transport; fragments not sent to server |
| **Brute Force** | 256-bit keys have 2^256 possible combinations |
| **Tamper Attacks** | GCM authentication tag detects modifications |

### What We Don't Protect Against

| Threat | Reason |
|--------|--------|
| **Compromised Sender Device** | If attacker has access to sender's device, they can see plaintext |
| **Compromised Recipient Device** | If attacker has access to recipient's device, they can see plaintext |
| **Link Interception** | If someone intercepts the URL, they can view the secret |
| **Screen Recording** | Physical access to the screen during viewing |

### Mitigation Recommendations

1. **Use one-time links** - Set `maxViews: 1` so the secret is deleted after viewing
2. **Short expiration** - Use `1h` or `24h` expiration when possible
3. **Verify recipient** - Confirm the recipient received and viewed the secret
4. **Secure communication** - Share links over encrypted channels (Signal, encrypted email)

## Data Retention

### What We Store

| Data | Stored | Duration |
|------|--------|----------|
| Encrypted content | Yes | Until expiration or max views |
| Encryption key | **No** | Never touches our servers |
| Encrypted vault files | Yes | Until removed from the vault |
| Vault metadata | Yes | Required for org access and version history |
| IP addresses | No | Not logged |
| User accounts | Optional | Required only for the vault |
| Analytics | Minimal | Aggregate only, no PII |

### Automatic Deletion

Secrets are automatically deleted when:

1. **Time expires** - MongoDB TTL index removes expired documents
2. **Max views reached** - Application deletes after final view
3. **Manual deletion** - (Future feature)

## Security Best Practices

### For Users

1. **Verify the URL** - Ensure you're on `sharepasswords.com`
2. **Use short expiration** - Don't leave secrets available longer than needed
3. **Use one-time links** - Prevent multiple viewings
4. **Share links securely** - Use encrypted messaging apps
5. **Confirm receipt** - Verify the recipient received the secret

### For Self-Hosting

1. **Use HTTPS** - Always enable TLS in production
2. **Secure MongoDB** - Enable authentication and encryption at rest
3. **Firewall** - Restrict database access to backend only
4. **Updates** - Keep dependencies updated
5. **Monitoring** - Monitor for unusual access patterns

## Compliance

### GDPR Considerations

- **No PII stored** - We don't collect personal information
- **Right to erasure** - Secrets auto-delete; nothing to retain
- **Data minimization** - We only store what's necessary (encrypted blobs)

### Security Audits

We welcome security researchers to review our implementation:

- **Source Code**: Available on GitHub
- **Responsible Disclosure**: security@sharepasswords.com
- **Bug Bounty**: Coming soon

## Cryptographic References

- [AES-GCM - NIST SP 800-38D](https://csrc.nist.gov/publications/detail/sp/800-38d/final)
- [Web Crypto API - W3C](https://www.w3.org/TR/WebCryptoAPI/)
- [RFC 3986 - URI Generic Syntax (Fragment Identifier)](https://www.rfc-editor.org/rfc/rfc3986#section-3.5)

## Frequently Asked Questions

### Can you see my secrets?

**No.** We never receive the encryption key. We only store encrypted data that we cannot decrypt.

### What if your servers are hacked?

Attackers would only get encrypted blobs. Without the keys (which we never have), the data is useless.

### Is the encryption really happening in my browser?

**Yes.** You can verify by:
1. Opening browser DevTools (F12)
2. Going to the Network tab
3. Creating a secret
4. Observing that only encrypted content is sent

### Why not use end-to-end encrypted messaging instead?

SharePasswords is useful when:
- You need to share with someone not on your messaging platform
- You want the secret to auto-delete after viewing
- You need a link-based sharing mechanism
- You want to avoid the secret being stored in message history
