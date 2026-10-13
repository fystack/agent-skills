# Signing Fystack API requests

Read this when calling the REST API without an official SDK, or when debugging a signature error. The TypeScript and Go SDKs do all of this for you. Source docs: `https://docs.fystack.io/authentication.md` and `https://docs.fystack.io/api-keys.md`.

## Contents

1. The canonical string
2. HMAC secret
3. Ed25519 client key pair (recommended)
4. AWS KMS
5. Complete request examples
6. Debugging signature failures

## 1. The canonical string

Every signed request carries three headers:

```
ACCESS-API-KEY: <api key>
ACCESS-TIMESTAMP: <unix time in seconds, as a string>
ACCESS-SIGN: <signature, see below>
```

The signature is computed over this exact string, fields in this order:

```
method={METHOD}&path={PATH}&timestamp={TIMESTAMP}&body={BODY}
```

- `METHOD`: uppercase HTTP method, e.g. `GET`, `POST`.
- `PATH`: the URL path **including** `/api/v1` and **excluding** the query string, e.g. `/api/v1/wallets/5f0c.../deposit-address` even when you call `...deposit-address?address_type=evm`. (This matches the official SDK, which signs `new URL(endpoint).pathname`.)
- `TIMESTAMP`: Unix time in **seconds**, the same value you send in `ACCESS-TIMESTAMP`.
- `BODY`: the exact JSON string you send as the request body. For `GET` or any request without a body, an empty string (not `{}` and not `null`).

**Serialize the body once.** Build the JSON string, sign it, and send that same string as the body. If your HTTP client serializes the object again (for example `requests.post(json=payload)` in Python), whitespace or key order can differ and the server rejects the signature even though the JSON is logically identical.

## 2. HMAC secret

`ACCESS-SIGN = base64( hex( HMAC-SHA256(key = api_secret as UTF-8 bytes, message = canonical) ) )`

Two encodings, in this order: hex-encode the digest, then base64-encode **the hex text**. Do not base64 the raw digest, and do not hex-decode the secret first; the docs and SDK use the secret string's bytes directly.

The secret is shown once when the key is created. If it is lost, create a new key.

```typescript
import { createHmac } from 'node:crypto'

function hmacSign(secret: string, canonical: string): string {
  const hex = createHmac('sha256', secret).update(canonical, 'utf8').digest('hex')
  return Buffer.from(hex, 'utf8').toString('base64')
}
```

```python
import base64, hashlib, hmac

def hmac_sign(secret: str, canonical: str) -> str:
    hex_digest = hmac.new(secret.encode(), canonical.encode(), hashlib.sha256).hexdigest()
    return base64.b64encode(hex_digest.encode()).decode()
```

```go
func hmacSign(secret, canonical string) string {
	mac := hmac.New(sha256.New, []byte(secret))
	mac.Write([]byte(canonical))
	hexDigest := hex.EncodeToString(mac.Sum(nil))
	return base64.StdEncoding.EncodeToString([]byte(hexDigest))
}
```

## 3. Ed25519 client key pair (recommended)

`ACCESS-SIGN = base64( Ed25519_Sign(private_key, canonical as UTF-8 bytes) )`

One encoding only: base64 of the raw 64-byte signature. There is no hex step here; mixing up the HMAC and Ed25519 encodings is the most common signing bug.

Generate the key pair locally and register only the public key in the dashboard. The private key never leaves your infrastructure and must never ship to a browser.

```bash
openssl genpkey -algorithm ed25519 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem
```

```typescript
import { createPrivateKey, sign } from 'node:crypto'

const privateKey = createPrivateKey(process.env.FYSTACK_PRIVATE_KEY_PEM!)

function ed25519Sign(canonical: string): string {
  return sign(null, Buffer.from(canonical, 'utf8'), privateKey).toString('base64')
}
```

```python
import base64, os
from cryptography.hazmat.primitives.serialization import load_pem_private_key

private_key = load_pem_private_key(os.environ["FYSTACK_PRIVATE_KEY_PEM"].encode(), password=None)

def ed25519_sign(canonical: str) -> str:
    return base64.b64encode(private_key.sign(canonical.encode())).decode()
```

```go
func loadEd25519(pemText string) (ed25519.PrivateKey, error) {
	block, _ := pem.Decode([]byte(pemText))
	if block == nil {
		return nil, errors.New("invalid PEM")
	}
	key, err := x509.ParsePKCS8PrivateKey(block.Bytes)
	if err != nil {
		return nil, err
	}
	priv, ok := key.(ed25519.PrivateKey)
	if !ok {
		return nil, errors.New("not an Ed25519 key")
	}
	return priv, nil
}

func ed25519Sign(priv ed25519.PrivateKey, canonical string) string {
	return base64.StdEncoding.EncodeToString(ed25519.Sign(priv, []byte(canonical)))
}
```

With the Go SDK, use `fystack.NewLocalPrivateKeySigner(pemBytes)` as `Credentials.Signer`. With the TypeScript SDK, pass a signer instead of a secret:

```typescript
import { FystackSDK, Environment, LocalPrivateKeySigner } from '@fystack/sdk'

const sdk = new FystackSDK({
  credentials: {
    apiKey: process.env.FYSTACK_API_KEY!,
    signer: new LocalPrivateKeySigner(process.env.FYSTACK_PRIVATE_KEY_PEM!),
  },
  workspaceId: process.env.FYSTACK_WORKSPACE_ID,
  environment: Environment.Sandbox,
})
```

## 4. AWS KMS

For production, keep the Ed25519 key in KMS so it cannot be exported:

- Key spec `ECC_NIST_EDWARDS25519`, key usage `SIGN_VERIFY`.
- Sign the canonical string with `SigningAlgorithm: 'ED25519_SHA_512'` and `MessageType: 'RAW'` (not `DIGEST`), then base64 the returned signature.
- Register the public key: `aws kms get-public-key` returns base64 DER; wrap it in `-----BEGIN PUBLIC KEY-----` / `-----END PUBLIC KEY-----`.
- In the SDK: `new AwsKmsSigner({ keyId: 'alias/...', clientConfig: { region } })` as the `signer` (needs `@aws-sdk/client-kms`). Use the instance's IAM role; never put AWS access keys in code.

## 5. Complete request examples

```python
import json, os, time, uuid, requests

BASE = "https://api.enterprise-sandbox.fystack.io"
API_KEY = os.environ["FYSTACK_API_KEY"]

def signed_request(method: str, path: str, payload=None, params=None, extra_headers=None):
    body = "" if payload is None else json.dumps(payload, separators=(",", ":"))
    timestamp = str(int(time.time()))
    canonical = f"method={method}&path={path}&timestamp={timestamp}&body={body}"
    headers = {
        "ACCESS-API-KEY": API_KEY,
        "ACCESS-TIMESTAMP": timestamp,
        "ACCESS-SIGN": ed25519_sign(canonical),  # or hmac_sign(secret, canonical)
        "Content-Type": "application/json",
        **(extra_headers or {}),
    }
    # data=body sends exactly the signed string; params are not part of the signature
    return requests.request(method, BASE + path, data=body or None, params=params, headers=headers, timeout=30)

# GET with a query string: the query is not signed
signed_request("GET", f"/api/v1/wallets/{wallet_id}/deposit-address", params={"address_type": "evm"})

# Withdrawal with an idempotency key, generated once and reused on retries
signed_request(
    "POST",
    f"/api/v1/wallets/{wallet_id}/request-withdrawal",
    payload={"asset_id": asset_id, "amount": "0.001", "recipient_address": recipient},
    extra_headers={"X-Idempotency-Key": withdrawal_request_id},
)
```

## 6. Debugging signature failures

Work through these in order; each one has caused real failures:

1. **Encoding**: HMAC is base64 of the hex string; Ed25519 is base64 of raw bytes.
2. **Body bytes**: log the exact body string you signed and the exact bytes sent. They must be identical. `GET` signs `body=` with nothing after it.
3. **Path**: includes `/api/v1`, excludes `?query`, no trailing slash unless the URL has one, no host.
4. **Timestamp**: seconds, not milliseconds; the same value in the header and the canonical string; host clock in sync (NTP).
5. **Method**: uppercase.
6. **Key**: the API key matches the secret or public key you are signing for, the key has not expired, and the caller's IP is on the key's whitelist if one is set.
7. **Environment**: keys belong to one environment. Use sandbox keys with `api.enterprise-sandbox.fystack.io` and production keys with `api.fystack.io`.

When logging for debugging, never log the secret or private key, and remove signature logging once fixed.
