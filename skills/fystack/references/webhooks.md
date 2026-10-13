# Receiving Fystack webhooks

Read this when building or debugging a webhook endpoint. Source doc: `https://docs.fystack.io/webhooks.md` (check it for the current event list and example payloads).

## Contents

1. What arrives
2. Verifying the signature
3. Handling events safely
4. Common mistakes

## 1. What arrives

Webhooks are configured in the Fystack dashboard (choose the URL and events there; the docs do not describe a registration API). Each delivery is a JSON `POST` with these headers:

- `x-webhook-signature`: hex-encoded Ed25519 signature.
- `x-webhook-event`: the event name.

The body is an envelope:

```json
{
  "webhook_id": "<uuid of the webhook configuration>",
  "resource_id": "<uuid of the resource that triggered it>",
  "url": "<your endpoint>",
  "event": "deposit.confirmed",
  "payload": { "...event-specific object..." }
}
```

Events documented today:

- `deposit.pending`, `deposit.confirmed`
- `withdrawal.pending`, `withdrawal.executed`, `withdrawal.confirmed`, `withdrawal.failed`
- Checkout: `payment.received`, `payment.success` (see `checkout.md`)

New events are added over time (for example gas funding for withdrawals), so handle unknown events by acknowledging them with 2xx and ignoring them rather than returning an error.

`resource_id` does not always point at the same kind of object across events, so read IDs from `payload` (for example `payload.id`, `payload.wallet_id`, `payload.tx_hash`) instead of relying on `resource_id`.

## 2. Verifying the signature

Verify before you trust anything in the body:

1. Get the workspace's Ed25519 public key once: `GET /api/v1/workspaces/{workspace_id}/webhook-verification-key` (SDK: `sdk.getWebhookPublicKey(workspaceId)`). It returns `data.public_key` as hex. Store it in configuration.
2. Parse the request body as JSON.
3. Canonicalize the **whole envelope** (not just `payload`): sort object keys recursively and serialize compactly, with no extra whitespace. Fystack follows RFC 8785 (JSON Canonicalization Scheme).
4. Verify the hex signature from `x-webhook-signature` against the UTF-8 bytes of that canonical string with the public key.
5. Reject with HTTP 400 if verification fails.

> The SDK's `APIService.Webhook.verifyEvent` helper implements an older HMAC-based scheme with the API secret, not the documented Ed25519 signature. Verify with the public key as shown below.

### TypeScript (Express)

```typescript
import express from 'express'
import { createPublicKey, verify } from 'node:crypto'

// Raw 32-byte Ed25519 key (hex, from webhook-verification-key) wrapped as SPKI DER.
const publicKey = createPublicKey({
  key: Buffer.from('302a300506032b6570032100' + process.env.FYSTACK_WEBHOOK_PUBLIC_KEY!, 'hex'),
  format: 'der',
  type: 'spki',
})

// RFC 8785 style: keys sorted at every level, compact output. Build the string
// directly; re-creating a sorted object and calling JSON.stringify would put
// integer-like keys ("10", "9") in numeric order instead of sorted order.
function canonicalize(value: unknown): string {
  if (Array.isArray(value)) return `[${value.map(canonicalize).join(',')}]`
  if (value !== null && typeof value === 'object') {
    const obj = value as Record<string, unknown>
    return `{${Object.keys(obj)
      .sort()
      .map((key) => `${JSON.stringify(key)}:${canonicalize(obj[key])}`)
      .join(',')}}`
  }
  return JSON.stringify(value)
}

function isValidFystackWebhook(body: unknown, signatureHex: string | undefined) {
  if (!signatureHex || !/^[0-9a-f]{128}$/i.test(signatureHex)) return false
  const message = Buffer.from(canonicalize(body), 'utf8')
  return verify(null, message, publicKey, Buffer.from(signatureHex, 'hex'))
}

const app = express()

app.post('/webhooks/fystack', express.json(), async (req, res) => {
  const ok = isValidFystackWebhook(req.body, req.header('x-webhook-signature'))
  if (!ok) return res.status(400).json({ error: 'invalid signature' })

  res.status(200).json({ received: true }) // acknowledge fast
  await enqueueFystackEvent(req.body)      // do the real work asynchronously
})
```

### Python

```python
import json
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey
from cryptography.exceptions import InvalidSignature

public_key = Ed25519PublicKey.from_public_bytes(bytes.fromhex(FYSTACK_WEBHOOK_PUBLIC_KEY))

def canonicalize(obj) -> str:
    # Recursively sorted keys, compact separators, UTF-8 kept as-is (no \u escapes).
    # For full RFC 8785 number handling, use the `rfc8785` package instead.
    return json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False)

def is_valid_fystack_webhook(raw_body: bytes, signature_hex: str | None) -> bool:
    if not signature_hex:
        return False
    try:
        envelope = json.loads(raw_body)
        public_key.verify(bytes.fromhex(signature_hex), canonicalize(envelope).encode("utf-8"))
        return True
    except (InvalidSignature, ValueError):
        return False
```

### Other languages

Use a JSON Canonicalization Scheme (RFC 8785) library rather than the language's default serializer, then any Ed25519 implementation. Watch for serializer defaults that change the bytes: HTML escaping of `<`, `>` and `&` (Go's `encoding/json`), escaped forward slashes (PHP `json_encode` without `JSON_UNESCAPED_SLASHES`), non-ASCII escaping (Python `ensure_ascii=True`), and unsorted nested objects.

## 3. Handling events safely

- **Make the effect idempotent, not the delivery.** Deliveries repeat, for example after a timeout. Guard the balance credit with `payload.idempotency_key` under a unique constraint, written in the same transaction as the credit, or with a `credited_at` column that is only set once. Do not record a key as processed when a `deposit.pending` arrives: the docs do not say whether pending and confirmed share a key, and if they do, a "skip seen keys" check would drop the confirmed credit. Do not deduplicate on `resource_id` or `id` alone.
- **Credit deposits on `deposit.confirmed`**, not `deposit.pending`. Pending means seen on-chain but not yet final.
- **Treat webhooks as notifications, not the source of truth for money movement.** For high-value actions, re-read the withdrawal or wallet from the API before acting.
- **Expect out-of-order delivery.** A `withdrawal.confirmed` can arrive before you processed `withdrawal.executed`; make handlers move state forward only (ignore transitions that would go backwards).
- **`withdrawal.failed`**: the payload is the withdrawal with `status: "FAILED"`, an `error_reason`, and `transaction` that may be `null`. Held funds are released, so a new withdrawal can be created, with a new idempotency key.
- **Respond within a few seconds.** If you have a durable queue, enqueue and return 2xx. Otherwise process in the request inside one transaction and return 5xx on failure, so Fystack redelivers rather than the event being lost. Never return 2xx before the event is durably recorded.
- **Amount units are not documented.** The example deposit shows `"amount": "1000"` for a 6-decimal token without saying whether that is whole tokens or base units. Confirm with one known sandbox deposit before crediting real balances.

## 4. Common mistakes

- Verifying only `payload` instead of the whole envelope.
- Verifying a re-serialized body without sorting keys, or with a serializer that escapes characters differently.
- Treating the hex signature or hex public key as base64.
- Mounting a raw-body parser and then canonicalizing the string instead of the parsed object.
- Returning 4xx or 5xx for unknown event types, which causes pointless retries.
