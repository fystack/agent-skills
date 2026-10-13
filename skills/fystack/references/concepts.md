# How Fystack fits together

The product model behind the API: who can do what, and how a withdrawal decides whether it needs humans. Read this for setup and "how do I configure X" questions, and before writing code that depends on approvals or roles. The source pages are under `https://docs.fystack.io/product/` (append `.md`).

## Contents

1. Workspaces
2. Roles
3. Wallets
4. Approval groups
5. How a withdrawal is decided
6. Policies
7. Operations features
8. Account security

## 1. Workspaces (`product/workspace.md`)

A workspace is the top-level, isolated unit. It has its own wallets, members and roles, API keys and history. Nothing is shared between workspaces. One account can belong to several workspaces, with a different role in each. Use separate workspaces for separate clients, business units or environments.

## 2. Roles (`product/user-management.md`)

Roles are enforced in both the API and the dashboard, per workspace:

- **Owner**: the workspace creator. Unrestricted, and only one per workspace. Only the Owner can create approval groups.
- **Admin**: manages wallets, members and settings. Can initiate withdrawals.
- **Signer**: approves or cancels pending transactions.
- **Proposer**: creates withdrawals but cannot approve them.
- **Viewer**: read-only.
- **Guest**: limited access to specific wallets or functions.

Maker-checker means Proposers create transactions and Signers approve them. To keep a team (for example engineering) from approving funds, give them Proposer or Viewer, and leave them out of the approval groups attached to wallets.

## 3. Wallets (`wallets.md`)

- **Hyper** wallets (API type `standard`) are HD-derived per workspace and created instantly. Use them for high-volume, per-user deposit wallets.
- **MPC** wallets split the key into shares across nodes, and creation takes seconds. Use them for hot, treasury and custody wallets.
- **Purpose** (`general`, `user`, `one_time_use`) is mostly organizational. `user` wallets are hidden from the dashboard's wallet list.
- **Assets on hold**: a pending withdrawal puts its funds on hold, so `available_balance` is `balance` minus holds.
- **Network fees** are paid in each chain's native token, so a wallet sending USDC on Base also needs ETH on Base, unless the Gas Station covers it.

## 4. Approval groups (`product/approval-group.md`)

An approval group is a named set of members plus a **threshold** (how many must approve, at most the member count). Groups attach to three things:

- **Wallets**: only the group's members can approve that wallet's withdrawals, and the group's threshold applies.
- **Policies**: changes to the policy need the group's sign-off.
- **Other groups**: creating a group is reviewed by an existing group.

Every workspace starts with a **Default Approval Group** (threshold 1).

Changes are themselves gated, so nobody can quietly widen their own access:

- creating a group (status `Pending` until approved)
- changing its members
- changing which group a wallet uses

A wallet's settings show two related values:

- **Multi-Approval Threshold**: taken from the assigned group. To change it, change the group.
- **Auto-Approval Limit (USD)**: withdrawals below this value skip approval entirely. `0` means every withdrawal needs approval.

All of this is configured in the dashboard. The docs describe no API for creating groups or approving requests.

## 5. How a withdrawal is decided

When a withdrawal is requested:

1. **Policies** on `withdrawal.create` that target the wallet are evaluated. Any matching **DENY** blocks it (`DENIED_BY_POLICY`). Otherwise, a matching **ALLOW** skips approval and the withdrawal goes straight to signing.
2. Otherwise, if the value is under the wallet's **Auto-Approval Limit**, it skips approval.
3. Otherwise it waits in `PENDING_APPROVAL` until enough members of the wallet's approval group approve (the threshold), or someone cancels it.
4. Once approved, it is signed and broadcast: `PENDING` → `EXECUTED` → `SUCCESS`, or `FAILED`, which releases the hold.

Common setups:

- **Tiered approvals.** "Small withdrawals automatic, large ones need two of three approvers" is an approval group with three members and threshold 2, plus a fast path for the small tier. The Auto-Approval Limit is the simplest fast path. A narrow ALLOW rule (amount cap, daily cap, whitelisted destinations) is safer. When its caps are hit, payments fall back to normal approval rather than being blocked, which a DENY cap would do.
- **Thresholds that change with size.** A single wallet has one group threshold. If different sizes need different thresholds, use separate wallets with different groups, or block the largest transfers with a DENY rule. Ask the user what they want for the amounts in between.
- **Hard limits.** "Never more than X per day" or "only to whitelisted addresses" is a DENY rule, such as `wallet.outflow_24h + withdrawal.value_usd > X` or `!withdrawal.is_whitelisted`.

## 6. Policies (`product/policies.md`, `policies.md`)

A policy names an approval group (which governs changes to it), targets some or all wallets, and holds up to 50 rules.

- **Triggers**: each rule has one, either `withdrawal.create` or `web3.contract_call`.
- **Effects**: `ALLOW` skips approval; `DENY` blocks. DENY always wins.
- **Fail-closed**: if evaluation errors or exceeds its 100 ms budget, the action is denied.
- **Only active policies count**: `pending`, `inactive` and `rejected` policies are not evaluated.
- **Conditions** use a CEL-like syntax (`&&`, `||`, `!`, comparisons) over fields:
  - `withdrawal.value_usd`, `withdrawal.asset_symbol`, `withdrawal.network_code`, `withdrawal.is_whitelisted`, `withdrawal.address_age_sec`
  - `wallet.outflow_24h`, `workspace.*` velocity fields
  - `principal.role`, and `context.hour` (UTC)
  - for contract calls: `resource.contract_address`, `resource.method_selector`, `resource.decoded_args`

  `GET /policy/schema` lists every field.
- **Velocity fields**: the outflow and count fields only include withdrawals that have already executed or succeeded.

## 7. Operations features

- **Gas Station** (`product/gas-station.md`): a Gas Tank wallet holding native tokens per chain, so other wallets (including sweeps) always have gas. Its Fee Saving tab can rent TRON energy through TronZap, so TRON transfers pay no TRX fee.
- **Automation and sweeps** (`product/automation.md`): sweep tasks move funds from source wallets to a destination wallet when a USD threshold is reached, checked on a schedule. Typical uses are consolidating user deposits into a hot wallet, or moving hot-wallet excess to cold storage. A task can be attached at wallet creation through the API, or managed in the dashboard. Repeatedly failing sweeps pause automatically.
- **Alerts** (`product/alerts.md`): notifications sent to Telegram channels.
  - **Event alerts**: a deposit, or a withdrawal being requested, approved, cancelled or executed.
  - **Trigger alerts**: a balance dropping below a threshold, a volume limit, or unusual activity.
  - Alerts are for people. Use webhooks for system integration.
- **Address risk screening** (`product/withdrawal.md`): withdrawals in the dashboard show a risk score and warnings about address poisoning. KYT results are included in deposit webhooks.
- **Network management and rescans** (`product/network.md`): shows network health, and lets you re-index missed blocks or transactions.

## 8. Account security (`product/security.md`)

Dashboard accounts support Passkeys (recommended) and TOTP two-factor authentication, and a workspace can require MFA for all members. API access is separate: it uses workspace-scoped API keys, which can be restricted by IP and given an expiry date.
