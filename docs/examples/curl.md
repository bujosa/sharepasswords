# cURL Examples

Command-line examples using cURL for the SharePasswords API.

> **Note:** cURL can only interact with the API. Encryption/decryption must be done separately using a tool like OpenSSL or a programming language.

## API Endpoints

### Create a Secret

```bash
curl -X POST https://api.sharepasswords.com/v1/secrets \
  -H "Content-Type: application/json" \
  -d '{
    "secretId": "abc12345",
    "encryptedContent": "YOUR_BASE64_ENCRYPTED_CONTENT",
    "expiresIn": "24h",
    "maxViews": 1
  }'
```

**Response:**

```json
{
  "status": "ok",
  "data": {
    "secretId": "abc12345",
    "expiresAt": "2026-01-17T12:00:00.000Z"
  }
}
```

### Get a Secret

```bash
curl https://api.sharepasswords.com/v1/secrets/abc12345
```

**Response:**

```json
{
  "status": "ok",
  "data": {
    "encryptedContent": "BASE64_ENCRYPTED_CONTENT",
    "remainingViews": 0
  }
}
```

### Check if Secret Exists

```bash
curl https://api.sharepasswords.com/v1/secrets/abc12345/exists
```

**Response:**

```json
{
  "status": "ok",
  "data": {
    "exists": true
  }
}
```

### Health Check

```bash
curl -X GET https://api.sharepasswords.com/v1/health
```

**Response:**

```json
{
  "status": "ok"
}
```

## Full Workflow with OpenSSL

Here's a complete example using OpenSSL for encryption:

### Step 1: Generate Encryption Key

```bash
# Generate a 256-bit (32 byte) random key
KEY=$(openssl rand -base64 32)
echo "Key: $KEY"
```

### Step 2: Encrypt Your Secret

```bash
# Your secret
SECRET="my-super-secret-password"

# Generate a random 12-byte IV
IV=$(openssl rand 12)
IV_BASE64=$(echo -n "$IV" | base64)

# Encrypt with AES-256-GCM
# Note: OpenSSL's GCM support is limited in CLI, this is a simplified example
ENCRYPTED=$(echo -n "$SECRET" | openssl enc -aes-256-cbc -base64 -K $(echo -n "$KEY" | xxd -p) -iv $(echo -n "$IV" | xxd -p))

echo "Encrypted: $ENCRYPTED"
```

> **Note:** For proper AES-256-GCM encryption, use a programming language. OpenSSL CLI has limited GCM support.

### Step 3: Create the Secret

```bash
SECRET_ID=$(openssl rand -hex 4)

curl -X POST https://api.sharepasswords.com/v1/secrets \
  -H "Content-Type: application/json" \
  -d "{
    \"secretId\": \"$SECRET_ID\",
    \"encryptedContent\": \"$ENCRYPTED\",
    \"expiresIn\": \"24h\",
    \"maxViews\": 1
  }"

echo "Share URL: https://sharepasswords.com/s/$SECRET_ID#$KEY"
```

### Step 4: Retrieve and Decrypt

```bash
# Fetch the encrypted content
RESPONSE=$(curl -s https://api.sharepasswords.com/v1/secrets/$SECRET_ID)
ENCRYPTED_CONTENT=$(echo $RESPONSE | jq -r '.data.encryptedContent')

# Decrypt (using the same key)
echo "$ENCRYPTED_CONTENT" | openssl enc -aes-256-cbc -d -base64 -K $(echo -n "$KEY" | xxd -p) -iv $(echo -n "$IV" | xxd -p)
```

## Bash Script

Here's a complete bash script wrapper:

```bash
#!/bin/bash

# SharePasswords CLI wrapper
# Usage:
#   ./sharepasswords.sh create "my secret" [expires_in] [max_views]
#   ./sharepasswords.sh get <secret_id>
#   ./sharepasswords.sh exists <secret_id>

API_BASE="https://api.sharepasswords.com/v1"

case "$1" in
  create)
    SECRET="$2"
    EXPIRES_IN="${3:-24h}"
    MAX_VIEWS="${4:-1}"
    SECRET_ID=$(openssl rand -hex 4)

    # Note: This is simplified - use proper AES-GCM in production
    ENCRYPTED=$(echo -n "$SECRET" | base64)

    RESPONSE=$(curl -s -X POST "$API_BASE/secrets" \
      -H "Content-Type: application/json" \
      -d "{
        \"secretId\": \"$SECRET_ID\",
        \"encryptedContent\": \"$ENCRYPTED\",
        \"expiresIn\": \"$EXPIRES_IN\",
        \"maxViews\": $MAX_VIEWS
      }")

    if echo "$RESPONSE" | jq -e '.success' > /dev/null; then
      echo "Secret created successfully!"
      echo "URL: https://sharepasswords.com/s/$SECRET_ID"
      echo "Expires: $(echo $RESPONSE | jq -r '.data.expiresAt')"
    else
      echo "Error: $(echo $RESPONSE | jq -r '.error.message')"
    fi
    ;;

  get)
    SECRET_ID="$2"

    RESPONSE=$(curl -s "$API_BASE/secrets/$SECRET_ID")

    if echo "$RESPONSE" | jq -e '.success' > /dev/null; then
      ENCRYPTED=$(echo "$RESPONSE" | jq -r '.data.encryptedContent')
      REMAINING=$(echo "$RESPONSE" | jq -r '.data.remainingViews')

      # Note: This is simplified - use proper AES-GCM in production
      SECRET=$(echo "$ENCRYPTED" | base64 -d)

      echo "Secret: $SECRET"
      echo "Remaining views: $REMAINING"
    else
      echo "Error: $(echo $RESPONSE | jq -r '.error.message')"
    fi
    ;;

  exists)
    SECRET_ID="$2"

    RESPONSE=$(curl -s "$API_BASE/secrets/$SECRET_ID/exists")
    EXISTS=$(echo "$RESPONSE" | jq -r '.data.exists')

    if [ "$EXISTS" = "true" ]; then
      echo "Secret exists"
    else
      echo "Secret not found or expired"
    fi
    ;;

  *)
    echo "Usage:"
    echo "  $0 create <secret> [expires_in] [max_views]"
    echo "  $0 get <secret_id>"
    echo "  $0 exists <secret_id>"
    echo ""
    echo "Options:"
    echo "  expires_in: 1h, 24h, 7d, 30d (default: 24h)"
    echo "  max_views: number or null (default: 1)"
    ;;
esac
```

## Error Handling Examples

### Handle 404 (Not Found)

```bash
SECRET_ID="nonexistent"

RESPONSE=$(curl -s -w "\n%{http_code}" "$API_BASE/secrets/$SECRET_ID")
HTTP_CODE=$(echo "$RESPONSE" | tail -n 1)
BODY=$(echo "$RESPONSE" | sed '$d')

if [ "$HTTP_CODE" = "404" ]; then
  echo "Secret not found or expired"
elif [ "$HTTP_CODE" = "200" ]; then
  echo "Secret found: $BODY"
else
  echo "Unexpected error: $HTTP_CODE"
fi
```

### Handle Rate Limiting

```bash
RESPONSE=$(curl -s -w "\n%{http_code}" "$API_BASE/secrets")
HTTP_CODE=$(echo "$RESPONSE" | tail -n 1)

if [ "$HTTP_CODE" = "429" ]; then
  echo "Rate limited. Please wait before retrying."
  exit 1
fi
```

## Pretty Print Responses

```bash
# Using jq for formatted output
curl -s "$API_BASE/secrets/$SECRET_ID" | jq .

# Colored output
curl -s "$API_BASE/secrets/$SECRET_ID" | jq -C .
```

## Testing with Verbose Output

```bash
# See full request/response headers
curl -v -X POST "$API_BASE/secrets" \
  -H "Content-Type: application/json" \
  -d '{
    "secretId": "test123",
    "encryptedContent": "dGVzdA==",
    "expiresIn": "1h",
    "maxViews": 1
  }'
```

## Important Security Note

The cURL examples above are simplified for demonstration. In production:

1. **Use proper AES-256-GCM encryption** - The base64 encoding shown is NOT encryption
2. **Use a client library** - JavaScript or Python examples provide proper encryption
3. **Never log keys** - Avoid echoing keys to console in production scripts
4. **Use HTTPS** - Always use the HTTPS endpoint
