# Python Examples

Complete examples for using SharePasswords API with Python.

## Requirements

```bash
pip install cryptography requests
```

## Full Implementation

```python
"""
SharePasswords Client Library
Zero-knowledge password sharing
"""

import os
import base64
import secrets
import requests
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

API_BASE = "https://api.sharepasswords.com/v1"


# ============================================
# CRYPTO UTILITIES
# ============================================

def generate_key() -> bytes:
    """Generate a new AES-256 encryption key (32 bytes)."""
    return AESGCM.generate_key(bit_length=256)


def encrypt(plaintext: str, key: bytes) -> str:
    """
    Encrypt plaintext with AES-256-GCM.
    Returns base64url-encoded IV + ciphertext + auth tag.
    """
    aesgcm = AESGCM(key)

    # Generate random 12-byte IV
    iv = os.urandom(12)

    # Encrypt (GCM automatically appends auth tag)
    ciphertext = aesgcm.encrypt(iv, plaintext.encode('utf-8'), None)

    # Combine IV + ciphertext (auth tag is already included)
    combined = iv + ciphertext

    # Encode as base64url
    return base64.urlsafe_b64encode(combined).decode('ascii').rstrip('=')


def decrypt(encrypted_base64: str, key: bytes) -> str:
    """
    Decrypt AES-256-GCM encrypted content.
    """
    # Add padding if needed
    padded = encrypted_base64 + '=' * (4 - len(encrypted_base64) % 4)
    combined = base64.urlsafe_b64decode(padded)

    # Extract IV (first 12 bytes)
    iv = combined[:12]

    # Extract ciphertext + auth tag (rest)
    ciphertext = combined[12:]

    # Decrypt
    aesgcm = AESGCM(key)
    plaintext = aesgcm.decrypt(iv, ciphertext, None)

    return plaintext.decode('utf-8')


def generate_secret_id() -> str:
    """Generate a random 8-character secret ID."""
    chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
    return ''.join(secrets.choice(chars) for _ in range(8))


def key_to_base64(key: bytes) -> str:
    """Convert key to base64url string for URL fragment."""
    return base64.urlsafe_b64encode(key).decode('ascii').rstrip('=')


def base64_to_key(key_base64: str) -> bytes:
    """Convert base64url string back to key bytes."""
    padded = key_base64 + '=' * (4 - len(key_base64) % 4)
    return base64.urlsafe_b64decode(padded)


# ============================================
# API CLIENT
# ============================================

class SharePasswordsClient:
    """Client for SharePasswords API."""

    def __init__(self, api_base: str = API_BASE):
        self.api_base = api_base

    def create_secret(
        self,
        secret: str,
        expires_in: str = "24h",
        max_views: int | None = 1
    ) -> dict:
        """
        Create a new encrypted secret.

        Args:
            secret: The plaintext secret to share
            expires_in: Expiration time ('1h', '24h', '7d', '30d')
            max_views: Maximum views (None for unlimited)

        Returns:
            dict with secretId, shareUrl, and expiresAt
        """
        # Generate key and encrypt locally
        key = generate_key()
        encrypted_content = encrypt(secret, key)
        secret_id = generate_secret_id()

        # Send to API
        response = requests.post(
            f"{self.api_base}/secrets",
            json={
                "secretId": secret_id,
                "encryptedContent": encrypted_content,
                "expiresIn": expires_in,
                "maxViews": max_views
            }
        )

        if not response.ok:
            error = response.json().get('error', {})
            raise Exception(error.get('message', 'Failed to create secret'))

        # Generate shareable URL with key in fragment
        key_base64 = key_to_base64(key)
        share_url = f"https://sharepasswords.com/s/{secret_id}#{key_base64}"

        data = response.json().get('data', {})

        return {
            "secretId": secret_id,
            "shareUrl": share_url,
            "expiresAt": data.get('expiresAt')
        }

    def get_secret(self, secret_id: str, key_base64: str) -> dict:
        """
        Retrieve and decrypt a secret.

        Args:
            secret_id: The secret's unique identifier
            key_base64: The base64url-encoded encryption key

        Returns:
            dict with secret and remainingViews
        """
        response = requests.get(f"{self.api_base}/secrets/{secret_id}")

        if not response.ok:
            error = response.json().get('error', {})
            raise Exception(error.get('message', 'Secret not found'))

        data = response.json().get('data', {})

        # Decrypt locally
        key = base64_to_key(key_base64)
        plaintext = decrypt(data['encryptedContent'], key)

        return {
            "secret": plaintext,
            "remainingViews": data.get('remainingViews')
        }

    def check_exists(self, secret_id: str) -> bool:
        """
        Check if a secret exists without consuming a view.

        Args:
            secret_id: The secret's unique identifier

        Returns:
            True if secret exists, False otherwise
        """
        response = requests.get(f"{self.api_base}/secrets/{secret_id}/exists")
        data = response.json().get('data', {})
        return data.get('exists', False)

    @staticmethod
    def parse_share_url(url: str) -> tuple[str, str]:
        """
        Parse a share URL to extract secret_id and key.

        Args:
            url: The full share URL

        Returns:
            Tuple of (secret_id, key_base64)
        """
        from urllib.parse import urlparse

        parsed = urlparse(url)
        secret_id = parsed.path.split('/')[-1]
        key_base64 = parsed.fragment

        return secret_id, key_base64


# ============================================
# USAGE EXAMPLES
# ============================================

def example_create_one_time_secret():
    """Example: Create a one-time secret."""
    client = SharePasswordsClient()

    result = client.create_secret(
        secret="my-super-secret-password",
        expires_in="24h",
        max_views=1
    )

    print(f"Share this URL: {result['shareUrl']}")
    print(f"Expires at: {result['expiresAt']}")

    return result['shareUrl']


def example_retrieve_secret(share_url: str):
    """Example: Retrieve a secret from URL."""
    client = SharePasswordsClient()

    # Parse the URL
    secret_id, key_base64 = client.parse_share_url(share_url)

    # Check if it exists first
    if client.check_exists(secret_id):
        result = client.get_secret(secret_id, key_base64)
        print(f"Secret: {result['secret']}")
        print(f"Remaining views: {result['remainingViews']}")
    else:
        print("Secret has expired or been viewed")


def example_create_team_secret():
    """Example: Create a secret for team sharing."""
    client = SharePasswordsClient()

    result = client.create_secret(
        secret="team-api-key-xyz123",
        expires_in="7d",
        max_views=None  # Unlimited views
    )

    print(f"Team share URL: {result['shareUrl']}")


def example_cli_tool():
    """Example: Simple CLI tool."""
    import sys

    client = SharePasswordsClient()

    if len(sys.argv) < 2:
        print("Usage:")
        print("  python sharepasswords.py create <secret> [expires_in] [max_views]")
        print("  python sharepasswords.py get <url>")
        return

    command = sys.argv[1]

    if command == "create":
        secret = sys.argv[2]
        expires_in = sys.argv[3] if len(sys.argv) > 3 else "24h"
        max_views = int(sys.argv[4]) if len(sys.argv) > 4 else 1

        result = client.create_secret(secret, expires_in, max_views)
        print(f"\nShare URL: {result['shareUrl']}\n")

    elif command == "get":
        url = sys.argv[2]
        secret_id, key_base64 = client.parse_share_url(url)

        result = client.get_secret(secret_id, key_base64)
        print(f"\nSecret: {result['secret']}\n")


if __name__ == "__main__":
    # Run examples
    print("=== Creating one-time secret ===")
    url = example_create_one_time_secret()

    print("\n=== Retrieving secret ===")
    example_retrieve_secret(url)
```

## Using as a Module

```python
from sharepasswords import SharePasswordsClient

# Initialize client
client = SharePasswordsClient()

# Create a secret
result = client.create_secret(
    secret="database-password-123",
    expires_in="1h",
    max_views=1
)

# Share the URL with someone
print(f"Send this link: {result['shareUrl']}")

# They can retrieve it with:
secret_id, key = client.parse_share_url(result['shareUrl'])
decrypted = client.get_secret(secret_id, key)
print(f"Password: {decrypted['secret']}")
```

## Error Handling

```python
from sharepasswords import SharePasswordsClient

client = SharePasswordsClient()

def safe_create(secret: str) -> dict | None:
    """Create secret with error handling."""
    try:
        return client.create_secret(secret)
    except Exception as e:
        print(f"Failed to create secret: {e}")
        return None


def safe_retrieve(url: str) -> str | None:
    """Retrieve secret with error handling."""
    try:
        secret_id, key = client.parse_share_url(url)

        if not client.check_exists(secret_id):
            print("Secret has expired or been viewed")
            return None

        result = client.get_secret(secret_id, key)
        return result['secret']

    except Exception as e:
        if "not found" in str(e).lower():
            print("Secret has expired or been viewed")
        elif "decrypt" in str(e).lower():
            print("Invalid encryption key")
        else:
            print(f"Error: {e}")
        return None
```

## Async Version

```python
"""Async version using aiohttp."""

import aiohttp
import asyncio


class AsyncSharePasswordsClient:
    """Async client for SharePasswords API."""

    def __init__(self, api_base: str = API_BASE):
        self.api_base = api_base

    async def create_secret(
        self,
        secret: str,
        expires_in: str = "24h",
        max_views: int | None = 1
    ) -> dict:
        """Create a new encrypted secret asynchronously."""
        key = generate_key()
        encrypted_content = encrypt(secret, key)
        secret_id = generate_secret_id()

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.api_base}/secrets",
                json={
                    "secretId": secret_id,
                    "encryptedContent": encrypted_content,
                    "expiresIn": expires_in,
                    "maxViews": max_views
                }
            ) as response:
                if not response.ok:
                    data = await response.json()
                    raise Exception(data.get('error', {}).get('message'))

                data = await response.json()

        key_base64 = key_to_base64(key)
        share_url = f"https://sharepasswords.com/s/{secret_id}#{key_base64}"

        return {
            "secretId": secret_id,
            "shareUrl": share_url,
            "expiresAt": data.get('data', {}).get('expiresAt')
        }

    async def get_secret(self, secret_id: str, key_base64: str) -> dict:
        """Retrieve and decrypt a secret asynchronously."""
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.api_base}/secrets/{secret_id}"
            ) as response:
                if not response.ok:
                    data = await response.json()
                    raise Exception(data.get('error', {}).get('message'))

                data = await response.json()

        key = base64_to_key(key_base64)
        plaintext = decrypt(data['data']['encryptedContent'], key)

        return {
            "secret": plaintext,
            "remainingViews": data['data'].get('remainingViews')
        }


# Usage
async def main():
    client = AsyncSharePasswordsClient()

    result = await client.create_secret("async-secret-123")
    print(f"URL: {result['shareUrl']}")

    secret_id, key = SharePasswordsClient.parse_share_url(result['shareUrl'])
    decrypted = await client.get_secret(secret_id, key)
    print(f"Secret: {decrypted['secret']}")


if __name__ == "__main__":
    asyncio.run(main())
```
