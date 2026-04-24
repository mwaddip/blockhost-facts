# Engine Interface Specification

> Authoritative reference for the blockchain adapter boundary.
> The engine is the only component that talks to the chain bidirectionally.
> Any second engine implementation (e.g., `blockhost-engine-opnet`) must satisfy
> this contract to be a drop-in replacement.
>
> Derived from the working EVM implementation (`blockhost-engine-evm`).
> See also: `COMMON_INTERFACE.md` (shared library API), `ADMIN_INTERFACE.md`
> (admin panel — a consumer of engine CLIs).

---

## 1. CLI Commands

All commands are installed to `/usr/bin/` by the engine `.deb`. Consumers shell out
to these as subprocesses. Exit 0 = success, non-zero = failure (stderr has error details).

---

### `is` (identity predicate)

**Installed:** `/usr/bin/is`

Standalone binary. Answers yes/no identity questions via exit code. No env vars, no addressbook — pure on-chain queries. Arguments are order-independent; types are unambiguous (addresses are `0x` + 40 hex, NFT IDs are integers, `contract` is a keyword).

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

### `blockhost-deploy-contracts`

**Installed:** `/usr/bin/blockhost-deploy-contracts` (bash on EVM, TypeScript on OPNet/Cardano/Ergo)

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

Works for both ERC20 tokens and NFT contracts — both implement `balanceOf()`. For NFTs, returns integer count (no decimals). The `role` argument also accepts a literal contract address (e.g., the NFT contract address), not just addressbook roles.

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
| `token` | no | Token to withdraw. Account-model chains: token address, defaults to primary stablecoin. UTXO chains: may ignore (withdrawal collects all claimable UTXOs regardless of token). |
| `to` | yes | Recipient address or role |

Collects accumulated subscription revenue. Account-model engines call a contract withdraw function. UTXO engines scan for claimable subscription UTXOs (via beacon tokens) and batch-collect them — the `token` parameter may be ignored since each UTXO carries its own payment asset in its datum.

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

Two forms — NFT owner lookup and signature recovery.

```
bw who <identifier>
bw who <message> <signature>
```

**Form 1: NFT owner lookup**

| Arg | Required | Description |
|-----|----------|-------------|
| `identifier` | yes | Numeric NFT token ID, or `admin` (resolves `admin.credential_nft_id` from `blockhost.yaml`) |

Queries `ownerOf(tokenId)` on the AccessCredentialNFT contract. Config sourced from `web3-defaults.yaml` (`blockchain.nft_contract`, `blockchain.rpc_url`). **No env vars or addressbook required.**

**stdout:** Owner address (`0x...`).
**Exit:** 0 if found, 1 if not found or config missing.
**Consumers:** `admin/auth.py` (`["bw", "who", "admin"]` — resolves admin NFT holder for auth)

**Form 2: Signature recovery**

| Arg | Required | Description |
|-----|----------|-------------|
| `message` | yes | The message that was signed |
| `signature` | yes | The signature (hex) |

Recovers the signer address from a message and signature. Replaces `cast wallet verify` — the caller compares the returned address with the expected address. Pure crypto, no chain queries.

**stdout:** Signer address (`0x...`).
**Exit:** 0 if recovery succeeds, 1 on invalid signature.
**Consumers:** `admin/auth.py` (replaces `cast wallet verify --address`; caller compares with `bw who admin` output)

#### `bw config stable`

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

#### `bw plan create`

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

#### `bw set encrypt`

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

**Environment:** `RPC_URL`, `BLOCKHOST_CONTRACT`.

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

Generate new secp256k1 keypair via `bhcrypt generate-keypair`. Saves private key to `/etc/blockhost/<name>.key` (chmod 600), adds entry to addressbook with `keyfile` field.

**Consumers:** `admin/system.py` (`_run_ab(["new", name])`)

#### `ab list`

```
ab list
```

**stdout:** JSON — full addressbook contents (`{role: {address, keyfile?}, ...}`).
**Exit:** Always 0.

**Consumers:** None directly (addressbook read by file I/O in most consumers).

#### `ab --init`

```
ab --init <admin> <server> [dev] [broker] <keyfile>
```

Bootstrap the addressbook with initial required entries. Positional args: admin address, server address, optionally dev and broker addresses, then server keyfile path (always last). Fails if addressbook already has entries.

The deployer/server wallet is generated interactively through the wizard UI (which calls `ab new server` behind the scenes when the user clicks "generate"). `ab --init` takes the generated entries and writes a properly structured addressbook.

**Consumers:** Engine wizard finalization step

#### Immutable Roles

`server`, `admin`, `hot`, `dev`, `broker` — cannot be added, deleted, updated, or generated via `ab add/del/up/new` (except `ab --init` for bootstrap). These are managed by the engine wizard finalization and fund-manager.

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

**Installed:** `/usr/bin/blockhost-mint-nft`

```
blockhost-mint-nft \
  --owner-wallet <address> \
  [--user-encrypted <hex>] \
  [--dry-run]
```

| Arg | Required | Default | Description |
|-----|----------|---------|-------------|
| `--owner-wallet` | yes | — | Wallet address to receive NFT (chain-agnostic format) |
| `--user-encrypted` | no | `""` | Hex-encoded encrypted connection details |
| `--dry-run` | no | false | Simulate without broadcasting |

**Config:** `web3-defaults.yaml` → `blockchain.nft_contract`, `blockchain.rpc_url`. Deployer key from env or `/etc/blockhost/deployer.mnemonic` (OPNet) / `deployer.key` (EVM).

**Implementation:** Calls `mint(address, userEncrypted)` on the AccessCredentialNFT contract (2 params). Returns the minted token ID on stdout.

**Consumers:**
- Monitor event handler (CLI: `blockhost-mint-nft --owner-wallet ... --user-encrypted ...`)
- Engine wizard `finalize_mint_nft()` (CLI call)

**Exit:** 0/1. stdout = token ID on success.

---

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

> **Scope note:** Sections 2–4 describe engine-internal semantics, not cross-boundary contracts. The function signatures below reflect the EVM/OPNet account-model pattern. UTXO-based engines (Cardano, Ergo) satisfy the same *external behavior* (plans exist, subscriptions are purchasable, funds are collectible) through different mechanisms (validator spending conditions, UTXO scanning, batch collection transactions). The portable interface boundary is the CLI commands in §1 and the monitor output behavior in §3 (triggers create/extend/destroy via provisioner CLI). Setup-only commands (`bw plan create`, `bw config stable`, `blockhost-deploy-contracts`) and revenue collection (`withdrawFunds`) are engine-internal — no other module calls them during normal operation.

The subscription contract manages plans, subscriptions, and payment methods. Any chain adapter must implement equivalent semantics.

### Timing Rule

**All on-chain timing must use block height, not timestamps.** Subscription expiry, fund cycles, any duration measured on-chain — express in blocks, not seconds. Block height is deterministic, monotonically increasing, and consistent across all chains. `block.timestamp` (or equivalent) is miner-adjustable, varies across chains, and drifts. The monitor and fund manager convert block heights to wall-clock estimates using the chain's known average block time when needed for display or provisioner calls — but the source of truth on-chain is always height.

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

### OPNet-Specific Notes

- `days` parameter in `buySubscription` and `extendSubscription` is capped at 36500 (~100 years) — reverts with `Days exceeds maximum` if exceeded. Guards against u256→u64 truncation in duration calculation.
- Per-subscriber subscription arrays use `StoredU256Array` with full 32-byte `Address` as sub-pointer (no truncation)

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
5. Periodic tasks (on their own intervals, all `await`ed, all guarded by `isPipelineBusy()`):
   - **Reconciliation:** every 5 minutes
   - **Fund cycle:** every 24 hours (configurable)
   - **Gas check:** every 30 minutes (configurable)

**Background task scheduling:** Reconciliation, fund cycle, and gas check are launched in the same polling cycle as event handlers. Each runs on its own interval and `await`s to completion (not fire-and-forget). There is no pipeline gate — handlers and background tasks share the polling tick and rely on the reconciler to clean up after partial failures.

### Event Handlers

**`SubscriptionCreated`** (handled inline in `src/handlers/index.ts`):
1. Format VM name: `blockhost-NNN` (3-digit zero-padded subscription ID)
2. Calculate expiry days from `expiresAt` timestamp
3. Decrypt `userEncrypted` using server private key (ECIES). If decryption fails, abort before creating VM.
4. Call provisioner: `blockhost-vm-create blockhost-NNN --owner-wallet <subscriber> --expiry-days <days> --apply`
5. Parse JSON output from provisioner (ip, vmid, ipv6, username)
6. Register VM in `vms.json` via `blockhost.vm_db.register_vm`
7. Encrypt connection details (hostname, port, username) using decrypted user signature (symmetric)
8. Call: `blockhost-mint-nft --owner-wallet <subscriber> --user-encrypted <encrypted_details>` → capture token ID from stdout
9. Call provisioner: `blockhost-vm-update-gecos blockhost-NNN <subscriber> --nft-id <actual_token_id>`
10. Mark NFT minted via `blockhost.vm_db.set_nft_minted(vm_name, token_id)`

No token reservation. The actual minted token ID comes from `blockhost-mint-nft` stdout, then gets baked into GECOS via `update-gecos`. If any step fails partway, the reconciler picks up missing NFTs on its 5-minute cycle.

**`SubscriptionExtended`:**
1. Calculate additional days
2. Update VM database expiry
3. If VM was suspended: call provisioner `resume` command

**`SubscriptionCancelled`:**
1. Call provisioner: `blockhost-vm-destroy blockhost-NNN`

**`PlanCreated`, `PlanUpdated`:**
Log only (informational).

### NFT Reconciliation

**Interval:** 5 minutes.

Verifies local `vms.json` NFT state matches on-chain. Two responsibilities:

**1. NFT minting reconciliation.** For each active/suspended VM where `nft_minted !== true`, queries on-chain `ownerOf(tokenId)`. If the token exists, marks it as minted locally and updates GECOS if needed. If not, logs a warning for operator attention.

**2. NFT ownership transfer detection.** For every active/suspended VM with a minted NFT, compares `ownerOf(tokenId)` on-chain with the locally stored `owner_wallet`. When a transfer is detected:
- Updates `owner_wallet` in `vms.json` to the new on-chain owner
- Sets `gecos_synced = false` on the VM entry
- Calls provisioner `update-gecos` to update the VM's GECOS field (`wallet=ADDRESS,nft=TOKEN_ID`)
- On success, sets `gecos_synced = true`

If `update-gecos` fails (VM stopped, guest agent unresponsive), `gecos_synced = false` persists and the next cycle retries. This is the sole mechanism by which VMs learn about NFT ownership changes post-creation. libpam-web3 verifies signatures against the GECOS-stored wallet address — no chain queries at auth time.

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
- Root agent `generate-wallet` action creates `/etc/blockhost/hot.key` (chmod 640)
- Added to addressbook as `hot` with keyfile path
- Acts as intermediary — contract funds flow through hot wallet before distribution

### Configuration

**`/etc/blockhost/blockhost.yaml`** — under `fund_manager:` key (all settings have defaults, entire section optional).

Threshold key naming follows the pattern `<purpose>_<unit>` where `<unit>` is the chain's native base unit. Each engine ships its own defaults and reads keys named for its chain. The interval keys (`fund_cycle_interval_hours`, `gas_check_interval_minutes`) are universal across all engines.

| Engine | Native unit suffix | Stablecoin unit suffix |
|--------|-------------------|----------------------|
| EVM | `_eth` (gas), `_usd` (balances) | `_usd` |
| OPNet | `_sats` | `_sats` |
| Cardano | `_lovelace` | `_lovelace` |
| Ergo | `_nanoerg` | `_nanoerg` |

EVM uses USD-denominated values for stablecoin thresholds because Uniswap V2 price discovery converts stablecoin↔ETH on the fly. The other engines use base units directly because they don't perform live USD conversion.

**Example (EVM):**

```yaml
fund_manager:
  fund_cycle_interval_hours: 24
  gas_check_interval_minutes: 30
  min_withdrawal_usd: 50
  gas_low_threshold_usd: 5
  gas_swap_amount_usd: 20
  server_stablecoin_buffer_usd: 50
  hot_wallet_gas_eth: 0.01
```

**Example (Cardano — all values in lovelace, 1 ADA = 1,000,000 lovelace):**

```yaml
fund_manager:
  fund_cycle_interval_hours: 24
  gas_check_interval_minutes: 30
  min_withdrawal_lovelace: 50000000           # 50 ADA
  gas_low_threshold_lovelace: 5000000         # 5 ADA
  gas_swap_amount_lovelace: 10000000          # 10 ADA
  server_stablecoin_buffer_lovelace: 5000000
  hot_wallet_gas_lovelace: 5000000            # 5 ADA
```

OPNet substitutes `_sats`, Ergo substitutes `_nanoerg` in the same positions.

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
| `/opt/blockhost/.env` | `RPC_URL`, `BLOCKHOST_CONTRACT`, plus engine-specific vars | Monitor, bw, ab (env vars) |
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

Two formats are accepted. Newer engines (OPNet, Cardano, Ergo) read and write the basis-points form; the percent form is supported for backward compatibility.

**Basis-points form (preferred):**

```json
{
  "enabled": true,
  "total_bps": 100,
  "recipients": [
    {"role": "dev", "bps": 50},
    {"role": "broker", "bps": 50}
  ]
}
```

`100 bps = 1%`, `10000 bps = 100%`. Recipients' `bps` must sum exactly to `total_bps`.

**Percent form (legacy, EVM):**

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

When loading, engines convert `total_percent → total_bps` via `Math.round(percent * 100)`. Both keys may coexist.

**Location:** `/etc/blockhost/revenue-share.json`
**Owner:** `root:blockhost` (mode 640)
**Written by:** engine wizard finalization (revenue_share step)
**Read by:** fund-manager (revenue distribution step)

`recipients[].role` maps to addressbook keys. If `enabled: false` or `total_bps == 0`, all hot wallet funds go directly to admin.

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
ExecStartPre=+/bin/bash -c 'chown root:blockhost /run/blockhost && chmod 2775 /run/blockhost'
ExecStart=/usr/bin/node /usr/share/blockhost/monitor.js
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
NoNewPrivileges=true
ProtectSystem=strict
PrivateTmp=true
ReadWritePaths=/var/lib/blockhost /run/blockhost

[Install]
WantedBy=multi-user.target
```

**Installed to:** `/lib/systemd/system/blockhost-monitor.service`

The `ExecStartPre=+` line runs as root (the `+` prefix bypasses `User=`) to ensure `/run/blockhost` exists with the correct ownership/perms before the monitor drops to the `blockhost` user.

All four engines ship this canonical unit.

### Package: `blockhost-engine-evm`

**Name:** `blockhost-engine-evm_<version>_all.deb`

**Dependencies:**
```
Depends: blockhost-common (>= 0.1.0), nodejs (>= 18), python3 (>= 3.10)
Provides: bhcrypt, blockhost-engine
Conflicts: blockhost-engine
```

> Virtual package pattern: every engine declares `Provides: blockhost-engine` and `Conflicts: blockhost-engine`. A package never conflicts with itself through a virtual package, so this prevents coinstallation without enumerating engine names.

### Installed File Locations

| Content | Destination |
|---------|-------------|
| Monitor (esbuild bundle) | `/usr/share/blockhost/monitor.js` |
| bw CLI (esbuild bundle) | `/usr/share/blockhost/bw.js` |
| bw wrapper | `/usr/bin/bw` |
| ab CLI (esbuild bundle) | `/usr/share/blockhost/ab.js` |
| ab wrapper | `/usr/bin/ab` |
| is CLI (esbuild bundle) | `/usr/share/blockhost/is.js` |
| is wrapper | `/usr/bin/is` |
| Contract deployer | `/usr/bin/blockhost-deploy-contracts` |
| NFT minter (Python) | `/usr/bin/blockhost-mint-nft` |
| NFT minter (module) | `/usr/lib/python3/dist-packages/blockhost/mint_nft.py` |
| Signup generator | `/usr/bin/blockhost-generate-signup` |
| Signup template | `/usr/share/blockhost/signup-template.html` |
| Wizard plugin | `/usr/lib/python3/dist-packages/blockhost/engine_<chain>/` |
| Crypto tool | `/usr/bin/bhcrypt` |
| NFT contract artifact | `/usr/share/blockhost/contracts/AccessCredentialNFT.json` |
| NFT contract source | `/usr/share/blockhost/contracts/AccessCredentialNFT.sol` |
| Contract source | `/opt/blockhost/contracts/BlockhostSubscriptions.sol` |
| Contract artifact | `/usr/share/blockhost/contracts/BlockhostSubscriptions.json` |
| Deploy scripts | `/opt/blockhost/scripts/deploy.ts`, `create-plan.ts` |
| Root agent action: wallet | `/usr/share/blockhost/root-agent-actions/wallet.py` |
| Systemd unit | `/lib/systemd/system/blockhost-monitor.service` |
| Env example | `/opt/blockhost/.env.example` |

### Engine-Provided Root Agent Actions

Engines must ship chain-specific root agent action plugins to `/usr/share/blockhost/root-agent-actions/`. The root agent (from common) loads these automatically via plugin discovery.

**Required action:**

| Action | Params | Returns | Purpose |
|--------|--------|---------|---------|
| `generate-wallet` | `{name: str}` | `{ok, address, keyfile}` | Generate chain-specific keypair, write keyfile, add to addressbook |

The handler must:
1. Validate name (short alphanumeric, reject reserved names: `admin`, `server`, `hot`, `dev`, `broker`)
2. Generate a private key (chain-specific derivation)
3. Write to `/etc/blockhost/<name>.key` (root:blockhost, 0640)
4. Derive the on-chain address from the private key
5. Add entry to `/etc/blockhost/addressbook.json` with `address` and `keyfile`

**Consumer:** Fund manager hot wallet auto-generation.

### Package Naming Convention (chain variants)

The EVM package is `blockhost-engine-evm`. All engine packages follow the `blockhost-engine-<chain>` naming convention (e.g., `blockhost-engine-opnet`). Engine packages must declare `Provides: blockhost-engine` and `Conflicts: blockhost-engine` — only one engine can be active per host. The virtual package pattern scales to any number of engines without updating existing control files.

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

## 10. Engine Wizard Plugin

The engine contributes a blockchain configuration page and finalization steps to the installer wizard, following the same plugin pattern as the provisioner (see `PROVISIONER_INTERFACE.md` §3). All four engines (EVM, OPNet, Cardano, Ergo) ship a wizard plugin under `/usr/lib/python3/dist-packages/blockhost/engine_<chain>/`.

### Background

The installer was originally built around hardcoded EVM finalization steps (`_finalize_keypair`, `_finalize_wallet`, `_finalize_contracts`, `_finalize_config` chain fields, `_finalize_mint_nft`, `_create_default_plan`). All chain-specific work has been moved into engine wizard plugins; the installer is now chain-agnostic.

### Engine Manifest Schema

**Installed to:** `/usr/share/blockhost/engine.json`

```json
{
  "name": "evm",
  "version": "0.1.0",
  "display_name": "EVM (Ethereum/Polygon)",

  "setup": {
    "first_boot_hook": "/usr/share/blockhost/engine-hooks/first-boot.sh",
    "wizard_module": "blockhost.engine_evm.wizard",
    "finalization_steps": ["wallet", "contracts", "chain_config"],
    "post_finalization_steps": ["mint_nft", "plan", "revenue_share"]
  },

  "config_keys": {
    "session_key": "blockchain"
  },

  "constraints": { "..." }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Machine-readable ID. |
| `version` | string | yes | Package version (semver). |
| `display_name` | string | yes | Shown in wizard UI. |
| `setup.first_boot_hook` | string | no | Absolute path to first-boot hook script. Called after engine .deb and Node.js are installed. Installs engine-specific host dependencies (e.g. Foundry for EVM). Exit 0 = success, non-zero = fatal. Inherits `STATE_DIR` and `LOG_FILE` env vars. |
| `setup.wizard_module` | string | yes | Python module path for dynamic import. |
| `setup.finalization_steps` | list | yes | Ordered step IDs run during wizard finalization. |
| `setup.post_finalization_steps` | list | yes | Steps run after main finalization (mint, plan, etc). |
| `config_keys.session_key` | string | yes | Flask session key where wizard stores engine config. |

### Consumers

| Consumer | File | What it reads |
|----------|------|---------------|
| First-boot | `scripts/first-boot.sh` | `setup.first_boot_hook` — runs it after engine install. Absent = skip. |
| Installer/wizard | `installer/web/app.py` | `setup.wizard_module`, `setup.finalization_steps`, `config_keys.session_key`, `display_name`, `constraints` |
| Admin panel | `admin/auth.py`, `admin/system.py` | `constraints` (format validation) |

### `constraints` (manifest key)

Chain-specific format patterns for input validation and UI rendering. Consumers (installer `app.py`, admin panel `auth.py`, `system.py`) load these at startup. No hardcoded format assumptions exist in consumers — all chain-specific validation comes from this key.

| Field | Type | Description | Example (EVM) |
|-------|------|-------------|---------------|
| `address_pattern` | regex | Valid wallet/contract address | `^0x[0-9a-fA-F]{40}$` |
| `signature_pattern` | regex | Valid signature format | `^0x[0-9a-fA-F]{130}$` |
| `native_token` | string | Native currency keyword for CLI | `eth` |
| `native_token_label` | string | Display label for native currency | `ETH` |
| `token_pattern` | regex | Valid token contract address | `^0x[0-9a-fA-F]{40}$` |
| `address_placeholder` | string | Input placeholder text | `0x...` |

All regex patterns must be anchored (`^...$`). Consumers compile with `re.compile()`.

If `constraints` is absent, consumers skip format validation and let CLIs (`bw`, `ab`, `is`) reject invalid input. The key is optional but strongly recommended — it enables pre-validation with friendly error messages.

**Consumers:**
- `admin/auth.py`: `address_pattern` (validate `bw who admin` output), `signature_pattern` (validate user-submitted signatures)
- `admin/system.py`: `address_pattern` (addressbook add), `token_pattern` + `native_token` (wallet send/withdraw)
- `admin/app.py` → templates: `native_token_label`, `address_placeholder` (UI labels and placeholders)
- `installer/web/app.py`: `validate_address()` from engine wizard module (separate mechanism, same purpose)

### Required Exports

Same pattern as provisioner wizard plugin:

| Export | Type | Signature |
|--------|------|-----------|
| `blueprint` | `flask.Blueprint` | Registers blockchain wizard route(s) |
| `get_finalization_steps()` | function | `-> list[tuple[str, str, callable]]` |
| `get_summary_data(session)` | function | `-> dict` |
| `get_summary_template()` | function | `-> str` |

### Optional Exports (Wallet Page)

| Export | Type | Signature | Purpose |
|--------|------|-----------|---------|
| `get_wallet_template()` | function | `-> Optional[str]` | Custom wallet connect template path (e.g. `"engine_evm/wallet.html"`) |
| `validate_signature(sig)` | function | `str -> bool` | Signature format validation (called by installer on wallet POST and config restore) |
| `decrypt_config(sig, ciphertext)` | function | `(str, str) -> dict` | Decrypt config backup file; returns parsed dict. Raise exception on failure |
| `encrypt_config(sig, plaintext)` | function | `(str, str) -> str` | Encrypt config for backup download; returns hex ciphertext string. Raise exception on failure |

If absent, the installer accepts any non-empty signature and falls back to `bhcrypt` for encrypt/decrypt operations.

### Optional Exports (Nginx)

| Export | Type | Signature | Purpose |
|--------|------|-----------|---------|
| `get_nginx_extra_locations(session)` | function | `(dict) -> str` | Extra nginx location blocks injected into the HTTPS server config. Called during nginx finalization. Return empty string if not needed. |

Used when the engine's signup page needs a server-side proxy to avoid CORS (e.g. Cardano engine proxies Koios API through nginx). The returned string is inserted verbatim inside the `server { }` block.

### Wallet Page: Deployer Top-Up

The engine's blockchain wizard page (typically `blockchain.html`) **must** include a top-up UI that lets the operator fund the deployer wallet directly from their browser wallet. This is shown on the same page where the deployer wallet is generated or imported — before contract deployment.

#### UI Requirements

| Element | Description |
|---------|-------------|
| Amount input | Numeric input for native currency amount (ETH, sats, ADA, ERG, etc.) |
| "Fund via wallet" button | Triggers browser wallet connection and transaction |
| Status display | Shows transaction progress: connecting → confirm in wallet → tx sent → confirmed |
| Balance display | Shows current deployer balance, updated after topup |

Both the "generate new wallet" and "import existing wallet" flows must have their own topup section.

#### Flow

1. Operator enters amount and clicks the fund button
2. Engine connects to the chain's browser wallet extension via its standard API:
   - EVM: `window.ethereum` (MetaMask) — `eth_requestAccounts` → `eth_sendTransaction`
   - OPNet: `window.opnet` (OPWallet) — `requestAccounts()` → `web3.sendBitcoin()`
   - Cardano: `window.cardano[wallet].enable()` (CIP-30) — `getUtxos()` → build tx → `signTx()` → `submitTx()`
   - Ergo: `window.ergo` (Nautilus) — EIP-12 connector
3. User confirms in wallet popup
4. Engine shows tx hash and polls for confirmation (receipt check, balance check, or both)
5. On confirmation, balance display refreshes

#### Confirmation Tracking

The engine must confirm the transaction before allowing the operator to proceed. Acceptable methods:
- Poll a tx-status backend endpoint (preferred)
- Poll the balance endpoint and detect increase
- Both (belt and suspenders — balance catch covers RPC edge cases)

Polling interval should match the chain's block time expectations. Display elapsed time and/or block info during the wait.

#### Balance Gating

The "Continue" button on the blockchain page **must** be disabled until the deployer wallet has sufficient funds for the next step (contract deployment). The minimum amount is engine-defined — it depends on deployment cost (gas, reference scripts, collateral, etc.).

#### Backend Endpoints (engine wizard plugin)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/blockchain/balance` | GET or POST | Query deployer balance via chain RPC/indexer. Returns balance in native units + `has_funds` boolean. |
| `/api/blockchain/tx-status` | GET | (Optional) Check transaction confirmation status. Returns `confirmed`/`pending`/`unknown`. |
| `/api/blockchain/block-info` | GET | (Optional) Current block height and age — useful for confirmation tracking UI. |
| `/api/blockchain/tip` | GET | (Optional) Current chain tip / slot — needed for TTL calculation (Cardano, Ergo). |

All endpoints that accept RPC URLs must validate them (http/https only, no private IPs — SSRF protection).

#### Error Handling

| Error | Display |
|-------|---------|
| No browser wallet detected | "No wallet detected — install [wallet name]" |
| User rejected transaction (code 4001) | "Transaction rejected by user." |
| Insufficient funds in wallet | Chain-specific error from wallet |
| Network/RPC error | Generic error with message |

After any error, the fund button must re-enable for retry.

### Wallet Page POST Contract

Engine wallet templates MUST POST to the `wizard_wallet` endpoint with three form fields:

| Field | Session key | Description |
|-------|-------------|-------------|
| `admin_wallet` | `admin_wallet` | Wallet address (format per engine's `validate_address()`) |
| `admin_signature` | `admin_signature` | Signature bytes (format engine-specific) |
| `public_secret` | `admin_public_secret` | The message that was signed |

**Downstream consumers** of these session values:
- `_gather_session_config()` — passes all three to finalization steps
- `_finalize_complete()` — writes `admin_signature` to `/etc/blockhost/admin-signature.key`
- Engine's `finalize_mint_nft()` — uses signature for NFT credential encryption (symmetric key derived from signature)
- `api_restore_config()` — uses signature for config backup decryption

### Config Backup Contract

**Decryption** (`decrypt_config`): Receives raw signature and ciphertext strings from the uploaded `.enc` file. Must return a parsed dict (YAML-decoded config). Raise an exception on failure — the installer catches it and returns a 400/500 to the client.

**Encryption** (`encrypt_config`): Receives the admin signature and YAML-serialized config plaintext. Must return a hex ciphertext string suitable for download as a `.enc` file. The ciphertext must be decryptable by `decrypt_config` with the same signature.

Without these exports, the installer falls back to shelling out to `bhcrypt encrypt-symmetric` / `decrypt-symmetric`.

### Crypto Tool Ownership

The engine ships its own crypto CLI tool (EVM: `bhcrypt`, a Python port of the former `pam_web3_tool`). The tool provides chain-specific crypto operations: keypair generation, ECIES encryption/decryption, symmetric encrypt/decrypt, and address derivation. Each engine ships the appropriate implementation for its chain's cryptographic primitives.

The `libpam-web3-tools` package is deprecated — its contents (crypto binary, NFT contract artifacts) are now engine-owned.

### What's in the Engine Wizard Plugin

| Element | Backed by |
|---------|-----------|
| Wallet template | Engine `templates/engine_<chain>/wallet.html` (optional override) |
| Blockchain wizard template | Engine `templates/engine_<chain>/blockchain.html` |
| Keypair generation | Engine `bhcrypt` CLI |
| Server wallet generation | `ab new server` (engine-owned) |
| Contract deployment | `blockhost-deploy-contracts` (engine-owned) |
| NFT minting + metadata | `blockhost-mint-nft` + `bw set encrypt` (engine-owned) |
| Default plan creation | `bw plan create` + `bw config stable` (engine-owned) |
| Signup page generation | `blockhost-generate-signup` (engine-owned) |

### What Stays in Installer

- Wizard framework (step bar, navigation, session management)
- Non-chain finalization steps (network, HTTPS, nginx, validation)
- OTP auth, provisioner plugin discovery
- `validate_system.py` (but chain-specific checks delegate to engine CLIs like `is contract`)

---

## 11. Auth Service & Signing Pages

Auth-svc, signing pages, and signature verification are **owned by libpam-web3 and its chain plugins** (`libpam-web3-<chain>`) — not by the engine.

The engine's role is limited to:
- Declaring which chain it uses (via `engine.json` → `name` field)
- The build system ensuring the matching `libpam-web3-<chain>` plugin is included in the VM template

See `libpam-web3` documentation for the plugin interface, `.sig` file format, and verification spec.

---

## 12. Known Issues & Abstraction Debt

### ~~`cast` used directly by installer and admin~~ (RESOLVED)

All `cast` calls in the installer and admin have been replaced by engine CLIs:

| `cast` call | Replaced by | Location |
|-------------|-------------|----------|
| `cast send --create` (contract deployment) | `blockhost-deploy-contracts [nft\|pos]` | `finalize.py` |
| `cast send createPlan` | `bw plan create <name> <price>` | `finalize.py` |
| `cast send setPrimaryStablecoin` | `bw config stable <address>` | `finalize.py` |
| `cast send updateUserEncrypted` | `bw set encrypt <nft_id> <data>` | `finalize.py` |
| `cast call balanceOf` (NFT) | `bw balance <contract>` | `finalize.py` |
| `cast call tokenOfOwnerByIndex` | Eliminated — ID is 0 (fresh deploy) or user-supplied | `finalize.py` |
| `cast wallet verify` | `bw who <message> <signature>` + engine `constraints` (signature validation) | `admin/auth.py` |
| `cast wallet address` | `ab new` (derives address internally) | `utils.py` |
| `cast call totalSupply` | `is contract <address>` | `validate_system.py` |

Chain-specific finalization steps live in the engine wizard plugin (§10). The installer is chain-agnostic.

### `mint_nft.py` dual install path (OPEN)

Installed as `/usr/bin/blockhost-mint-nft` (CLI). The monitor calls it as a CLI; the engine wizard finalization calls it as a CLI. The EVM engine also installs it as `/usr/lib/python3/dist-packages/blockhost/mint_nft.py` (importable module). The OPNet engine's mint script is TypeScript (`npx tsx`).

### Hardcoded chain configs in `chain-pools.ts` (OPEN)

Uniswap V2 router addresses, WETH addresses, and pair addresses are hardcoded per chain ID. Adding a new EVM chain requires code changes. Should be configurable (in `blockhost.yaml` or `web3-defaults.yaml`).

### ~~No BLOCK_TIME constant~~ (RESOLVED)

Monitor and reconciler are engine-internal processes. Each engine is built for a specific blockchain and knows its own block time — no cross-boundary constant needed. The engine sets whatever polling intervals make sense for its chain internally.

### ~~Signup page placeholders are EVM-specific~~ (RESOLVED)

Each engine ships its own signup template via `blockhost-generate-signup`. The template and its placeholders are engine-owned. See `PAGE_TEMPLATE_INTERFACE.md` for the per-chain placeholder set.

### ~~`installer/web/utils.py` generates secp256k1 keys directly~~ (RESOLVED)

Key generation lives in the engine wizard plugin finalization. The wizard UI calls `ab new server` when the user clicks "generate." The installer no longer imports `eth_keys`, `ecdsa`, or calls `cast wallet address` — all key operations go through engine CLIs.

### ~~`validate_system.py` uses `cast call` for contract queries~~ (RESOLVED)

Replaced by `is contract <address>` — a chain-agnostic contract liveness check. The script reads `engine.json` to determine which keys to validate.

### ~~`cast wallet verify` in admin/auth.py~~ (RESOLVED)

Replaced by `bw who <message> <signature>` for signature recovery + engine `constraints.signature_pattern` for input validation. The caller compares the returned address with `bw who admin` output. No direct `cast` dependency remains in auth.

### ~~Admin panel hardcoded EVM format assumptions~~ (RESOLVED)

All format validation in the admin panel (`auth.py`, `system.py`) now comes from engine manifest `constraints`. No hardcoded `0x` prefix checks, `len == 42`, or `len == 132` remain. If `constraints` is absent from the manifest, format validation is skipped and CLIs handle rejection.

---

## 13. Network Hook Integration

The engine handler (fund manager / subscription handler) calls the network hook after VM creation to resolve the subscriber-facing connection endpoint. The engine itself is network-mode-agnostic — it calls `get_connection_endpoint()` and receives an opaque host string.

### VM Creation Flow

```
1. If broker mode: broker-client allocation (existing)
2. provisioner.create(name, wallet, ...) → {ip, ipv6, vmid, username}
3. host = get_connection_endpoint(vm_name, result.ip, network_mode)
     → broker:  result.ipv6
     → manual:  static_ip from config
     → onion:   creates hidden service, pushes .onion to VM, returns .onion
4. Mint NFT (existing)
5. provisioner.guest-exec(name, "sed GECOS with NFT ID")
6. encrypt_connection_details({host, port: 22}) → subscriber
```

### VM Destroy Flow

```
1. provisioner.destroy(name)
2. network_hook.cleanup(name, network_mode)
3. Broker release if applicable
```

### Network Mode

The engine reads the current network mode from `/etc/blockhost/network-mode` (single line: `broker`, `manual`, or `onion`). Written by wizard finalization. If the file is absent, default to `broker` for backwards compatibility.

### Guest-Exec

The `update-gecos` provisioner command is superseded by the generic `guest-exec`:

```
blockhost-vm-guest-exec <name> "sed GECOS command..."
```

See `PROVISIONER_INTERFACE.md` §2 (`guest-exec`) for the full command spec.
