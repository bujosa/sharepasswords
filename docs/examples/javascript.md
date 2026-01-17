# JavaScript Examples

Complete examples for using SharePasswords API with JavaScript (Browser and Node.js).

## Browser Example (Web Crypto API)

### Full Implementation

```javascript
/**
 * SharePasswords Client Library
 * Zero-knowledge password sharing
 */

const API_BASE = 'https://api.sharepasswords.com/api/v1';

// ============================================
// CRYPTO UTILITIES
// ============================================

/**
 * Generate a new AES-256-GCM encryption key
 */
async function generateKey() {
  return await crypto.subtle.generateKey(
    { name: 'AES-GCM', length: 256 },
    true, // extractable
    ['encrypt', 'decrypt']
  );
}

/**
 * Export key to base64 string for URL fragment
 */
async function exportKey(key) {
  const keyBytes = await crypto.subtle.exportKey('raw', key);
  return arrayBufferToBase64(keyBytes);
}

/**
 * Import key from base64 string
 */
async function importKey(base64Key) {
  const keyBytes = base64ToArrayBuffer(base64Key);
  return await crypto.subtle.importKey(
    'raw',
    keyBytes,
    { name: 'AES-GCM', length: 256 },
    false, // not extractable
    ['decrypt']
  );
}

/**
 * Encrypt plaintext with AES-256-GCM
 */
async function encrypt(plaintext, key) {
  const encoder = new TextEncoder();
  const data = encoder.encode(plaintext);

  // Generate random 12-byte IV
  const iv = crypto.getRandomValues(new Uint8Array(12));

  // Encrypt
  const ciphertext = await crypto.subtle.encrypt(
    { name: 'AES-GCM', iv },
    key,
    data
  );

  // Combine IV + ciphertext
  const combined = new Uint8Array(iv.length + ciphertext.byteLength);
  combined.set(iv, 0);
  combined.set(new Uint8Array(ciphertext), iv.length);

  return arrayBufferToBase64(combined.buffer);
}

/**
 * Decrypt ciphertext with AES-256-GCM
 */
async function decrypt(encryptedBase64, key) {
  const combined = base64ToArrayBuffer(encryptedBase64);
  const combinedArray = new Uint8Array(combined);

  // Extract IV (first 12 bytes)
  const iv = combinedArray.slice(0, 12);

  // Extract ciphertext (rest)
  const ciphertext = combinedArray.slice(12);

  // Decrypt
  const decrypted = await crypto.subtle.decrypt(
    { name: 'AES-GCM', iv },
    key,
    ciphertext
  );

  const decoder = new TextDecoder();
  return decoder.decode(decrypted);
}

/**
 * Generate a random secret ID
 */
function generateSecretId() {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  const randomValues = crypto.getRandomValues(new Uint8Array(8));
  for (let i = 0; i < 8; i++) {
    result += chars[randomValues[i] % chars.length];
  }
  return result;
}

// ============================================
// BASE64 HELPERS
// ============================================

function arrayBufferToBase64(buffer) {
  const bytes = new Uint8Array(buffer);
  let binary = '';
  for (let i = 0; i < bytes.byteLength; i++) {
    binary += String.fromCharCode(bytes[i]);
  }
  return btoa(binary)
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');
}

function base64ToArrayBuffer(base64) {
  const standardBase64 = base64
    .replace(/-/g, '+')
    .replace(/_/g, '/');
  const binary = atob(standardBase64);
  const bytes = new Uint8Array(binary.length);
  for (let i = 0; i < binary.length; i++) {
    bytes[i] = binary.charCodeAt(i);
  }
  return bytes.buffer;
}

// ============================================
// API CLIENT
// ============================================

/**
 * Create a new secret
 */
async function createSecret(secret, options = {}) {
  const { expiresIn = '24h', maxViews = 1 } = options;

  // Generate key and encrypt
  const key = await generateKey();
  const encryptedContent = await encrypt(secret, key);
  const secretId = generateSecretId();

  // Send to API
  const response = await fetch(`${API_BASE}/secrets`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      secretId,
      encryptedContent,
      expiresIn,
      maxViews
    })
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Failed to create secret');
  }

  // Export key for URL
  const keyBase64 = await exportKey(key);

  // Generate shareable URL
  const shareUrl = `https://sharepasswords.com/s/${secretId}#${keyBase64}`;

  return {
    secretId,
    shareUrl,
    expiresAt: (await response.json()).data.expiresAt
  };
}

/**
 * Retrieve and decrypt a secret
 */
async function getSecret(secretId, keyBase64) {
  // Fetch encrypted content
  const response = await fetch(`${API_BASE}/secrets/${secretId}`);

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Secret not found');
  }

  const { data } = await response.json();

  // Import key and decrypt
  const key = await importKey(keyBase64);
  const plaintext = await decrypt(data.encryptedContent, key);

  return {
    secret: plaintext,
    remainingViews: data.remainingViews
  };
}

/**
 * Check if a secret exists
 */
async function checkSecretExists(secretId) {
  const response = await fetch(`${API_BASE}/secrets/${secretId}/exists`);
  const { data } = await response.json();
  return data.exists;
}

// ============================================
// USAGE EXAMPLES
// ============================================

// Example 1: Create a one-time secret
async function example1() {
  const result = await createSecret('my-super-secret-password', {
    expiresIn: '24h',
    maxViews: 1
  });

  console.log('Share this URL:', result.shareUrl);
  console.log('Expires at:', result.expiresAt);
}

// Example 2: Retrieve a secret from URL
async function example2() {
  // Parse URL: https://sharepasswords.com/s/abc123#keyBase64
  const url = new URL(window.location.href);
  const secretId = url.pathname.split('/').pop();
  const keyBase64 = url.hash.slice(1); // Remove the #

  if (await checkSecretExists(secretId)) {
    const { secret, remainingViews } = await getSecret(secretId, keyBase64);
    console.log('Secret:', secret);
    console.log('Remaining views:', remainingViews);
  } else {
    console.log('Secret has expired or been viewed');
  }
}

// Example 3: Create with unlimited views
async function example3() {
  const result = await createSecret('shared-api-key-for-team', {
    expiresIn: '7d',
    maxViews: null // Unlimited views
  });

  console.log('Team share URL:', result.shareUrl);
}
```

## Node.js Example

For Node.js, you'll need the `node:crypto` module:

```javascript
import crypto from 'node:crypto';

const API_BASE = 'https://api.sharepasswords.com/api/v1';

/**
 * Encrypt using Node.js crypto
 */
function encrypt(plaintext, key) {
  const iv = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);

  const encrypted = Buffer.concat([
    cipher.update(plaintext, 'utf8'),
    cipher.final()
  ]);

  const authTag = cipher.getAuthTag();

  // Combine: IV + ciphertext + authTag
  const combined = Buffer.concat([iv, encrypted, authTag]);
  return combined.toString('base64url');
}

/**
 * Decrypt using Node.js crypto
 */
function decrypt(encryptedBase64, key) {
  const combined = Buffer.from(encryptedBase64, 'base64url');

  const iv = combined.subarray(0, 12);
  const authTag = combined.subarray(-16);
  const ciphertext = combined.subarray(12, -16);

  const decipher = crypto.createDecipheriv('aes-256-gcm', key, iv);
  decipher.setAuthTag(authTag);

  const decrypted = Buffer.concat([
    decipher.update(ciphertext),
    decipher.final()
  ]);

  return decrypted.toString('utf8');
}

/**
 * Generate encryption key
 */
function generateKey() {
  return crypto.randomBytes(32); // 256 bits
}

/**
 * Generate secret ID
 */
function generateSecretId() {
  return crypto.randomBytes(6).toString('base64url');
}

/**
 * Create a secret
 */
async function createSecret(secret, options = {}) {
  const { expiresIn = '24h', maxViews = 1 } = options;

  const key = generateKey();
  const encryptedContent = encrypt(secret, key);
  const secretId = generateSecretId();

  const response = await fetch(`${API_BASE}/secrets`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      secretId,
      encryptedContent,
      expiresIn,
      maxViews
    })
  });

  if (!response.ok) {
    throw new Error('Failed to create secret');
  }

  const keyBase64 = key.toString('base64url');
  const shareUrl = `https://sharepasswords.com/s/${secretId}#${keyBase64}`;

  const { data } = await response.json();

  return {
    secretId,
    shareUrl,
    expiresAt: data.expiresAt
  };
}

/**
 * Get and decrypt a secret
 */
async function getSecret(secretId, keyBase64) {
  const response = await fetch(`${API_BASE}/secrets/${secretId}`);

  if (!response.ok) {
    throw new Error('Secret not found');
  }

  const { data } = await response.json();
  const key = Buffer.from(keyBase64, 'base64url');
  const plaintext = decrypt(data.encryptedContent, key);

  return {
    secret: plaintext,
    remainingViews: data.remainingViews
  };
}

// Usage
async function main() {
  // Create a secret
  const { shareUrl } = await createSecret('my-password-123', {
    expiresIn: '1h',
    maxViews: 1
  });

  console.log('Share URL:', shareUrl);

  // Parse and retrieve
  const url = new URL(shareUrl);
  const secretId = url.pathname.split('/').pop();
  const keyBase64 = url.hash.slice(1);

  const { secret } = await getSecret(secretId, keyBase64);
  console.log('Retrieved secret:', secret);
}

main().catch(console.error);
```

## TypeScript Types

```typescript
interface CreateSecretOptions {
  expiresIn?: '1h' | '24h' | '7d' | '30d';
  maxViews?: number | null;
}

interface CreateSecretResult {
  secretId: string;
  shareUrl: string;
  expiresAt: string;
}

interface GetSecretResult {
  secret: string;
  remainingViews: number | null;
}

interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    message: string;
    code: string;
  };
}
```

## Error Handling

```javascript
async function safeCreateSecret(secret, options) {
  try {
    return await createSecret(secret, options);
  } catch (error) {
    if (error.message.includes('409')) {
      // Secret ID collision, retry with new ID
      return await createSecret(secret, options);
    }
    throw error;
  }
}

async function safeGetSecret(secretId, keyBase64) {
  try {
    return await getSecret(secretId, keyBase64);
  } catch (error) {
    if (error.message.includes('not found')) {
      return { secret: null, error: 'Secret expired or already viewed' };
    }
    if (error.message.includes('decrypt')) {
      return { secret: null, error: 'Invalid encryption key' };
    }
    throw error;
  }
}
```
