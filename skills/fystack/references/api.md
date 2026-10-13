# Fystack REST API map

Every documented endpoint, grouped by resource, with the fields that matter most. Use it to find the right call quickly, then read the linked doc page (append `.md`) before relying on exact request or response fields. Paths are relative to the base URL:

- Production: `https://api.fystack.io/api/v1`
- Sandbox: `https://api.enterprise-sandbox.fystack.io/api/v1`

All endpoints are signed (see [authentication.md](authentication.md)) except `GET /networks` and `GET /assets`, which are public. In Go, `client.Do` from the official SDK sends signed requests to any of these paths.

## Contents

1. Conventions
2. Wallets
3. Assets and networks
4. Withdrawals
5. Webhooks
6. Web3 signing
7. Policies
8. Workspaces
9. Checkout
10. Dashboard-only features

## 1. Conventions

- **Envelope**: most responses are `{ "success": true, "message": "success", "code": 0, "data": ... }`. Errors are `{ "success": false, "message", "code", "errors"?: [{ "field", "reason" }] }`, with internal codes 2407 to 2422 described in `errors.md`. A few endpoints return the object or array without the envelope, so unwrap defensively.
- **Amounts and balances** are decimal strings (`"0.001"`). Keep them as strings or decimals.
- **Pagination** uses `offset` and `limit`. Limits differ per endpoint (some allow up to 1000, and on balance endpoints `limit=0` returns everything), so check the page.
- **Repeated query parameters** (`?symbol=USDC&symbol=SOL`) mean OR within a filter; different filters combine with AND.
- **Idempotency**: `POST /wallets/:id/request-withdrawal` and `POST /web3/:id/sign` accept an `X-Idempotency-Key` UUID. A repeat within 24 hours returns the original response.
- **IDs** are UUIDs. Network codes look like `ETHEREUM_MAINNET`, `BASE_MAINNET`, `SOL_MAINNET`, `POLYGON_MAINNET`, `BSC_MAINNET`, `TRON_MAINNET`, `BTC_MAINNET` (the field is `internal_code` in `/networks` and `network_code` elsewhere).

## 2. Wallets (`wallets.md`)

- `POST /wallets`: create a wallet.
  - Body: `name`, `wallet_type` (`standard` = Hyper, or `mpc`), optional `wallet_purpose` (`general`, `user`, `one_time_use`), optional `sweep_task_params { min_trigger_value_usd, destination_wallet_id, destination_type: internal_wallet | address_book }` or `sweep_task_id`.
  - Returns `{ wallet_id, status }`. Hyper wallets are `success` immediately; MPC wallets start `pending`.
- `GET /wallets/creation-status/:wallet_id`: returns `pending`, `success` or `error`.
- `GET /workspaces/:workspace_id/wallets`: list wallets. Query: `wallet_type`, `limit`, `offset`.
- `GET /wallets/:wallet_id/deposit-address?address_type=evm|sol|tron|btc`: returns `{ address, qr_code }`. One EVM address serves every EVM network.
- `GET /wallets/:wallet_id/overview`: assets with balances, holds, prices and network details.
- `GET /wallets/:wallet_id/balances`: flat per-asset balances.
  - Filters: `symbol`, `asset_id`, `network_id`, `network_code` (all repeatable), plus `offset`, `limit`.
  - Each row has `balance`, `on_hold`, `available_balance` (balance minus holds), `price_usd`, `value_usd`, `network_code`.
- `GET /wallets/:wallet_id/balance/:asset_id`: total balance of one asset, as a string.

## 3. Assets and networks (`assets.md`, `networks.md`)

- `GET /assets` (public): supported assets. Filters: `network_id`, `search` (name or symbol), `is_native`, `is_whitelisted`, `offset`, `limit`. Use it to resolve an `asset_id` for "USDC on Base": find the network ID first, then filter assets by `network_id` and `search=USDC`.
- `PUT /wallets/:wallet_id/asset/:asset_id`: enable an asset on a wallet. Idempotent, and restores a previously removed asset. The wallet must already have an address type for that asset's network (for example an EVM address for an ERC-20).
- `DELETE /wallets/:wallet_id/asset/:asset_id`: remove an asset. Fails while the balance is non-zero.
- `GET /networks` (public): supported networks. Filter with `search`.
- `POST /networks/rescan-transaction`: body `{ tx_hash, network_id }`. Asks Fystack to re-index a transaction it missed, for example a deposit that never showed up.

## 4. Withdrawals (`withdrawals.md`)

- `POST /wallets/:wallet_id/request-withdrawal`: send funds.
  - Body: `asset_id`, `amount` (decimal string), `recipient_address`.
  - Header: `X-Idempotency-Key` (strongly recommended).
  - Returns the withdrawal: `id`, `status`, `withdrawal_approvals`, and more. The SDK's typed response is `{ auto_approved, approval_bypassed_by_policy?, withdrawal }`, so accept both shapes.
- `GET /withdrawals/:withdrawal_id`: status and details, including `transaction { tx_hash, status, ... }` once broadcast. The TypeScript SDK calls `GET /wallets/:wallet_id/withdrawals/:withdrawal_id` instead; prefer the documented path unless the docs change.
- Statuses: `PENDING_APPROVAL` → `PENDING` → `EXECUTED` → `SUCCESS`. Other outcomes: `FAILED`, `REJECTED`, `CANCELLED`, `DENIED_BY_POLICY`, and `FUNDING_GAS` while gas is being topped up.

## 5. Webhooks (`webhooks.md`)

- `GET /workspaces/:workspace_id/webhook-verification-key`: returns `data.public_key`, the hex Ed25519 key for verifying deliveries. See [webhooks.md](webhooks.md).
- Registering webhook URLs and choosing events happens in the dashboard.

## 6. Web3 signing (`web3-sign.md`)

Every signing request waits for approval in the dashboard (**Approval → DApp Transaction**).

- `POST /web3/:wallet_id/sign`: sign a message or typed data.
  - Body: `method` (`eth_sign`, `eth_signTypedData_v4`, `solana_signMessage`), `message` (empty string for typed data), `chain_id` (EVM chain ID, or `0` for Solana), and `typed_data` (a JSON **string** with `domain`, `types`, `message`).
  - Optional header `X-Idempotency-Key`. Returns `{ transaction_id }`.
- `GET /web3/:wallet_id/sign/:transaction_id`: signing status. Values: `pending_approval`, `pending`, `completed`, `confirmed`, `failed`, `rejected`. The signature is available when confirmed.
- `POST /web3/transaction/:wallet_id/signRaw`: sign, and optionally broadcast, a raw transaction. Marked unsafe in the docs: only sign transactions you built or verified.
  - EVM fields: `from`, `to`, `nonce`, `gasLimit`, `data`, `value`, `chainId`, `maxFeePerGas`, `maxPriorityFeePerGas`, `accessList`.
  - Solana: base64 `data`, `from`, `meta.tx_method: "solana_signTransaction"`.
- `GET /web3/transaction/:wallet_id/:transaction_id`: transaction status, with `hash` and `failed_reason` when available.

## 7. Policies (`policies.md`)

`{ws}` is the workspace ID. Create, update and delete are approval-gated: they return `{ status: "pending_approval", approval_request_id }`, and the change applies after the policy's approval group signs off in the dashboard.

- `GET /policy/schema`: the fields available in conditions for each `trigger_action`.
- `POST /workspaces/{ws}/policies/rules/validate`: compile-check a rule before proposing it.
- `POST /workspaces/{ws}/policies`: create a policy.
  - Body: `name`, `description?`, `approval_group_id` (required), `wallet_ids` (an empty list means all wallets), `rules[]`.
  - Each rule has `trigger_action` (`withdrawal.create` or `web3.contract_call`), `condition`, `effect` (`ALLOW` or `DENY`), `description?` and `is_enabled`.
- `GET /workspaces/{ws}/policies`: list policies. Query: `status`, `limit`, `offset`.
- `GET | PUT | DELETE /workspaces/{ws}/policies/{policy_id}`: read, update or delete a policy.
- `GET /workspaces/{ws}/policies/{policy_id}/history`: past revisions.
- `GET | PUT /workspaces/{ws}/policies/{policy_id}/wallets`: read or replace wallet targeting. **PUT applies immediately, with no approval.**
- `POST /workspaces/{ws}/policies/{policy_id}/rules`: add a rule.
- `PUT | DELETE /workspaces/{ws}/policies/rules/{rule_id}`: update or delete a rule. `trigger_action` cannot change; delete the rule and recreate it instead.

## 8. Workspaces (`workspaces.md`)

- `GET /workspaces/{workspace_id}/stats`: wallet, transaction, volume and fee statistics.
- `GET /workspaces/{workspace_id}/balances?symbol=USDC&symbol=SOL`: balances aggregated across wallets by symbol.

## 9. Checkout (`checkout.md`)

Hosted payment pages. In TypeScript, use `PaymentService`.

- `POST /checkouts`: create a checkout.
  - Body: `price`, `currency`, `supported_assets`, `description`, `success_url`, `cancel_url`, `destination_wallet_id`, `expiry_duration_seconds`, and order and customer references.
- `GET /checkouts/{id}`: read a checkout.
- `POST /checkouts/{id}/payment`: body `{ pay_asset_id }`. Creates a payment for the asset the customer chose.
- `GET /checkouts/payment/{id}`: payment status. Related webhooks: `payment.received`, `payment.success`.

Some Checkout examples in the docs use an older host and path (`api.fystack.com/v1/...`). Use the base URLs above.

## 10. Dashboard-only features

The docs do not describe API endpoints for these, so do not invent them. Point the user to the dashboard (`https://app.fystack.io`) instead:

- Approving, rejecting or cancelling withdrawals, signing requests and policy changes
- Creating approval groups, changing members, thresholds and a wallet's auto-approval limit
- Inviting users and assigning roles
- Registering webhooks
- Managing API keys
- Gas Station and fee saving
- Alerts and channels
- Address book entries
- Sweep task management, apart from attaching one at wallet creation (and `automation.createSweepTask` in the SDK)
