# NFT Contract Interface — AccessCredentialNFT

> Authoritative reference for the AccessCredentialNFT contract across chain implementations.
> Each BlockHost engine deploys a chain-native version. Consumers (monitor, reconciler,
> provisioner mint script, admin panel) program against this spec.
>
> | Chain | Model | Implementation |
> |-------|-------|----------------|
> | EVM | Account-model contract | Solidity, ERC-721 + ERC721Enumerable + Ownable (OpenZeppelin) |
> | OPNet | Account-model contract | OP_NET runtime, custom implementation (no inherited ERC721 base) |
> | Cardano | UTXO + minting policy | Aiken validator (`validators/nft.ak`); per-token reference data lives in subscription datum, not the NFT itself |
> | Ergo | UTXO + reference box | ErgoScript; per-token data lives in box register R8, updated by spending and recreating the box |
>
> Sections 2–8 below describe the EVM/OPNet account-model interface in detail. Section 9 covers the UTXO chains.

---

## 1. Purpose

One NFT per provisioned VM. The token ID is the canonical VM credential — ownership
proves access rights. The `userEncrypted` field stores ECIES-encrypted connection
details (IP, username) that only the server can decrypt.

**Auth-time usage:** None. PAM verifies signatures against GECOS-stored wallet addresses.
The reconciler syncs NFT ownership to GECOS asynchronously. No contract queries at login.

**Provision-time usage:** Engine reserves token ID, calls provisioner to create VM, then
calls `mint()` with encrypted connection details.

---

## 2. Core Functions (Account-Model Chains: EVM, OPNet)

Functions that both account-model implementations provide with equivalent semantics. UTXO chains satisfy the same external behavior through different mechanisms — see §9.

### Owner-Only (Write)

| Function | Purpose |
|----------|---------|
| `mint(to, userEncrypted)` | Mint credential NFT to recipient (2 params) |
| `updateUserEncrypted(tokenId, data)` | Update encrypted connection details |

### Public (Read-Only)

| Function | Returns | Purpose |
|----------|---------|---------|
| `ownerOf(tokenId)` | `address` | Token owner |
| `balanceOf(address)` | `uint256` | NFT count for wallet |
| `totalSupply()` | `uint256` | Total minted tokens (OPNet: minus burned) |
| `tokenOfOwnerByIndex(address, index)` | `uint256` | Token ID by owner index |
| `getUserEncrypted(tokenId)` | `bytes`/`string` | Encrypted connection details |

---

## 3. EVM-Specific

### Storage

```solidity
mapping(uint256 => bytes) private _userEncrypted;  // ECIES-encrypted connection details per token
```

No metadata fields. `publicSecret` is a host config value (`blockhost.yaml`), not stored per-token.

### Functions (EVM Only)

| Function | Modifier | Purpose |
|----------|----------|---------|
| `getUserEncrypted(tokenId)` | view | Returns `bytes` encrypted connection details |
| `tokenByIndex(index)` | view | Global enumeration (inherited ERC721Enumerable) |

### Inherited ERC721 (EVM)

Standard OpenZeppelin ERC721 + Enumerable + Ownable:
- `approve(to, tokenId)`, `setApprovalForAll(operator, approved)`
- `getApproved(tokenId)`, `isApprovedForAll(owner, operator)`
- `transferFrom(from, to, tokenId)`, `safeTransferFrom(from, to, tokenId)`
- `owner()`, `renounceOwnership()`, `transferOwnership(newOwner)`
- `supportsInterface(interfaceId)` — ERC165 (ERC721 + Enumerable)

### Events (EVM)

| Event | Fields | Emitted |
|-------|--------|---------|
| `CredentialMinted` | `indexed tokenId, indexed recipient` | On mint |
| `CredentialUpdated` | `indexed tokenId` | On updateUserEncrypted |
| `Transfer` | `indexed from, indexed to, indexed tokenId` | Inherited ERC721 (mint/burn/transfer) |
| `Approval` | `indexed owner, indexed approved, indexed tokenId` | On approve() |
| `ApprovalForAll` | `indexed owner, indexed operator, approved` | On setApprovalForAll() |

---

## 4. OPNet-Specific

### Storage

- `userEncrypted` per token (string) — ML-DSA encrypted connection details
- `_totalSupply` tracked separately from `_nextTokenId` (burn decrements supply, IDs never reused)
- Per-owner token arrays use `StoredU256Array` with the full 32-byte `Address` as sub-pointer (no truncation)

No `publicSecret`, `description`, `imageUri`, `animationUrlBase64`, or `expiresAt` fields.

### Functions (OPNet Only)

| Function | Access | Purpose |
|----------|--------|---------|
| `mint(to, userEncrypted)` | deployer | Mint with encrypted data only (2 params) |
| `getUserEncrypted(tokenId)` | view | Individual getter (replaces `getAccessData`) |
| `transfer(to, tokenId)` | owner/approved | Direct transfer |
| `transferFrom(from, to, tokenId)` | approved | Third-party transfer (verifies `from == actual owner`) |
| `approve(approved, tokenId)` | owner | Single-token approval |
| `setApprovalForAll(operator, approved)` | owner | Operator approval |
| `getApproved(tokenId)` | view | Check single-token approval |
| `isApprovedForAll(owner, operator)` | view | Check operator approval |
| `burn(tokenId)` | owner/approved | Destroy token (decrements totalSupply) |

### Events (OPNet)

| Event | Fields | Emitted |
|-------|--------|---------|
| `Transferred` | `from, to, tokenId` | On mint, transfer, and burn |

Single event covers all ownership changes. Mint: `from = address(0)`. Burn: `to = address(0)`.

---

## 5. EVM vs OPNet Differences

| Aspect | EVM | OPNet |
|--------|-----|-------|
| **mint signature** | `mint(address, bytes)` | `mint(address, string)` |
| **userEncrypted type** | `bytes` | `string` |
| **updateUserEncrypted type** | `(uint256, bytes)` | `(uint256, string)` |
| **getUserEncrypted return** | `bytes` | `string` |
| **Transfer** | Inherited ERC721 (`transferFrom`, `safeTransferFrom`) | Custom `transfer()` and `transferFrom()` |
| **Burn** | Not implemented (inherited ERC721 has no burn) | `burn(tokenId)` — owner or approved |
| **totalSupply** | Minted count (ERC721Enumerable) | Minted minus burned |
| **Transfer event** | Inherited `Transfer(from, to, tokenId)` | Custom `Transferred(from, to, tokenId)` |
| **Encryption** | ECIES (secp256k1) + AES-GCM | ML-DSA (post-quantum) |
| **Base** | OpenZeppelin ERC721 + Enumerable + Ownable | Custom OP_NET implementation |

---

## 6. Function Selectors

### EVM

```
mint(address,bytes)                                       → 0xb510391f
updateUserEncrypted(uint256,bytes)                        → 0x804a083e
getUserEncrypted(uint256)                                 → 0x191803d1
```

> Selectors are `keccak256(signature)[:4]`. Recompute with `cast sig "mint(address,bytes)"` if the contract's function signatures change.

Standard ERC721/Enumerable/Ownable selectors omitted — use OpenZeppelin reference.

### OPNet

```
mint(address,string)                                      → 0x066b1ee9
transfer(address,uint256)                                 → 0x3b88ef57
transferFrom(address,address,uint256)                     → 0x4b6685e7
approve(address,uint256)                                  → 0x9f0bb8a9
setApprovalForAll(address,bool)                           → 0xd97fb4c0
isApprovedForAll(address,address)                         → 0x67da1fb2
getApproved(uint256)                                      → 0xedc57d99
burn(uint256)                                             → 0x308dce5f
totalSupply()                                             → 0xa368022e
getUserEncrypted(uint256)                                 → 0x62e7bebe
updateUserEncrypted(uint256,string)                       → 0x724def55
balanceOf(address)                                        → 0x5b46f8f6
ownerOf(uint256)                                          → 0x06f6d69b
tokenOfOwnerByIndex(address,uint256)                      → 0x489f3960
```

#### Removed OPNet Selectors (previously existed)

```
mint(address,string,string)                               → 0xa89ce876
getPublicSecret(uint256)                                  → 0xfc49c0ea
getAccessData(uint256)                                    → 0x77c265de
```

---

## 7. Consumers

### Shared Consumers (Both Chains)

| Consumer | Operations | Notes |
|----------|-----------|-------|
| Engine (handler) | `mint()` | Called after VM creation. Passes `userEncrypted` with encrypted connection details |
| Engine (handler) | `totalSupply()` | Token ID reservation floor before provisioning |
| Engine (reconciler) | `ownerOf()`, `totalSupply()` | Periodic ownership verification, GECOS sync |
| Engine (`bw set encrypt`) | `updateUserEncrypted()` | Update connection details post-creation |
| Admin panel | `ownerOf()` via `bw who` | Admin NFT ownership check for auth |
| Installer validation | `totalSupply()` via `is contract` | Contract liveness check |

### OPNet-Specific Consumers

| Consumer | Operations | Notes |
|----------|-----------|-------|
| Auth-svc | None (chain queries removed) | ML-DSA signature verification is local. No chain interaction at auth time |
| Monitor (reconciler) | `Transferred` event, `ownerOf()` | Watches ownership transfers, syncs GECOS |
| Monitor | `getUserEncrypted()` | Read encrypted connection details |
| Engine | `transfer()`, `burn()` | VM lifecycle management (ownership transfer, credential revocation) |

---

## 8. Constructor

### EVM

```solidity
constructor(string name, string symbol)
```

Deploys with token name and symbol. Owner = `msg.sender`.

### OPNet

Constructor takes token name and symbol. Owner = deployer. No image URI support.

---

## 9. UTXO Chains: Cardano & Ergo

The account-model contract interface (§2) does not translate directly to UTXO chains. Each NFT is a token issued by a minting policy; per-token data lives in script datums or box registers, not in mutable contract storage. The CLI surface (`bw who`, `is`, `blockhost-mint-nft`, `bw set encrypt`) presents the same behavior — the underlying mechanism differs.

### Cardano (Aiken)

| Element | Where it lives |
|---------|----------------|
| NFT minting policy | `validators/nft.ak` — handles `MintNft` and `BurnNft` redeemers only |
| Token name | Per-mint identifier (typically VM token ID) |
| Owner / `ownerOf` equivalent | The wallet currently holding the asset (queried via Koios `address_assets` / asset history) |
| `userEncrypted` equivalent | Field on the **subscription validator datum** (`SubscriptionDatum.userEncrypted`), not on the NFT itself. Lives in `lib/blockhost/types.ak` |
| `updateUserEncrypted` equivalent | None at the NFT layer. The subscription datum can be updated by spending and re-creating the subscription UTXO via the subscription validator's `Update` redeemer. The NFT metadata is fixed at mint time |

The CIP-68 reference token / user token pattern would let metadata live alongside the NFT, but the current implementation does not use it — encrypted connection details ride on the subscription datum and the NFT is a bare credential token.

**Consumers querying the chain:** Use Koios `asset_addresses(policy_id, asset_name)` to find current owner and `tx_metadata` plus subscription validator UTXO scans to find the encrypted blob.

### Ergo (ErgoScript)

| Element | Where it lives |
|---------|----------------|
| NFT issuance | UTXO with token quantity 1 and unique tokenId (= the box's first input id) |
| Token reference | Stored in a **reference box** locked at the NFT contract, addressed by the NFT's tokenId |
| Owner equivalent | Wallet holding the asset (queried via node `/blockchain/box/byTokenId` etc.) |
| `userEncrypted` equivalent | Reference box register R8 |
| `updateUserEncrypted` equivalent | Spend the reference box and recreate it with new R8 contents (no callable contract method) |

Updates happen by transaction: spend the existing reference box, build a new box at the same script address with the same tokenId in `tokens[]` but new R8 bytes. The NFT contract enforces continuity rules.

### Common Behavior Across UTXO Chains

- `bw who <token_id>` resolves the current holder via the chain's indexer.
- `blockhost-mint-nft --owner-wallet … --user-encrypted …` builds and submits a mint transaction; on Cardano this also creates a subscription UTXO carrying `userEncrypted`.
- `bw set encrypt <token_id> <data>`: on Ergo, builds a reference-box update tx; on Cardano, builds a subscription-datum update tx (Update redeemer).
- Reconciler logic still uses ownership checks, but reads from the indexer rather than `ownerOf()`.
