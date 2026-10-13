# Go SDK quick reference (`github.com/fystack/platform-sdk-go`)

A summary of the official Go SDK as of v0.1.0. The repository README (`https://github.com/fystack/platform-sdk-go`) and the package's Go doc comments are the source of truth, so check them before relying on a signature. The SDK requires Go 1.24 or newer.

## Contents

1. Setup and environments
2. Wallets and deposits
3. Withdrawals
4. Calling endpoints the SDK does not wrap
5. Web3 signing, sweeps and payments
6. Webhooks

## 1. Setup and environments

```bash
go get github.com/fystack/platform-sdk-go
```

```go
import fystack "github.com/fystack/platform-sdk-go"

client, err := fystack.NewClient(fystack.ClientOptions{
	Credentials: fystack.Credentials{
		APIKey:    os.Getenv("FYSTACK_API_KEY"),
		APISecret: os.Getenv("FYSTACK_API_SECRET"), // HMAC; or Signer for Ed25519 / KMS
	},
	WorkspaceID: os.Getenv("FYSTACK_WORKSPACE_ID"),
	Environment: fystack.EnvironmentSandbox,
})
```

- **Credentials**: `Credentials{APIKey, APISecret}` for HMAC, or `Credentials{APIKey, Signer}`.
  - Build the signer with `fystack.NewLocalPrivateKeySigner(pemBytes)` (Ed25519 PKCS8).
  - For AWS KMS, use `awskms.New(ctx, awskms.Options{KeyID: "alias/..."})` from `github.com/fystack/platform-sdk-go/signer/awskms`.
- **Environment**: an empty `Environment` means production, just like the TypeScript SDK. Always set it explicitly.
- **Sandbox host**: in v0.1.0, `fystack.EnvironmentSandbox` resolves to `https://api-dev.fystack.io/api/v1`. That host did not answer when last checked; the working sandbox used by the TypeScript SDK is `api.enterprise-sandbox.fystack.io`. If sandbox calls fail, set `Domain: "api.enterprise-sandbox.fystack.io"`. `Domain` takes precedence over `Environment` and resolves to `https://<domain>/api/v1`. Mention this to the user rather than silently picking one.
- **Other options**: `BaseURL` (tests and gateways) and `HTTPClient` (timeouts, proxies, tracing). Every call takes a `context.Context`.

## 2. Wallets and deposits

- `CreateWallet(ctx, CreateWalletParams{Name, WalletType, WalletPurpose, NetworkIDs?})` waits for an MPC wallet to reach a terminal state. `CreateWalletAsync` returns immediately; then poll `GetWalletCreationStatus(ctx, walletID)` with your own deadline. Prefer the async form when the caller must not lose the `WalletID` on a timeout.
- `WalletType`: `fystack.WalletTypeHyper` (API value `standard`) or `fystack.WalletTypeMPC`.
- `GetWallets(ctx)`, `GetWalletAssets(ctx, walletID)`, `GetWalletDetail(ctx, walletID, addressType)`.
- `GetDepositAddress(ctx, walletID, fystack.AddressTypeEVM | AddressTypeSolana | AddressTypeTron | AddressTypeBitcoin)`. For Bitcoin, use `GetDepositAddressByAssetID(ctx, walletID, bitcoinAssetID)` so the correct mainnet or testnet address is chosen.
- `RescanTransaction(ctx, RescanTransactionParams{...})` re-indexes a missed transaction.

## 3. Withdrawals

```go
resp, err := client.RequestWithdrawalWithOptions(ctx, walletID,
	fystack.RequestWithdrawalParams{
		AssetID:          assetID, // or Asset: "USDC", Network: "BASE_MAINNET"
		Amount:           "25.00", // decimal string
		RecipientAddress: recipient,
		Notes:            "Invoice 1042",
	},
	fystack.RequestOptions{IdempotencyKey: payoutID}, // stable across retries
)
status, err := client.GetWithdrawalStatus(ctx, walletID, resp.Withdrawal.ID)
```

Unlike the TypeScript SDK, the Go SDK sends an idempotency key: use `RequestWithdrawalWithOptions` with a key derived from your own payout ID, and reuse it on every retry. `FeePriority` is available on the params for supported chains.

## 4. Calling endpoints the SDK does not wrap

Balances, assets, networks, policies and workspace stats are not wrapped. Use the signed generic call rather than writing your own signing:

```go
var out struct {
	Data []struct {
		AssetID          string `json:"asset_id"`
		Symbol           string `json:"symbol"`
		NetworkCode      string `json:"network_code"`
		Balance          string `json:"balance"`
		AvailableBalance string `json:"available_balance"`
	} `json:"data"`
}
err := client.Do(ctx, fystack.APIRequest{
	Method: http.MethodGet,
	Path:   "/wallets/" + walletID + "/balances?symbol=USDC&network_code=BASE_MAINNET",
}, &out)
```

`APIRequest` has `Method`, `Path`, `Body` and `Header`. `Path` must start with `/` and is relative to the `/api/v1` base URL. Check `client.go` for how the response envelope is decoded before relying on the shape of `out`. See [api.md](api.md) for routes and fields.

## 5. Web3 signing, sweeps and payments

- **EVM**: `evm.New(client, provider)` from `signer/evm`, then `SetWallet(walletID)`, `SignMessage`, `SignTypedData` and `SendTransaction(ctx, tx, fystack.RequestOptions{IdempotencyKey: ...})`. Transaction quantities are decimal strings.
- **Solana**: `solana.New(client)` from `signer/solana`. Its README notes that current deployments reject Solana raw-transaction signing; use withdrawals for SOL transfers.
- **Lower level**: `RequestSign` / `GetSignStatus` and `SignTransaction` / `GetTransactionStatus` on the client.
- **Sweeps**: `CreateSweepTask(ctx, CreateSweepTaskParams{Name, Strategy, MinTriggerValueUSD, DestinationWalletID, DestinationType, FrequencyInSeconds, WalletIDs})`.
- **Payments**: `payment.New(apiKey, fystack.ClientOptions{...})` from the `payment` package, then `CreateCheckout`.

## 6. Webhooks

```go
valid, err := fystack.VerifyWebhookEd25519(publicKeyHex, event, signatureHex)
```

`event` is the parsed request body (the whole envelope), and `publicKeyHex` comes from `client.GetWebhookPublicKey(ctx, workspaceID)`. The SDK canonicalizes the JSON for you (`fystack.CanonicalJSON`). Prefer the Ed25519 check, which matches the documented scheme, over `VerifyWebhookHMAC`. The handling rules in [webhooks.md](webhooks.md) still apply.
