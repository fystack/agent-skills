---
name: fystack
description: Build, debug and explain integrations with Fystack, the self-custody digital asset platform. Covers the REST API in any language (request signing, wallets, deposit addresses, balances, withdrawals, webhooks, Web3 signing, policies, checkout), the official TypeScript (@fystack/sdk) and Go (github.com/fystack/platform-sdk-go) SDKs, and how the product works (workspaces, roles, approval groups, auto-approval limits, Gas Station, sweeps, alerts). Use it whenever a task involves Fystack, including api.fystack.io, ACCESS-SIGN headers, x-webhook-signature, EtherSigner or SolanaSigner, policy rules, or questions like "how do we require 2 of 3 approvals" or "why is this withdrawal stuck", even if the user just says "our custody provider" or "the wallet API" and the project already uses Fystack.
license: MIT
metadata:
  author: fystack
  version: "0.1.0"
---

# Building on Fystack

Fystack is a self-custody platform for digital assets. Through its REST API, its TypeScript and Go SDKs, or the dashboard, teams can:

- create MPC and HD ("Hyper") wallets
- issue deposit addresses and track balances
- send withdrawals through approval groups and programmable policies
- sign Web3 messages and transactions
- receive webhooks for deposits and withdrawals

It supports EVM chains, Solana, Tron and Bitcoin.

Mistakes here move or lock real money. This skill does two things: it gets the details right by working from the current docs, and it keeps humans in control of funds.

## What to load for the task

- **Calling the API** from any language: [references/api.md](references/api.md) maps every endpoint, with key fields and statuses. [references/authentication.md](references/authentication.md) covers request signing.
- **TypeScript projects**: [references/sdk.md](references/sdk.md) covers `@fystack/sdk`, which wraps the same API.
- **Go projects**: [references/sdk-go.md](references/sdk-go.md) covers `github.com/fystack/platform-sdk-go`.
- **Receiving webhooks**: [references/webhooks.md](references/webhooks.md).
- **Product questions** ("how do we set up approvals", "who can approve", "why is this withdrawal stuck") and code that depends on roles or approvals: [references/concepts.md](references/concepts.md) explains workspaces, roles, approval groups, policies and the operations features. Many of these questions are answered with dashboard steps, not code. Say so, rather than inventing an API.

## Work from the docs, not from memory

The Fystack docs are published for agents. Use them as the source of truth and fetch them as you go:

- **Index**: `https://docs.fystack.io/llms.txt` lists every page with a one-line summary. Read it when you are not sure which page covers a task.
- **Any page as Markdown**: append `.md`, for example `https://docs.fystack.io/wallets.md`. If a `.md` URL returns 404, fetch the same URL without `.md`.
- **Everything at once**: `https://docs.fystack.io/llms-full.txt` (about 370 KB). Only use it if you truly need the whole site; individual pages are cheaper.

Read the page for the feature you are about to touch before writing code against it. Field names, status values and endpoints change between releases, and a confident guess that is slightly wrong produces signature failures or stuck withdrawals that are painful to debug.

**For SDK signatures, trust the installed types over doc samples.** Some doc snippets lag behind the SDK (for example, `getWithdrawalStatus` takes `(walletId, withdrawalId)` in current types, and the constructor option is `debug`, not `logger`). When `@fystack/sdk` is installed, read `node_modules/@fystack/sdk/dist/types/index.d.ts` for exact method names, parameters and enums. [references/sdk.md](references/sdk.md) summarizes the SDK surface.

### Which page to read

| Task | Read |
|---|---|
| Product setup: roles, approval groups, auto-approval | `product/user-management.md`, `product/approval-group.md`, then [references/concepts.md](references/concepts.md) |
| Set up credentials, sign requests | `api-keys.md`, `authentication.md`, then [references/authentication.md](references/authentication.md) |
| Create wallets, deposit addresses, balances, sweeps | `wallets.md`, `assets.md` |
| Send funds, withdrawal statuses, idempotency | `withdrawals.md`, `product/withdrawal.md` |
| Receive and verify webhooks | `webhooks.md`, then [references/webhooks.md](references/webhooks.md) |
| Sign messages, typed data, raw transactions | `web3-sign.md` |
| Policy rules (ALLOW / DENY) | `policies.md`, `product/policies.md` |
| Accept payments | `checkout.md` |
| Networks, assets, error codes, pagination | `networks.md`, `assets.md`, `errors.md`, `pagination.md` |

All paths are relative to `https://docs.fystack.io/`.

## Choose the integration path

The REST API is the product; the official SDKs wrap the most common parts of it.

- **TypeScript or JavaScript**: prefer `@fystack/sdk` (`npm install @fystack/sdk`). It signs requests, polls asynchronous operations and wraps Web3 signing. For endpoints the SDK does not wrap, such as balances, assets, policies and workspace stats, call the API directly using the same signing scheme.
- **Go**: prefer `github.com/fystack/platform-sdk-go` (`go get github.com/fystack/platform-sdk-go`). It also supports withdrawal idempotency keys and Ed25519 webhook verification. Its generic `client.Do` makes signed calls to endpoints it does not wrap. Check its sandbox host (see [references/sdk-go.md](references/sdk-go.md)).
- **Any other language** (Python, Java, PHP, Rust, and so on): there is no official SDK, so call the REST API directly. Implement request signing exactly as described in [references/authentication.md](references/authentication.md), and use [references/api.md](references/api.md) for routes and fields. Signing is the most common source of bugs: write one small signing helper, unit-test it, and keep a one-call smoke test (for example listing wallets in sandbox) for the user to run before building further.

## Facts that hold across the API

- **Environments**: production `https://api.fystack.io/api/v1`, sandbox `https://api.enterprise-sandbox.fystack.io/api/v1`. In the SDK, `Environment.Production` or `Environment.Sandbox`. **If the environment is omitted, both SDKs silently use production**, so always set it explicitly, and default to sandbox while building.
- **Credentials** come from the dashboard (`https://app.fystack.io/api-key`) and are scoped to a workspace. Pass `workspaceId` to the SDK; workspace-scoped calls such as listing wallets need it.
- **Responses** are usually `{ success, message, code, data }`; errors are `{ success: false, message, code, errors?: [{ field, reason }] }` with internal codes 2407 to 2422 (see `errors.md`).
- **Amounts are decimal strings** (`"0.001"`), never floats. Keep them as strings or use a decimal library.
- **Pagination**: `offset` and `limit`. Defaults and maximums differ by endpoint, so check the page.
- **Public endpoints**: `GET /networks` and `GET /assets` need no auth. Everything else is signed.

## Core flows and their gotchas

**Wallets.** `walletType` is Hyper (API value `standard`, HD-derived, ready instantly) or MPC (`mpc`, key shares across nodes, takes several seconds). Creation is asynchronous for MPC: poll the creation status before using the wallet, and treat `error` as a failure. The SDK's built-in wait gives up after about a minute and throws without the new wallet's ID, so for MPC prefer `createWallet(options, false)`, keep the returned `wallet_id`, and poll `getWalletCreationStatus` with your own timeout. Get deposit addresses per chain family with `address_type` `evm`, `sol`, `tron` or `btc`.

**Deposits.** Learn about incoming funds from webhooks (`deposit.pending`, `deposit.confirmed`) rather than polling balances. Deliveries can repeat, so make the *credit* idempotent: credit only on `deposit.confirmed`, and guard it with `payload.idempotency_key` under a unique constraint (or a `credited_at` column set once). Do not mark a key as "seen" when a `deposit.pending` arrives; the docs do not say whether pending and confirmed share a key, and if they do, that would silently skip the credit.

**Withdrawals.** `POST /wallets/:wallet_id/request-withdrawal` (SDK `requestWithdrawal`). A retry after a timeout must not send funds twice: over REST, send an `X-Idempotency-Key` UUID generated once per logical withdrawal and reused on every retry (cached for 24 hours). The Go SDK takes the key through `RequestWithdrawalWithOptions(..., fystack.RequestOptions{IdempotencyKey: ...})`. The TypeScript SDK's `requestWithdrawal` has no idempotency option, so when using it, persist the withdrawal intent first and check for an existing withdrawal before calling again. Typical lifecycle: `PENDING_APPROVAL` → `PENDING` → `EXECUTED` → `SUCCESS`, with `FAILED`, `REJECTED`, `CANCELLED` and `DENIED_BY_POLICY` as other outcomes. Funds are held while a withdrawal is pending, and network fees are paid in the chain's native token, so the wallet needs gas.

**Approvals.** Whether a withdrawal waits for humans depends on the wallet's approval group, its auto-approval limit and any matching policy. Approving, rejecting and creating approval groups happen in the dashboard; the docs do not describe an approval API, and code should not try to approve its own withdrawals. Design for `PENDING_APPROVAL` lasting minutes or hours.

**Webhooks.** Registered in the dashboard. Verify every request's `x-webhook-signature` (Ed25519 over canonical JSON of the whole body) before trusting it. Respond within a few seconds: either enqueue to a durable queue and return 2xx, or do the database work in the request inside one transaction and return 5xx on failure so Fystack redelivers. Verification has exact canonicalization rules; follow [references/webhooks.md](references/webhooks.md).

**Web3 signing.** `EtherSigner` (ethers-compatible) and `SolanaSigner` sign messages, EIP-712 typed data and transactions with a Fystack wallet. Every signing request goes through approval in the dashboard, so calls block until approved or time out. If a call times out, check the existing request's status before submitting another. Raw transaction signing (`signRaw`) is marked unsafe: only sign payloads you built or verified.

**Policies.** Rules on `withdrawal.create` or `web3.contract_call` with effect `ALLOW` or `DENY`. Any matching DENY blocks; otherwise a matching ALLOW **skips human approval entirely**; otherwise the normal approval flow runs. Evaluation is fail-closed. Creating or changing a policy is itself approval-gated, but replacing a policy's wallet targeting (`PUT .../policies/{id}/wallets`) applies immediately, so treat that endpoint as sensitive. When an ALLOW rule is warranted:

- List existing policies first, since a DENY elsewhere wins and another ALLOW may already overlap.
- Scope it to explicit `wallet_ids` (an empty list means every wallet).
- Cap per-withdrawal value and include the current withdrawal in rolling caps (`wallet.outflow_24h + withdrawal.value_usd <= cap`).
- Require `withdrawal.value_usd > 0` so unpriced assets cannot pass as $0, and restrict assets and networks.
- Prefer whitelisted destinations with a minimum address age (`withdrawal.address_age_sec`).
- Ask the user for every limit rather than inventing defaults.
- Check the condition with `POST /workspaces/{ws}/policies/rules/validate` and the field list with `GET /policy/schema` before proposing it.

## Keep humans in control of funds

These guardrails exist because an agent can write and run code quickly, and a mistake with custody credentials is often irreversible.

- **Build against sandbox.** Use sandbox credentials and `Environment.Sandbox` unless the user explicitly asks for production. Before running anything that would move funds or change policies in production, stop and confirm with the user, stating the wallet, asset, amount and destination.
- **Keep secrets out of code, logs and chat.** Read API keys, secrets and private keys from environment variables or a secret manager. Never hard-code them, print them, commit them, or paste them into a prompt. Prefer the Ed25519 client key pair (with KMS in production) over the HMAC secret.
- **Do not route around approvals.** Do not write code that approves its own withdrawals, raises auto-approval limits, or adds broad `ALLOW` rules to make a flow "just work". If a task seems to need that, explain the trade-off and let the user decide. Keep `ALLOW` conditions narrow (specific wallets, assets, destinations, amounts).
- **Never guess an address.** Recipient addresses, wallet IDs and asset IDs must come from the user, configuration or an API response, never be invented or "example" values in code that can run against a real environment.

## When the docs do not answer

Some behavior is not documented, including the request timestamp tolerance, rate limits, webhook retry schedules, whether a deposit webhook's `amount` is in whole tokens or base units, and APIs for registering webhooks or approving requests. When you hit one of these, say so plainly, choose the conservative option (for example, sign the path exactly as sent and test it in sandbox), and suggest the user confirm with Fystack support. Do not invent endpoints or fields.
