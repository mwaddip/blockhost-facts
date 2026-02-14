# Engine Interface Specification

> Authoritative reference for the blockchain adapter boundary.
> The engine is the only component that talks to the chain bidirectionally.
> Any second engine implementation (e.g., `blockhost-engine-opnet`) must satisfy
> this contract to be a drop-in replacement.
>
> Derived from the working EVM implementation (`blockhost-engine`).
> See also: `COMMON_INTERFACE.md` (shared library API), `ADMIN_INTERFACE.md`
> (admin panel — a consumer of engine CLIs).

---

## 1. CLI Commands

All commands are installed to `/usr/bin/` by the engine `.deb`. Consumers shell out
to these as subprocesses. Exit 0 = success, non-zero = failure (stderr has error details).

---

### `is` (identity predicate) — PLANNED

**Installed:** `/usr/bin/is`

Standalone binary. Answers yes/no identity questions via exit code. No env vars, no addressbook — pure cryptographic and on-chain queries. Arguments are order-independent; types are unambiguous (signatures are long hex blobs, addresses are `0x` + 40 hex, NFT IDs are integers, `contract` is a keyword).

#### `is <signature> <wallet>`

Verify that a wallet produced a given signature. Replaces `cast wallet verify`.

**Exit:** 0 = signature matches wallet, 1 = does not match.
**Consumers:** `admin/auth.py` (replaces `cast wallet verify --address`)

#### `is <wallet> <nft_id>`

Check whether a wallet owns a specific NFT. Queries `ownerOf(tokenId)` on the AccessCredentialNFT contract, compares with provided wallet.

**Config:** `web3-defaults.yaml` (nft_contract, rpc_url)
**Exit:** 0 = wallet owns the NFT, 1 = does not.
**Consumers:** Installer (verify admin NFT ownership after setup with pre-deployed contracts)

#### `is contract <address>`

Check whether a smart contract exists at the given address. Replaces `cast call totalSupply()` as a contract liveness check.

**Config:** `web3-defaults.yaml` (rpc_url)
**Exit:** 0 = contract exists, 1 = no contract at address.
**Consumers:** `installer/web/validate_system.py` (replaces `cast call` health checks)

---

### `blockhost-deploy-contracts` — PLANNED

**Installed:** `/usr/bin/blockhost-deploy-contracts` (bash script)

```
blockhost-deploy-contracts          # deploy both NFT and subscription contracts
blockhost-deploy-contracts nft      # deploy NFT contract only
blockhost-deploy-contracts pos      # deploy subscription contract only
```

No flags — positional argument, optional. Absent means both. When deploying both, NFT is deployed first and its address is passed to the subscription contract constructor automatically.

Reads deployer key and RPC from existing config (`/etc/blockhost/deployer.key`, `web3-defaults.yaml`). Replaces all `cast send --create` + `cast abi-encode` calls in `finalize.py`.

**stdout:** Contract address(es), one per line. Installer captures and writes to config files.
**Exit:** 0/1.
**Consumers:** Engine wizard finalization step (replaces `_deploy_contract_with_forge` in installer)

---

### `bw` (blockwallet)

**Installed:** `/usr/bin/bw` (wrapper → `/usr/share/blockhost/bw.js`)

**Environment:** `RPC_URL`, `BLOCKHOST_CONTRACT` (from `/opt/blockhost/.env`). Exception: `bw who` needs neither — reads config from YAML files directly.

**Addressbook:** `/etc/blockhost/addressbook.json` — role-to-wallet mapping. Roles with `keyfile` can sign transactions.

#### `bw send`

```
bw send <amount> <token> <from> <to>
```

| Arg | Required | Description |
|-----|----------|-------------|
| `amount` | yes | Numeric token amount |
| `token` | yes | `eth`, `stable`, or `0x` token address |
| `from` | yes | Addressbook role or `0x` address (must have keyfile) |
| `to` | yes | Addressbook role or `0x` address |

**stdout:** Transaction hash on success.
**Exit:** 0/1.
**Consumers:** `admin/system.py` (`_run_bw(["send", ...])`), fund-manager (internal import of `executeSend()`)

#### `bw balance`

```
bw balance <role> [token]
```

| Arg | Required | Description |
|-----|----------|-------------|
| `role` | yes | Addressbook role, `0x` address, or contract address (read-only — no keyfile needed) |
| `token` | no | `eth`, `stable`, or `0x` address. If omitted, shows all active payment method tokens |

Works for both ERC20 tokens and NFT contracts — both implement `balanceOf()`. For NFTs, returns integer count (no decimals). **PLANNED:** extend to accept contract addresses directly (not just addressbook roles) for NFT balance checks.

**stdout:** Balance information (human-readable).
**Exit:** 0/1.
**Consumers:** `admin/system.py` (`_run_bw(["balance", role])`), installer validation (NFT balance check, replaces `cast call balanceOf`)

#### `bw split`

```
bw split <amount> <token> <ratios> <from> <to1> <to2> ...
```

| Arg | Required | Description |
|-----|----------|-------------|
| `amount` | yes | Total amount to split |
| `token` | yes | `eth`, `stable`, or `0x` address |
| `ratios` | yes | Comma-separated numbers (e.g., `1,2,1`) |
| `from` | yes | Signer (must have keyfile) |
| `to1 to2 ...` | yes | Recipient addresses/roles (count must match ratio count) |

**stdout:** Transaction hashes.
**Exit:** 0/1.
**Consumers:** fund-manager (internal)

#### `bw withdraw`

```
bw withdraw [token] <to>
```

| Arg | Required | Description |
|-----|----------|-------------|
| `token` | no | Token address to withdraw (defaults to primary stablecoin) |
| `to` | yes | Recipient address or role |

Calls `contract.withdrawFunds(token, to)` — moves accumulated tokens from subscription contract to recipient.

**stdout:** Transaction hash.
**Exit:** 0/1.
**Consumers:** `admin/system.py` (`_run_bw(["withdraw", ...])`), fund-manager (internal import of `executeWithdraw()`)

#### `bw swap`

```
bw swap <amount> <from-token> eth <wallet>
```

| Arg | Required | Description |
|-----|----------|-------------|
| `amount` | yes | Numeric amount of source token |
| `from-token` | yes | `stable` or `0x` address |
| `eth` | yes | Literal string `eth` (destination — only ETH supported) |
| `wallet` | yes | Signer role or `0x` address (must have keyfile) |

Uses Uniswap V2 `swapExactTokensForETH()` with 1% slippage buffer.

**stdout:** Transaction hash.
**Exit:** 0/1.
**Consumers:** fund-manager gas check (internal import of `executeSwap()`)

#### `bw who`

```
bw who <identifier>
```

| Arg | Required | Description |
|-----|----------|-------------|
| `identifier` | yes | Numeric NFT token ID, or `admin` (resolves `admin.credential_nft_id` from `blockhost.yaml`) |

Queries `ownerOf(tokenId)` on the AccessCredentialNFT contract. Config sourced from `web3-defaults.yaml` (`blockchain.nft_contract`, `blockchain.rpc_url`). **No env vars or addressbook required.**

**stdout:** Owner address (`0x...`).
**Exit:** 0 if found, 1 if not found or config missing.
**Consumers:** `admin/auth.py` (`["bw", "who", "admin"]` — resolves admin NFT holder for auth)

#### `bw config stable` — PLANNED

```
bw config stable [address]
```

| Arg | Required | Description |
|-----|----------|-------------|
| `address` | no | Stablecoin token address to set as primary |

No argument: show current primary stablecoin address. With argument: call `setPrimaryStablecoin(address)` on the subscription contract. Replaces `cast send ... setPrimaryStablecoin` in `finalize.py`.

**stdout:** Current or newly set stablecoin address.
**Exit:** 0/1.
**Consumers:** Engine wizard finalization step (replaces `_create_default_plan` stablecoin setup)

#### `bw plan create` — PLANNED

```
bw plan create <name> <price>
```

| Arg | Required | Description |
|-----|----------|-------------|
| `name` | yes | Plan name (string) |
| `price` | yes | Price in USD cents per day (integer) |

Calls `createPlan(name, pricePerDayUsdCents)` on the subscription contract. Replaces `cast send ... createPlan` in `finalize.py`.

**stdout:** Created plan ID.
**Exit:** 0/1.
**Consumers:** Engine wizard finalization step (replaces `_create_default_plan` plan creation)

#### `bw set encrypt` — PLANNED

```
bw set encrypt <nft_id> <userEncrypted>
```

| Arg | Required | Description |
|-----|----------|-------------|
| `nft_id` | yes | NFT token ID (integer) |
| `userEncrypted` | yes | Hex-encoded encrypted data |

Calls `updateUserEncrypted(tokenId, bytes)` on the NFT contract. Replaces `cast send ... updateUserEncrypted` in `finalize.py`.

**stdout:** Transaction hash.
**Exit:** 0/1.
**Consumers:** Installer finalization (`_finalize_mint_nft` — update admin NFT metadata)

#### `bw --debug --cleanup`

```
bw --debug --cleanup <address>
```

Debug utility. Sweeps all ETH from signing wallets (server, hot, dev, broker) to the target address. Requires both `--debug` and `--cleanup` flags as safety guards. Skips wallets that are the target or have insufficient balance for gas.

**Consumers:** Manual/testing only.

#### Token Shortcuts

| Shortcut | Resolves to |
|----------|-------------|
| `eth` | Native ETH (not an ERC20) |
| `stable` | Contract's primary stablecoin (payment method ID 1) |
| `0x...` | Literal token address |

#### Addressbook Roles

| Role | Purpose | Has keyfile |
|------|---------|-------------|
| `admin` | Contract owner, receives fund remainder | No (external wallet) |
| `server` | Primary signer, gas payer | Yes (`deployer.key`) |
| `hot` | Intermediate distribution wallet | Yes (`hot.key`, auto-generated) |
| `dev` | Revenue share recipient | No (optional) |
| `broker` | Revenue share recipient | No (optional) |

Custom roles can be added via `ab add` and addressed by name.

---

### `ab` (addressbook)

**Installed:** `/usr/bin/ab` (wrapper → `/usr/share/blockhost/ab.js`)

**Environment:** `RPC_URL`, `BLOCKHOST_CONTRACT` (loaded but only used for wallet generation via root agent).

**File:** `/etc/blockhost/addressbook.json`

#### `ab add`

```
ab add <name> <0xaddress>
```

Add a new wallet entry. Exits 1 if name is immutable or address invalid.

**Consumers:** `admin/system.py` (`_run_ab(["add", name, address])`)

#### `ab del`

```
ab del <name>
```

Delete an addressbook entry. Does NOT delete the keyfile (if any). Exits 1 if immutable or not found.

**Consumers:** `admin/system.py` (`_run_ab(["del", name])`)

#### `ab up`

```
ab up <name> <0xaddress>
```

Update address for existing entry. Preserves `keyfile` field if present.

**Consumers:** None currently (available for admin panel future use).

#### `ab new`

```
ab new <name>
```

Generate new secp256k1 keypair via `pam_web3_tool generate-keypair`. Saves private key to `/etc/blockhost/<name>.key` (chmod 600), adds entry to addressbook with `keyfile` field.

**Consumers:** `admin/system.py` (`_run_ab(["new", name])`)

#### `ab list`

```
ab list
```

**stdout:** JSON — full addressbook contents (`{role: {address, keyfile?}, ...}`).
**Exit:** Always 0.

**Consumers:** None directly (addressbook read by file I/O in most consumers).

#### `ab --init` — PLANNED

```
ab --init
```

Bootstrap the addressbook with initial required entries. Sets up any wallets that must exist before the system is operational. Replaces the wallet-generation portion of `blockhost-init`.

The deployer/server wallet is generated interactively through the wizard UI (which calls `ab new server` behind the scenes when the user clicks "generate"). `ab --init` handles non-interactive bootstrap entries only.

**Consumers:** Engine wizard finalization or first-boot

#### Immutable Roles

`server`, `admin`, `hot`, `dev`, `broker` — cannot be added, deleted, updated, or generated via `ab` (except `ab --init` for bootstrap). These are managed by the installer finalization and fund-manager.

#### Addressbook JSON Schema

```json
{
  "role-name": {
    "address": "0x...",
    "keyfile": "/etc/blockhost/role-name.key"
  }
}
```

`address` is required. `keyfile` is optional — present only for signing wallets.

---

### `blockhost-mint-nft`

**Installed:** `/usr/bin/blockhost-mint-nft` (Python script)
**Python module:** `/usr/lib/python3/dist-packages/blockhost/mint_nft.py` (importable as `from blockhost.mint_nft import mint_nft`)

```
blockhost-mint-nft \
  --owner-wallet <0x...> \
  --machine-id <vm-name> \
  [--user-encrypted <hex>] \
  [--public-secret <string>] \
  [--dry-run]
```

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| `--owner-wallet` | yes | — | Ethereum address to receive NFT |
| `--machine-id` | yes | — | VM name (used in NFT description) |
| `--user-encrypted` | no | `0x` | Hex-encoded encrypted connection details |
| `--public-secret` | no | `""` | Message user signed (`libpam-web3:<address>:<nonce>`) |
| `--dry-run` | no | false | Print cast command without executing |

**Config:** `load_web3_config()` → `deployer.private_key_file`, `blockchain.nft_contract`, `blockchain.rpc_url`

**Implementation:** Shells out to Foundry `cast send` with the NFT contract's `mint()` function.

**Python function signature:**
```python
def mint_nft(
    owner_wallet: str,
    machine_id: str,
    user_encrypted: str = "0x",
    public_secret: str = "",
    config: dict = None,       # Override for load_web3_config()
    dry_run: bool = False,
) -> dict:
    """Returns {"success": True, "tx_hash": "0x..."} or {"success": False, "error": "..."}"""
```

**Consumers:**
- Monitor event handler (CLI: `blockhost-mint-nft --owner-wallet ... --machine-id ...`)
- `installer/web/finalize.py` (Python import: `from blockhost.mint_nft import mint_nft`)

**Exit:** 0/1.

---

### `blockhost-init` — DEPRECATION PLANNED

**Installed:** `/usr/bin/blockhost-init` (bash script)

> **Planned:** Wallet generation responsibilities move to `ab new server` (called by wizard UI)
> and `ab --init` (non-interactive bootstrap). Server ECIES key generation (not a wallet —
> used for userEncrypted decryption and admin command decryption) remains as a distinct
> operation, either retained in a slimmed-down `blockhost-init` or moved to engine wizard
> finalization. Config file initialization (`blockhost.yaml`, `vms.json`) moves to engine
> wizard finalization steps.

```
sudo blockhost-init \
  [--public-secret <message>] \
  [--deployer-key <0x...>] \
  [--deployer-key-file <path>] \
  [--help]
```

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| `--public-secret` | no | `blockhost-access` | Static message users sign for auth |
| `--deployer-key` | no | — | Use existing private key (hex, strips `0x`) |
| `--deployer-key-file` | no | — | Read key from file |

**Must run as root.** One-time server initialization:

1. Generate server secp256k1 keypair → `/etc/blockhost/server.key` (chmod 600)
2. Generate or import deployer keypair → `/etc/blockhost/deployer.key` (chmod 600)
3. Derive deployer address via `cast wallet address`
4. Write `/etc/blockhost/blockhost.yaml` (public_secret, server_public_key, deployer_address, contract_address="")
5. Initialize `/var/lib/blockhost/vms.json` (empty database)

**Prerequisites:** `pam_web3_tool`, `cast` (Foundry), `/etc/blockhost/`, `/var/lib/blockhost/`

**stdout:** Server public key, deployer address, public secret.
**Exit:** 0/1.
**Consumers:** `scripts/first-boot.sh`

---

### `blockhost-generate-signup`

**Installed:** `/usr/bin/blockhost-generate-signup` (Python script)

```
blockhost-generate-signup \
  [--config <path>] \
  [--web3-config <path>] \
  [--output <path>] \
  [--template <path>] \
  [--serve <port>]
```

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| `--config` | no | `/etc/blockhost/blockhost.yaml` | Server config |
| `--web3-config` | no | `/etc/blockhost/web3-defaults.yaml` | Web3 config |
| `--output` | no | `signup.html` | Output file path |
| `--template` | no | auto-detected | HTML template path |
| `--serve` | no | — | Start HTTP server on port (IPv4+IPv6 dual-stack) |

**Template search order:**
1. `--template` argument
2. `./scripts/signup-template.html` (development)
3. `/usr/share/blockhost/signup-template.html` (installed)

**Template placeholders:**

| Placeholder | Source |
|-------------|--------|
| `{{SERVER_PUBLIC_KEY}}` | `blockhost.yaml` → `server_public_key` |
| `{{PUBLIC_SECRET}}` | `blockhost.yaml` → `public_secret` |
| `{{CHAIN_ID}}` | `web3-defaults.yaml` → `blockchain.chain_id` |
| `{{RPC_URL}}` | `web3-defaults.yaml` → `blockchain.rpc_url` |
| `{{NFT_CONTRACT}}` | `web3-defaults.yaml` → `blockchain.nft_contract` |
| `{{SUBSCRIPTION_CONTRACT}}` | `blockhost.yaml` → `contract_address` |
| `{{USDC_ADDRESS}}` | Primary stablecoin address |
| `{{PAGE_TITLE}}` | Default: `"Blockhost - Get Your Server"` |
| `{{PRIMARY_COLOR}}` | Default: `"#6366f1"` |

**stdout:** Generated HTML file path.
**Exit:** 0/1.
**Consumers:** `installer/web/finalize.py` (`["blockhost-generate-signup", "--output", path]`)

---

## 2. Smart Contract Interface

The subscription contract manages plans, subscriptions, and payment methods. Any chain adapter must implement equivalent semantics.

### Abstract Functions

#### Owner-only (contract deployer)

| Function | Purpose |
|----------|---------|
| `createPlan(name, pricePerDayUsdCents)` | Create a subscription tier |
| `updatePlan(planId, name, pricePerDayUsdCents, active)` | Modify tier |
| `cancelSubscription(subscriptionId)` | Immediately expire a subscription |
| `setPrimaryStablecoin(tokenAddress)` | Set direct-USD payment token (method ID 1) |
| `addPaymentMethod(tokenAddress, pairAddress, stablecoinAddress)` | Add alternative token |
| `updatePaymentMethod(paymentMethodId, active)` | Enable/disable token |
| `withdrawFunds(tokenAddress, to)` | Move contract balance to recipient |
| `setSlippageBps(bps)` | Set price slippage tolerance (max 1000 = 10%) |
| `setMinLiquidity(amount)` | Set minimum liquidity for DEX pairs |

#### Public (anyone)

| Function | Purpose |
|----------|---------|
| `buySubscription(planId, days, paymentMethodId, userEncrypted)` | Purchase subscription, emit event |
| `extendSubscription(subscriptionId, days, paymentMethodId)` | Extend (anyone can extend any — enables gifting) |

#### View (read-only)

| Function | Returns | Purpose |
|----------|---------|---------|
| `getSubscription(id)` | `(planId, subscriber, expiresAt, isActive, cancelled)` | Subscription details |
| `getPlan(id)` | `(name, pricePerDayUsdCents, active)` | Plan details |
| `daysRemaining(id)` | `uint256` | 0 if expired/cancelled |
| `getPrimaryStablecoin()` | `address` | Primary stablecoin address |
| `getPaymentMethodIds()` | `uint256[]` | All method IDs (includes 1 if configured) |
| `getPaymentMethod(id)` | `(tokenAddress, pairAddress, stablecoinAddress, tokenDecimals, stablecoinDecimals, active)` | Method details |
| `calculatePayment(planId, days, paymentMethodId)` | `uint256` | Estimated token cost |
| `getTokenPriceUsdCents(paymentMethodId)` | `uint256` | Token price (100 = $1.00 for stablecoin) |
| `getTotalSubscriptionCount()` | `uint256` | Total subscriptions |
| `getTotalPlanCount()` | `uint256` | Total plans |
| `getSubscriptionsBySubscriber(address)` | `uint256[]` | Subscriber's subscription IDs |

### Events (monitored by engine)

| Event | Key Fields | Monitor Handler |
|-------|-----------|-----------------|
| `SubscriptionCreated` | `subscriptionId, planId, subscriber, expiresAt, paidAmount, paymentToken, userEncrypted` | Create VM + mint NFT |
| `SubscriptionExtended` | `subscriptionId, planId, extendedBy, newExpiresAt, paidAmount, paymentToken` | Extend VM expiry |
| `SubscriptionCancelled` | `subscriptionId, planId, subscriber` | Destroy VM |
| `PlanCreated` | `planId, name, pricePerDayUsdCents` | Log only |
| `PlanUpdated` | `planId, name, pricePerDayUsdCents, active` | Log only |
| `PrimaryStablecoinSet` | `stablecoinAddress, decimals` | Log only |
| `PaymentMethodAdded` | `paymentMethodId, tokenAddress, pairAddress, stablecoinAddress` | Log only |
| `PaymentMethodUpdated` | `paymentMethodId, active` | Log only |
| `FundsWithdrawn` | `token, to, amount` | Log only |

### Payment Calculation

**Primary stablecoin (method ID 1):** Direct USD. No price discovery.
```
tokenAmount = pricePerDayUsdCents * days * 10^decimals / 100
```

**Other tokens (ID 2+):** Uniswap V2 constant product pricing.
```
tokenAmount = (totalUsdCost * tokenReserve / stablecoinReserve) * (1 + slippageBps/10000)
```
Requires `stablecoinReserve >= minLiquidityUsd` (default $10,000).

### EVM-Specific Notes

- Contract inherits OpenZeppelin `Ownable`, `ReentrancyGuard`
- Token transfers use `SafeERC20`
- Price feed uses Uniswap V2 pair reserves (no Chainlink oracles)
- `userEncrypted` in `buySubscription` is ECIES-encrypted connection details — the engine decrypts with `server.key` after VM creation and re-encrypts for the NFT
- Subscription IDs are sequential starting from 1

---

## 3. Monitor Service

The monitor is the adapter boundary between the chain and the provisioner. It translates chain events into provisioner CLI calls.

### Event Polling Loop

**Interval:** 5 seconds

**Cycle:**
1. Query contract logs from `lastProcessedBlock` to `currentBlock`
2. Decode events against contract ABI
3. Dispatch to event handlers (see below)
4. Process admin commands (if configured)
5. Periodic tasks (on their own intervals):
   - **Reconciliation:** every 5 minutes
   - **Fund cycle:** every 24 hours (configurable)
   - **Gas check:** every 30 minutes (configurable)

### Event Handlers

**`SubscriptionCreated`:**
1. Format VM name: `blockhost-NNN` (3-digit zero-padded subscription ID)
2. Calculate expiry days from `expiresAt` timestamp
3. Call provisioner: `blockhost-vm-create blockhost-NNN --owner-wallet <subscriber> --no-mint --apply`
4. Parse JSON output from provisioner (ip, vmid, nft_token_id, username)
5. Decrypt `userEncrypted` using server private key
6. Call: `blockhost-mint-nft --owner-wallet <subscriber> --machine-id blockhost-NNN --user-encrypted <encrypted_details> --public-secret <secret>`
7. Mark NFT minted in database

**`SubscriptionExtended`:**
1. Calculate additional days
2. Update VM database expiry
3. If VM was suspended: call provisioner `resume` command

**`SubscriptionCancelled`:**
1. Call provisioner: `blockhost-vm-destroy blockhost-NNN`

**`PlanCreated`, `PlanUpdated`:**
Log only (informational).

### NFT Reconciliation

**Interval:** 5 minutes. **Guards:** skips if provisioning in progress (`pgrep` for create command or lock file).

Verifies local `vms.json` NFT state matches on-chain:
- Checks for tokens that exist on-chain but aren't marked minted locally
- Checks `reserved_nft_tokens` map for un-minted reservations
- Fixes discrepancies by marking tokens as minted (via Python `blockhost.vm_db` or direct JSON update)

**Config reads:** `web3-defaults.yaml` (nft_contract), `blockhost.yaml` (fallback)

### Graceful Shutdown

On SIGINT: `closeAllKnocks()` (if admin commands enabled) → `process.exit(0)`.

---

## 4. Fund Manager

Automated financial operations, integrated into the monitor polling loop.

### Fund Cycle (default: every 24 hours)

1. **Load addressbook**, ensure hot wallet exists (auto-generate via root agent if missing)
2. **Withdraw** — for each active payment method with balance > `min_withdrawal_usd`: call `contract.withdrawFunds(token, hot_wallet)`
3. **Hot wallet gas top-up** — if `hot.eth < hot_wallet_gas_eth`, server sends ETH to hot wallet
4. **Server stablecoin buffer** — if `server.stablecoin < server_stablecoin_buffer_usd`, hot sends stablecoin to server
5. **Revenue shares** — if enabled in `revenue-share.json`, hot distributes configured % to dev, broker
6. **Remainder to admin** — all remaining hot wallet token balances → admin
7. **Update state:** `last_fund_cycle = Date.now()`

### Gas Check (default: every 30 minutes)

1. Check server wallet ETH balance (convert to USD via Uniswap V2 pair)
2. If below `gas_low_threshold_usd`: swap `gas_swap_amount_usd` of USDC → ETH
3. Top up hot wallet gas if needed
4. **Update state:** `last_gas_check = Date.now()`

### Hot Wallet

Auto-generated on first fund cycle if not in addressbook:
- Root agent `generate-wallet` action creates `/etc/blockhost/hot.key` (chmod 600)
- Added to addressbook as `hot` with keyfile path
- Acts as intermediary — contract funds flow through hot wallet before distribution

### Configuration

**`/etc/blockhost/blockhost.yaml`** — under `fund_manager:` key (all settings have defaults, entire section optional):

```yaml
fund_manager:
  fund_cycle_interval_hours: 24        # Hours between fund cycles
  gas_check_interval_minutes: 30       # Minutes between gas checks
  min_withdrawal_usd: 50               # Min USD balance to trigger withdrawal
  gas_low_threshold_usd: 5             # Server ETH (in USD) that triggers swap
  gas_swap_amount_usd: 20              # USDC amount to swap per cycle
  server_stablecoin_buffer_usd: 50     # Target server stablecoin balance
  hot_wallet_gas_eth: 0.01             # Target hot wallet ETH balance
```

### DEX Integration

**Module:** `chain-pools.ts` — per-chain Uniswap V2 configuration.

```typescript
interface ChainConfig {
  router: string;         // Uniswap V2 Router address
  weth: string;           // WETH token address
  usdc: string;           // USDC stablecoin address
  usdc_weth_pair: string; // USDC/WETH pair address
}
```

Known chains: Ethereum mainnet, Base, Sepolia. Router/WETH/pair addresses are hardcoded per chain ID.

**ABI used:**
- `swapExactTokensForETH(uint amountIn, uint amountOutMin, address[] path, address to, uint deadline)`
- `getAmountsOut(uint amountIn, address[] path)`

---

## 5. Admin Command Protocol

Encrypted on-chain commands from the admin wallet, processed by the monitor.

### Configuration

**`/etc/blockhost/blockhost.yaml`** — under `admin:` key:

```yaml
admin:
  wallet_address: "0x..."           # Admin wallet that sends commands
  credential_nft_id: 0              # NFT token ID for admin auth
  max_command_age: 300              # Reject commands older than N seconds
  destination_mode: "self"          # "any" | "self" | "server" | "null"
  destination_address: "0x..."      # For "null" mode (optional)
```

### Command Database

**File:** `/etc/blockhost/admin-commands.json`

```json
{
  "commands": {
    "secret-command-name": {
      "action": "knock",
      "description": "Open SSH and management ports",
      "params": {
        "allowed_ports": [22, 8006],
        "default_duration": 300
      }
    }
  }
}
```

### Processing Flow

1. Monitor scans transactions from `admin.wallet_address` in each block range
2. Destination check (based on `destination_mode`):
   - `any`: accept all tx destinations
   - `self`: only `tx.to == admin.wallet_address`
   - `server`: only `tx.to == ethers.computeAddress(server_public_key)`
   - `null`: only `tx.to == destination_address`
3. ECIES decryption of `tx.data` using `/etc/blockhost/server.key`
4. Parse JSON payload, validate command format
5. Timestamp check (not older than `max_command_age`, not in future)
6. Nonce anti-replay check (each nonce used only once)
7. Dispatch to action handler (lookup in command database)
8. Log result

### Command Payload (encrypted in tx.data)

```json
{
  "command": "secret-command-name",
  "params": {"ports": [22], "duration": 60, "source": "2001:db8::1"},
  "nonce": "uuid-string",
  "timestamp": 1708000000
}
```

### Implemented Handlers

#### `knock` — temporary firewall port opening

**Params:**

| Field | Type | Description |
|-------|------|-------------|
| `ports` | `number[]` | Ports to open (validated against `allowed_ports` in command config) |
| `duration` | `number` | Seconds to keep open (defaults to `default_duration` from config) |
| `source` | `string` | Optional IPv6 source filter |

**Two-phase lifecycle:**

1. **Phase 1 (pre-login):** Open ports via root agent `iptables-open`. Start heartbeat poller (checks `/run/blockhost/knock.active`). Duration timeout triggers automatic close.
2. **Phase 2 (post-login):** When `/run/blockhost/knock.active` appears with login IP, close broad rules and reopen narrowed to login IP. Heartbeat monitors file staleness (15 min timeout).

**Key files:**
- `/run/blockhost/knock.active` — login IP written by auth.log monitor
- `/var/log/auth.log` — monitored for `"Accepted publickey..."` lines

---

## 6. Config Files

### Files Read

| File | What | Read by |
|------|------|---------|
| `/opt/blockhost/.env` | `RPC_URL`, `BLOCKHOST_CONTRACT` | Monitor, bw, ab (env vars) |
| `/etc/blockhost/blockhost.yaml` | Server keys, contract address, admin config, fund_manager config | Monitor, bw who, init, generate-signup, reconcile |
| `/etc/blockhost/web3-defaults.yaml` | Chain ID, NFT contract, RPC URL, deployer key path | mint_nft, bw who, reconcile, generate-signup |
| `/etc/blockhost/addressbook.json` | Role-to-wallet mapping | bw, ab, fund-manager, monitor |
| `/etc/blockhost/admin-commands.json` | Knock command definitions | Monitor admin command handler |
| `/etc/blockhost/revenue-share.json` | Revenue distribution config | Fund-manager |
| `/usr/share/blockhost/provisioner.json` | Provisioner manifest (command dispatch) | Monitor event handlers, reconcile |

### Files Written

| File | What | Written by |
|------|------|-----------|
| `/var/lib/blockhost/fund-manager-state.json` | Last run timestamps, hot wallet status | Fund-manager |
| `/var/lib/blockhost/vms.json` | VM database (NFT reconciliation updates) | Reconcile module |
| `/etc/blockhost/addressbook.json` | Hot wallet entry (via root agent) | Fund-manager, ab CLI |
| `/run/blockhost/knock.active` | Login IP for knock phase 2 | Knock handler auth.log monitor |

### Shared Files (read-only for engine, written by installer)

| File | Schema reference |
|------|-----------------|
| `/etc/blockhost/blockhost.yaml` | See `COMMON_INTERFACE.md` §7 |
| `/etc/blockhost/web3-defaults.yaml` | See `COMMON_INTERFACE.md` §7 |
| `/etc/blockhost/addressbook.json` | See §1 `ab` (addressbook JSON schema) |

### `fund-manager-state.json` Schema

```json
{
  "last_fund_cycle": 0,        // Unix timestamp (ms) of last fund cycle
  "last_gas_check": 0,         // Unix timestamp (ms) of last gas check
  "hot_wallet_generated": false // Whether hot wallet has been created
}
```

**Location:** `/var/lib/blockhost/fund-manager-state.json`
**Owner:** `blockhost:blockhost`
**Written by:** fund-manager state module (on every cycle/check completion)

### `revenue-share.json` Schema

```json
{
  "enabled": true,
  "total_percent": 1.0,
  "recipients": [
    {"role": "dev", "percent": 0.5},
    {"role": "broker", "percent": 0.5}
  ]
}
```

**Location:** `/etc/blockhost/revenue-share.json`
**Owner:** `root:blockhost` (mode 640)
**Written by:** installer finalization (`_finalize_config`)
**Read by:** fund-manager (revenue distribution step)

`recipients[].role` maps to addressbook keys. Percentages should sum to ≤ 1.0. If `enabled: false`, all hot wallet funds go directly to admin.

---

## 7. Environment Variables

| Variable | Source | Consumers | Description |
|----------|--------|-----------|-------------|
| `RPC_URL` | `/opt/blockhost/.env` | Monitor, bw, ab | Ethereum JSON-RPC endpoint |
| `BLOCKHOST_CONTRACT` | `/opt/blockhost/.env` | Monitor, bw, ab | Subscription contract address |
| `NODE_OPTIONS` | Systemd unit / wrapper scripts | Monitor, bw, ab | Set to `--dns-result-order=ipv4first` |

### `.env` File Schema

```bash
RPC_URL=https://ethereum-sepolia-rpc.publicnode.com
BLOCKHOST_CONTRACT=0xYourContractAddressHere
```

**Location:** `/opt/blockhost/.env`
**Owner:** `root:blockhost` (mode 640)
**Written by:** installer finalization (`_finalize_config`)
**Also contains (written by installer but not consumed by engine runtime):**
- `NFT_CONTRACT` — used by `validate_system.py`
- `DEPLOYER_KEY_FILE` — used by `validate_system.py`

### Loading Pattern

Consumers load via `EnvironmentFile=` in systemd units (monitor) or by parsing the file directly (admin panel's `_get_bw_env()`).

---

## 8. Systemd Units & .deb Package

### Service: `blockhost-monitor.service`

```ini
[Unit]
Description=Blockhost Subscriptions Event Monitor
After=network.target
Requires=blockhost-root-agent.service
After=blockhost-root-agent.service

[Service]
Type=simple
User=blockhost
Group=blockhost
Environment=HOME=/var/lib/blockhost
Environment=NODE_OPTIONS=--dns-result-order=ipv4first
EnvironmentFile=/opt/blockhost/.env
ExecStart=/usr/bin/node /usr/share/blockhost/monitor.js
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
NoNewPrivileges=true
ProtectSystem=strict
PrivateTmp=true
ReadWritePaths=/var/lib/blockhost

[Install]
WantedBy=multi-user.target
```

**Installed to:** `/lib/systemd/system/blockhost-monitor.service`

### Package: `blockhost-engine`

**Name:** `blockhost-engine_<version>_all.deb`

**Dependencies:**
```
Depends: blockhost-common (>= 0.1.0), libpam-web3-tools (>= 0.5.0), nodejs (>= 18), python3 (>= 3.10)
```

### Installed File Locations

| Content | Destination |
|---------|-------------|
| Monitor (esbuild bundle) | `/usr/share/blockhost/monitor.js` |
| bw CLI (esbuild bundle) | `/usr/share/blockhost/bw.js` |
| bw wrapper | `/usr/bin/bw` |
| ab CLI (esbuild bundle) | `/usr/share/blockhost/ab.js` |
| ab wrapper | `/usr/bin/ab` |
| is CLI (esbuild bundle) | `/usr/share/blockhost/is.js` — PLANNED |
| is wrapper | `/usr/bin/is` — PLANNED |
| Contract deployer | `/usr/bin/blockhost-deploy-contracts` — PLANNED |
| NFT minter (Python) | `/usr/bin/blockhost-mint-nft` |
| NFT minter (module) | `/usr/lib/python3/dist-packages/blockhost/mint_nft.py` |
| Init script | `/usr/bin/blockhost-init` — DEPRECATION PLANNED |
| Signup generator | `/usr/bin/blockhost-generate-signup` |
| Signup template | `/usr/share/blockhost/signup-template.html` |
| Wizard plugin | `/usr/lib/python3/dist-packages/blockhost/engine_evm/` — PLANNED |
| Contract source | `/opt/blockhost/contracts/BlockhostSubscriptions.sol` |
| Contract artifact | `/usr/share/blockhost/contracts/BlockhostSubscriptions.json` |
| Deploy scripts | `/opt/blockhost/scripts/deploy.ts`, `create-plan.ts` |
| Systemd unit | `/lib/systemd/system/blockhost-monitor.service` |
| Env example | `/opt/blockhost/.env.example` |

### Package Naming Convention (chain variants)

The current package is `blockhost-engine` (EVM). Future chain-specific variants should follow `blockhost-engine-<chain>` (e.g., `blockhost-engine-opnet`). Engine packages should declare `Conflicts:` with each other — only one engine can be active per host.

---

## 9. Signup Page

The signup page is a static HTML file generated by `blockhost-generate-signup` and served by nginx at `/`.

### Generation Flow

1. Installer finalization calls `blockhost-generate-signup --output /var/www/blockhost/signup.html`
2. Script reads `blockhost.yaml` and `web3-defaults.yaml`
3. Template placeholders replaced with config values
4. Static HTML written to output path

### nginx Serving

```
location / {
    root /var/www/blockhost;
    index signup.html;
}
```

### Chain-Specific Fields

The following template placeholders are EVM-specific and would need equivalents for other chains:

| Placeholder | EVM meaning | Abstraction needed |
|-------------|-------------|-------------------|
| `{{CHAIN_ID}}` | Ethereum chain ID (e.g., 11155111) | Chain identifier |
| `{{RPC_URL}}` | JSON-RPC endpoint | Node endpoint |
| `{{USDC_ADDRESS}}` | ERC20 stablecoin address | Payment token identifier |
| `{{NFT_CONTRACT}}` | ERC721 contract address | Credential contract identifier |
| `{{SUBSCRIPTION_CONTRACT}}` | Custom contract address | Subscription contract identifier |

`{{SERVER_PUBLIC_KEY}}` and `{{PUBLIC_SECRET}}` are chain-agnostic (secp256k1 ECIES).

---

## 10. Engine Wizard Plugin — PLANNED

The engine contributes a blockchain configuration page and finalization steps to the installer wizard, following the same plugin pattern as the provisioner (see `PROVISIONER_INTERFACE.md` §3).

### Motivation

The installer's `blockchain.html` wizard page and all chain-specific finalization steps (`_finalize_keypair`, `_finalize_wallet`, `_finalize_contracts`, `_finalize_config` chain fields, `_finalize_mint_nft`, `_create_default_plan`) currently live in the installer repo with hardcoded EVM assumptions. Moving them to the engine makes the installer chain-agnostic.

### Engine Manifest Extension

Add to a new engine manifest file (analogous to `provisioner.json`):

```json
{
  "name": "evm",
  "version": "0.1.0",
  "display_name": "EVM (Ethereum/Base/Sepolia)",

  "setup": {
    "wizard_module": "blockhost.engine_evm.wizard",
    "finalization_steps": ["keypair", "wallet", "contracts", "config", "mint_nft", "plan", "signup"]
  }
}
```

### Required Exports

Same pattern as provisioner wizard plugin:

| Export | Type | Signature |
|--------|------|-----------|
| `blueprint` | `flask.Blueprint` | Registers blockchain wizard route(s) |
| `get_finalization_steps()` | function | `-> list[tuple[str, str, callable]]` |
| `get_summary_data(session)` | function | `-> dict` |
| `get_summary_template()` | function | `-> str` |

### What Moves to Engine

| Current location | Moves to |
|-----------------|----------|
| `installer/web/templates/wizard/blockchain.html` | Engine wizard template |
| `installer/web/finalize.py` → `_finalize_keypair` | Engine finalization step |
| `installer/web/finalize.py` → `_finalize_wallet` | Engine finalization step (calls `ab new server`) |
| `installer/web/finalize.py` → `_finalize_contracts` | Engine finalization step (calls `blockhost-deploy-contracts`) |
| `installer/web/finalize.py` → `_finalize_mint_nft` | Engine finalization step (calls `blockhost-mint-nft` + `bw set encrypt`) |
| `installer/web/finalize.py` → `_create_default_plan` | Engine finalization step (calls `bw plan create` + `bw config stable`) |
| `installer/web/finalize.py` → `_finalize_signup` | Engine finalization step (calls `blockhost-generate-signup`) |
| `installer/web/utils.py` → `generate_secp256k1_keypair*` | Engine wizard module (or `ab new`) |
| `installer/web/utils.py` → `get_address_from_key` | Engine wizard module (or `ab new`) |

### What Stays in Installer

- Wizard framework (step bar, navigation, session management)
- Non-chain finalization steps (network, HTTPS, nginx, validation)
- OTP auth, provisioner plugin discovery
- `validate_system.py` (but chain-specific checks delegate to engine CLIs like `is contract`)

---

## 11. Known Issues & Abstraction Debt

### ~~`cast` used directly by installer and admin~~ (PLANNED)

All `cast` calls in the installer and admin are replaced by engine CLIs:

| `cast` call | Replaced by | Location |
|-------------|-------------|----------|
| `cast send --create` (contract deployment) | `blockhost-deploy-contracts [nft\|pos]` | `finalize.py` |
| `cast send createPlan` | `bw plan create <name> <price>` | `finalize.py` |
| `cast send setPrimaryStablecoin` | `bw config stable <address>` | `finalize.py` |
| `cast send updateUserEncrypted` | `bw set encrypt <nft_id> <data>` | `finalize.py` |
| `cast call balanceOf` (NFT) | `bw balance <contract>` | `finalize.py` |
| `cast call tokenOfOwnerByIndex` | Eliminated — ID is 0 (fresh deploy) or user-supplied | `finalize.py` |
| `cast wallet verify` | `is <signature> <wallet>` | `admin/auth.py` |
| `cast wallet address` | `ab new` (derives address internally) | `utils.py` |
| `cast call totalSupply` | `is contract <address>` | `validate_system.py` |

Chain-specific finalization steps move to engine wizard plugin (§10). The installer becomes chain-agnostic.

### `mint_nft.py` dual install path (OPEN)

Installed as both `/usr/bin/blockhost-mint-nft` (CLI) and `/usr/lib/python3/dist-packages/blockhost/mint_nft.py` (importable module). The monitor calls it as a CLI; the installer imports it as Python. Both paths must work. With the engine wizard plugin, the Python import path is used by the engine's own finalization step (same submodule), reducing the cross-boundary concern.

### Hardcoded chain configs in `chain-pools.ts` (OPEN)

Uniswap V2 router addresses, WETH addresses, and pair addresses are hardcoded per chain ID. Adding a new EVM chain requires code changes. Should be configurable (in `blockhost.yaml` or `web3-defaults.yaml`).

### No BLOCK_TIME constant (OPEN)

Monitor polling intervals (5s event poll, 5min reconcile, 30min gas check, 24h fund cycle) are hardcoded or configured in seconds/minutes/hours. A chain-specific `BLOCK_TIME` constant could scale these intervals relative to block production speed, making them meaningful across chains with different block times.

### ~~Signup page placeholders are EVM-specific~~ (PLANNED)

Resolved by engine wizard plugin. Each engine ships its own signup template via `blockhost-generate-signup`. The template and its placeholders are engine-owned — a second engine provides its own template with chain-appropriate fields.

### ~~`installer/web/utils.py` generates secp256k1 keys directly~~ (PLANNED)

Key generation moves to engine wizard plugin finalization steps. The wizard UI calls `ab new server` when the user clicks "generate." The installer no longer imports `eth_keys`, `ecdsa`, or calls `cast wallet address` — all key operations go through engine CLIs.

### ~~`validate_system.py` uses `cast call` for contract queries~~ (PLANNED)

Replaced by `is contract <address>` — a chain-agnostic contract liveness check.
