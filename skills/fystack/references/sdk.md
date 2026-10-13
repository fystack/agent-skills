# `@fystack/sdk` quick reference

Summary of the TypeScript SDK's public surface as of version 0.1.19. The SDK changes faster than this file: when the package is installed, `node_modules/@fystack/sdk/dist/types/index.d.ts` is the source of truth, and you should check it before relying on any signature below.

## Contents

1. Setup
2. Wallets and deposits
3. Withdrawals
4. Web3 signing
5. Checkout
6. Enums
7. Where docs and SDK disagree

## 1. Setup

```bash
npm install @fystack/sdk
```

```typescript
import { FystackSDK, Environment } from '@fystack/sdk'

const sdk = new FystackSDK({
  credentials: {
    apiKey: process.env.FYSTACK_API_KEY!,
    apiSecret: process.env.FYSTACK_API_SECRET, // HMAC, or:
    // signer: new LocalPrivateKeySigner(pem) or new AwsKmsSigner({ keyId, clientConfig: { region } })
  },
  workspaceId: process.env.FYSTACK_WORKSPACE_ID, // needed for workspace-scoped calls
  environment: Environment.Sandbox,              // required in practice: omitting it means production
  debug: false,
})
```

`credentials` is `{ apiKey, apiSecret?, signer?, authToken? }`. Use either `apiSecret` (HMAC) or `signer` (Ed25519). See [authentication.md](authentication.md).

## 2. Wallets and deposits

- `createWallet({ name, walletType, walletPurpose?, sweepTaskParams?, sweepTaskId? }, waitForCompletion = true)` → `{ wallet_id, status }`. The built-in wait polls about 10 times (roughly a minute), then **throws without the wallet ID**; if creation fails it returns `status: "error"` instead of throwing. For MPC wallets, pass `false`, store `wallet_id`, and poll yourself:
- `getWalletCreationStatus(walletId)` → `{ status }` (`pending`, `success`, `error`).

```typescript
const { wallet_id } = await sdk.createWallet({ name, walletType: WalletType.MPC }, false)
const deadline = Date.now() + 3 * 60_000
let status = 'pending'
while (status === 'pending' && Date.now() < deadline) {
  await new Promise((r) => setTimeout(r, 2000))
  ;({ status } = await sdk.getWalletCreationStatus(wallet_id))
}
if (status !== 'success') throw new Error(`Wallet ${wallet_id} creation ${status}`)
```
- `getWallets()` → wallets in the workspace (needs `workspaceId`).
- `getWalletAssets(walletId)` → assets and balances in a wallet.
- `getDepositAddress(walletId, AddressType.Evm | AddressType.Solana | AddressType.Tron)` → `{ address, qr_code, asset_id? }`. `AddressType` has no Bitcoin member in 0.1.19; Bitcoin addresses (`address_type=btc`) are available through the REST endpoint.
- `rescanTransaction({ txHash, networkId })` → asks Fystack to re-index a transaction that was missed.
- `getWebhookPublicKey(workspaceId)` → the Ed25519 key for verifying webhooks.
- `automation.createSweepTask(params)` → create a sweep task to attach to wallets.

## 3. Withdrawals

```typescript
const { auto_approved, approval_bypassed_by_policy, withdrawal } = await sdk.requestWithdrawal(walletId, {
  assetId,                 // or asset + network
  amount: '0.001',         // decimal string
  recipientAddress,
  notes: 'Invoice 1042',
})

const latest = await sdk.getWithdrawalStatus(walletId, withdrawal.id)
// latest.status: WithdrawalStatus, latest.error_reason, latest.withdrawal_approvals, latest.transaction_id
```

The SDK method has no idempotency option. If the integration can retry a withdrawal (job queues, HTTP retries), make the call idempotent in your own system (persist the intent and check for an existing withdrawal before calling again), or call the REST endpoint directly with an `X-Idempotency-Key` header.

## 4. Web3 signing

Signers sign with a Fystack wallet and wait for dashboard approval before returning.

```typescript
import { EtherSigner, SolanaSigner, Environment } from '@fystack/sdk'
import { JsonRpcProvider } from 'ethers'

const evm = new EtherSigner(credentials, Environment.Sandbox, new JsonRpcProvider(rpcUrl))
evm.setWallet(walletId)
await evm.getAddress()
await evm.signMessage('hello')
await evm.signTypedData(domain, types, value)   // EIP-712
await evm.sendTransaction({ to, value })        // signs and broadcasts

const sol = new SolanaSigner(credentials, Environment.Sandbox)
sol.setWallet(walletId)
await sol.signMessage('hello')
await sol.signTransaction(base64SerializedTx)
await sol.signAndSendTransaction(base64SerializedTx)
```

Each method accepts an optional final `options` argument for polling behavior. A call that times out does not cancel the request; check its status before creating another.

## 5. Checkout

`PaymentService` creates hosted payment pages: `createCheckout({...})`, `getCheckout(id)`, `createCheckoutPayment(checkoutId, { pay_asset_id })`, `getCheckoutPayment(id)`. Its parameters use snake_case keys, unlike `FystackSDK`. Read `https://docs.fystack.io/checkout.md` before using it.

## 6. Enums

- `Environment`: `Sandbox` (`"sandbox"`), `Production` (`"production"`), `Local`.
- `WalletType`: `Hyper` (`"standard"`), `MPC` (`"mpc"`).
- `WalletPurpose`: `General`, `User`, `OneTimeUse`, `Gastank`, `Deployment`, `Custody`, `Payment`.
- `AddressType`: `Evm` (`"evm"`), `Solana` (`"sol"`), `Tron` (`"tron"`).
- `WithdrawalStatus`: `PendingApproval`, `Pending`, `Executed`, `Success`, `Failed`, `Cancelled`, `FundingGas`, `DeniedByPolicy` (values are the uppercase API strings, e.g. `"PENDING_APPROVAL"`).
- `DestinationType` (sweeps): `InternalWallet`, `AddressBook`.

## 7. Where docs and SDK disagree

Some doc snippets predate the current SDK. Prefer the installed types:

- `getWithdrawalStatus(walletId, withdrawalId)` takes two arguments and calls `GET /wallets/{walletId}/withdrawals/{withdrawalId}`; the docs show one argument and `GET /withdrawals/{id}`.
- The constructor option is `debug`; some docs show `logger: true`.
- The default `environment` is production; doc snippets always pass one, so the default is easy to miss.
- The sandbox host is `api.enterprise-sandbox.fystack.io`; some older material says `api-sandbox.fystack.io`, which does not resolve.
- The Hyper wallet type is `WalletType.Hyper` (API value `standard`).
- Doc snippets often use `WalletType`, `WalletPurpose` and `AddressType` without importing them; import them from `@fystack/sdk`.
- For webhook verification, follow [webhooks.md](webhooks.md) (Ed25519 with the public key) rather than `APIService.Webhook.verifyEvent`.
