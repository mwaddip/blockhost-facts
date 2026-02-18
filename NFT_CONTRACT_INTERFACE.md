# NFT Contract Interface — AccessCredentialNFT

> Authoritative reference for the AccessCredentialNFT contract across chain implementations.
> Each BlockHost engine deploys a chain-native version. Consumers (monitor, reconciler,
> provisioner mint script, admin panel) program against this spec.
>
> **EVM:** Solidity, ERC-721 + ERC721Enumerable + Ownable (OpenZeppelin).
> **OPNet:** OP_NET runtime, custom implementation (no inherited ERC721 base).

---

## 1. Purpose

One NFT per provisioned VM. The token ID is the canonical VM credential — ownership
proves access rights. The `userEncrypted` field stores ECIES-encrypted connection
details (IP, username) that only the server can decrypt.

**Auth-time usage:** None. PAM verifies signatures against GECOS-stored wallet addresses.
The reconciler syncs NFT ownership to GECOS asynchronously. No contract queries at login.

**Provision-time usage:** Monitor calls `mint()` after VM creation. Provisioner reserves
token ID, monitor mints with encrypted connection details.

---

## 2. Core Functions (Both Chains)

Functions that both implementations provide with equivalent semantics.

### Owner-Only (Write)

| Function | Purpose |
|----------|---------|
| `mint(...)` | Mint credential NFT to recipient (signature differs — see §5) |
| `updateUserEncrypted(tokenId, data)` | Update encrypted connection details |

### Public (Read-Only)

| Function | Returns | Purpose |
|----------|---------|---------|
| `ownerOf(tokenId)` | `address` | Token owner |
| `balanceOf(address)` | `uint256` | NFT count for wallet |
| `totalSupply()` | `uint256` | Total minted tokens (OPNet: minus burned) |
| `tokenOfOwnerByIndex(address, index)` | `uint256` | Token ID by owner index |

---

## 3. EVM-Specific

### Storage (AccessData Struct)

```solidity
struct AccessData {
    bytes userEncrypted;           // ECIES-encrypted connection details (AES-GCM)
    string publicSecret;           // Key derivation message: "libpam-web3:<address>:<nonce>"
    string description;            // Human-readable (e.g., "Production Server Access")
    string imageUri;               // IPFS or HTTPS image (empty → default)
    string animationUrlBase64;     // Signing page HTML (raw base64, NOT data URI)
    uint256 issuedAt;              // Block timestamp at mint
    uint256 expiresAt;             // Expiration (0 = never)
}
```

### Functions (EVM Only)

| Function | Modifier | Purpose |
|----------|----------|---------|
| `mint(to, userEncrypted, publicSecret, description, imageUri, animationUrlBase64, expiresAt)` | onlyOwner | Full mint with metadata (7 params) |
| `mintBatch(recipients[], ...)` | onlyOwner | Batch mint (all arrays must equal length) |
| `updateExpiration(tokenId, newExpiration)` | onlyOwner | Extend credential validity |
| `updateAnimationUrl(tokenId, base64)` | onlyOwner | Update per-token signing page |
| `setDefaultImageUri(uri)` | onlyOwner | Contract-wide default image |
| `getAccessData(tokenId)` | view | Returns `(userEncrypted, publicSecret, issuedAt, expiresAt)` |
| `getPublicSecret(tokenId)` | view | Returns `publicSecret` string |
| `isExpired(tokenId)` | view | `true` if `expiresAt != 0 && now > expiresAt` |
| `tokenURI(tokenId)` | view | On-chain JSON metadata (base64 data URI) |
| `tokenByIndex(index)` | view | Global enumeration |

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
| `CredentialMinted` | `indexed tokenId, indexed recipient, issuedAt, expiresAt` | On mint |
| `CredentialUpdated` | `indexed tokenId` | On updateUserEncrypted/Expiration/AnimationUrl |
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
| **mint signature** | `mint(address, bytes, string, string, string, string, uint256)` — 7 params | `mint(address, string)` — 2 params |
| **mint selector** | `0xd204c45e` | `0x066b1ee9` |
| **publicSecret** | Stored per token, returned by `getAccessData()` and `getPublicSecret()` | Removed — ML-DSA replaces sign-publicSecret key derivation |
| **getAccessData** | Returns `(userEncrypted, publicSecret, issuedAt, expiresAt)` | Removed — use `getUserEncrypted()` |
| **getUserEncrypted** | Not present (use `getAccessData`) | Returns `string` for single token |
| **updateUserEncrypted type** | `(uint256, bytes)` — binary data | `(uint256, string)` — string data |
| **Expiration** | Per-token `expiresAt`, `isExpired()`, `updateExpiration()` | Not implemented |
| **Metadata** | `description`, `imageUri`, `animationUrlBase64`, `tokenURI()` | Not implemented |
| **Batch mint** | `mintBatch()` | Not implemented |
| **Transfer** | Inherited ERC721 (`transferFrom`, `safeTransferFrom`) | Custom `transfer()` and `transferFrom()` |
| **Burn** | Not implemented (inherited ERC721 has no burn) | `burn(tokenId)` — owner or approved |
| **totalSupply** | Minted count (ERC721Enumerable) | Minted minus burned |
| **Transfer event** | Inherited `Transfer(from, to, tokenId)` | Custom `Transferred(from, to, tokenId)` |
| **Encryption** | ECIES (secp256k1) + AES-GCM via publicSecret key derivation | ML-DSA (post-quantum) |
| **Base** | OpenZeppelin ERC721 + Enumerable + Ownable | Custom OP_NET implementation |

---

## 6. Function Selectors

### EVM

```
mint(address,bytes,string,string,string,string,uint256)  → 0xd204c45e
mintBatch(...)                                            → 0xb71d3f55
updateUserEncrypted(uint256,bytes)                        → 0x73f2ccc8
updateExpiration(uint256,uint256)                         → 0x3a2a3e1f
updateAnimationUrl(uint256,string)                        → 0x40f3b0e1
setDefaultImageUri(string)                                → 0xb13fbeb7
getAccessData(uint256)                                    → 0xbe3e7b20
getPublicSecret(uint256)                                  → 0xfc49c0ea
isExpired(uint256)                                        → 0x2efb2c55
tokenURI(uint256)                                         → 0xc87b56dd
```

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
| Monitor (mint handler) | `mint()` | Called after VM creation. Passes `userEncrypted` with ECIES-encrypted connection details |
| Monitor (reconciler) | `ownerOf()`, `totalSupply()` | Periodic ownership verification, GECOS sync |
| Provisioner (mint script) | `mint()` | Via `blockhost-mint-nft` CLI or Python import |
| Admin panel | `ownerOf()` via `bw who` | Admin NFT ownership check for auth |
| Installer validation | `totalSupply()` via `is contract` | Contract liveness check |

### EVM-Specific Consumers

| Consumer | Operations | Notes |
|----------|-----------|-------|
| Monitor (mint handler) | `getAccessData()` | Reads back publicSecret after mint for verification |
| Signup page (JS) | `tokenURI()` | Display credential metadata in wallet/marketplace |
| Monitor (reconciler) | `getPublicSecret()` | Key derivation path verification |
| Engine finalization | `updateUserEncrypted()`, `updateExpiration()` | Post-setup metadata updates |

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
constructor(string name, string symbol, string defaultImageUri)
```

Deploys with token name, symbol, and default image URI. Owner = `msg.sender`.

### OPNet

Constructor takes token name and symbol. Owner = deployer. No image URI support.
