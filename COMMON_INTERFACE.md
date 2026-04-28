# blockhost-common Interface Specification

> Authoritative contract for everything blockhost-common exposes to other packages.
> Derived from source code audit (2026-02-09). When code and this doc disagree, investigate both.

---

## 1. Configuration API

Module: `blockhost.config`

### Path Constants

```python
CONFIG_DIR = Path("/etc/blockhost")
DATA_DIR   = Path("/var/lib/blockhost")
```

### Functions

| Function | Signature | Returns | Reads |
|----------|-----------|---------|-------|
| `load_config` | `(filename, fallback_dir=None)` | `dict` | `CONFIG_DIR/{filename}` |
| `load_db_config` | `(fallback_dir=None)` | `dict` | `db.yaml` (see schema below) |
| `load_web3_config` | `(fallback_dir=None)` | `dict` | `web3-defaults.yaml` (see schema below) |
| `load_blockhost_config` | `(fallback_dir=None)` | `dict` | `blockhost.yaml` (see schema below) |
| `load_broker_allocation` | `(fallback_dir=None)` | `Optional[dict]` | `broker-allocation.json` (returns None if missing) |
| `get_config_path` | `(filename, fallback_dir=None)` | `Path` | Search order: CONFIG_DIR, fallback_dir, `./config/` |
| `get_db_file_path` | `(fallback_dir=None)` | `Path` | Derives from `db.yaml` → `db_file` key |
| `is_development_mode` | `()` | `bool` | `BLOCKHOST_DEV` env var or CONFIG_DIR missing |

**Search order**: `/etc/blockhost/{filename}` → `fallback_dir/{filename}` → `./config/{filename}`. Raises `FileNotFoundError` if all fail.

**`fallback_dir`**: Every load function accepts this. Intended for development and testing — lets scripts run without `/etc/blockhost/` existing.

### Contract Violations (existing)

| Export | Problem | Resolution |
|--------|---------|------------|
| `get_terraform_dir()` | Terraform is Proxmox-specific. Common shouldn't know about it. | Move to provisioner-proxmox or make generic ("provisioner working dir") |
| `TERRAFORM_DIR` constant | Same — hardcoded `/var/lib/blockhost/terraform` | Remove from common |

---

## 2. VM Database API

Module: `blockhost.vm_db`

### Factory

```python
get_database(use_mock=False, config_path=None, fallback_dir=None) -> VMDatabaseBase
```

Returns `MockVMDatabase` if `use_mock=True`, otherwise `VMDatabase`.

### Public Methods (all implementations)

#### VM Lifecycle

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `register_vm` | `(name, vmid, ip, ipv6=None, owner="", expiry_days=30, purpose="", wallet_address=None, username=None)` | `dict` (VM record) | **Provisioner-owned.** Creates new record. Rejects duplicate `name` regardless of status — call `delete_vm` first to clobber a destroyed record. The provisioner calls this from `vm-create` before printing its result line; engines do NOT call it. |
| `get_vm` | `(name)` | `Optional[dict]` | Lookup by name |
| `list_vms` | `(status=None)` | `list[dict]` | Filter: `"active"`, `"suspended"`, `"destroyed"`, or `None` for all |
| `extend_expiry` | `(name, days)` | `None` | Engine-owned. Extends from current expiry. |
| `mark_suspended` | `(name)` | `None` | Sets status + suspended_at |
| `mark_active` | `(name, new_expiry=None)` | `None` | Reactivates, optionally sets new expiry |
| `mark_destroyed` | `(name)` | `None` | **Provisioner-owned.** Sets status + destroyed_at, releases IPs. The provisioner calls this from `vm-destroy` after the VM is gone; engines do NOT call it. |
| `delete_vm` | `(name)` | `None` | **Provisioner-owned.** Permanently removes the record. Releases IPv4/IPv6 allocations. Raises `ValueError` if not found. Used for destroyed-name reuse — provisioners call this before re-registering a name that previously belonged to a destroyed VM. |

#### Allocation

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `allocate_ip` | `()` | `Optional[str]` | Next available IPv4 from pool |
| `allocate_ipv6` | `()` | `Optional[str]` | Next available IPv6 (requires broker allocation) |
| `allocate_vmid` | `()` | `int` | Next available VMID. Raises `RuntimeError` if `vmid_range` not configured |
| `release_ip` | `(ip)` | `None` | VMDatabase only (not on mock) |
| `release_ipv6` | `(ipv6)` | `None` | VMDatabase only (not on mock) |

**VMID semantics for non-numeric provisioners:** Provisioners without a numeric VMID (libvirt) call `register_vm` with `vmid=0`. The natural key for those VMs is `vm_name`. Code reading `vm['vmid']` from the DB should fall through to `vm['vm_name']` when it sees `0`.

#### Garbage Collection

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `get_vms_to_suspend` | `()` | `list[dict]` | Active VMs past expiry |
| `get_vms_to_destroy` | `(grace_days)` | `list[dict]` | Suspended VMs past grace period |
| `get_expired_vms` | `(grace_days=0)` | `list[dict]` | All VMs past expiry + grace |

#### Chain State Recording

Mutators that record on-chain provisioning state onto the local VM record. Routed through `_atomic_update` so they serialise with all other mutators on the same lockfile.

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `set_nft_minted` | `(vm_name, token_id)` | `None` | Records the minted NFT on the VM record. Sets `nft_token_id`, `nft_minted = True`, `nft_minted_at`. Raises `ValueError` if VM not found. |
| `update_beacon_info` | `(vm_name, beacon_name, utxo_ref)` | `bool` | Sets `beacon_name` and `utxo_ref` on an existing VM record. Returns `False` if `vm_name` not found (no-op, no exception); `True` on success. |

**Note on `update_beacon_info` return convention.** Every other mutator on `VMDatabaseBase` raises `ValueError` when the target VM is missing. `update_beacon_info` is the deliberate exception: callers invoke it after a chain commit, and a concurrent VM deletion shouldn't blow up the post-commit handler with an exception that's not actionable. Returning `False` lets the caller race a deletion safely. Future `update_*_info`-style mutators that face the same race should follow this convention.

**Note on field naming.** `beacon_name` and `utxo_ref` are Cardano-specific terminology — beacons are Cardano's chain-side provisioning markers, UTxO refs are Cardano transaction outputs. The method and fields are nominally chain-specific; other chains writing analogous state may grow their own fields in the same `vms[vm_name]` dict (no schema migration), or this method may evolve into a generic `update_chain_state(vm_name, **fields)` form if multiple chains start needing similar writes. For now, the Cardano-specific shape is acknowledged.

There is no separate NFT reservation lifecycle. The engine handler mints the NFT (the chain assigns the token ID), reads the actual token ID from `blockhost-mint-nft` stdout, then calls `set_nft_minted()` to record it. If anything fails between mint and recording, the reconciler picks it up by querying `ownerOf(tokenId)` on-chain and matching back to the VM. Local pre-mint reservation is intentionally absent.

### VM Record Schema

```python
{
    "vm_name": str,
    "vmid": int,
    "ip_address": str,
    "ipv6_address": Optional[str],
    "status": "active" | "suspended" | "destroyed",
    "owner": str,
    "wallet_address": Optional[str],
    "username": Optional[str],       # Linux username provisioned on the VM (set by register_vm)
    "purpose": str,
    "created_at": str,       # ISO 8601
    "expires_at": str,       # ISO 8601
    "suspended_at": Optional[str],   # Added on suspend
    "destroyed_at": Optional[str],   # Added on destroy
    "nft_token_id": Optional[int],     # Added by set_nft_minted()
    "nft_minted": Optional[bool],      # Added by set_nft_minted()
    "nft_minted_at": Optional[str],    # Added by set_nft_minted() — ISO 8601
    "beacon_name": Optional[str],      # Added by update_beacon_info() — chain-defined; today: Cardano provisioning beacon
    "utxo_ref": Optional[str],         # Added by update_beacon_info() — chain-defined; today: Cardano UTxO reference
    "gecos_synced": Optional[bool],    # Set by reconciler when GECOS update succeeds
}
```

NFT state is stored inline on the VM record. There is no top-level `reserved_nft_tokens` map.

### Locking Model

Production (`VMDatabase`) uses a separate lockfile at `{db_file}.lock` to avoid the truncation-before-lock problem (`open("w")` truncates before `fcntl.flock()` runs on the same fd).

| Method | Lock | Purpose |
|--------|------|---------|
| `_atomic_update(mutator)` | Exclusive on lockfile | All mutations. Acquires lock → reads DB → calls `mutator(db_dict)` → writes via temp+rename → releases lock. Single critical section eliminates TOCTOU. |
| `_read_db()` | Shared on DB file | Read-only access for non-mutating methods (`get_vm`, `list_vms`, `get_expired_vms`, etc.) |
| `_read_db_unlocked()` | None (internal) | Called within `_atomic_update` only — lock already held |
| `_write_db_unlocked()` | None (internal) | Atomic write via temp file + rename. Called within `_atomic_update` only. |

`_atomic_update` is abstract on `VMDatabaseBase`. `VMDatabase` implements it with `fcntl.LOCK_EX` on the lockfile. `MockVMDatabase` implements it as a passthrough (read → mutate → write, no locking).

All mutating methods in the base class use `_atomic_update`: `register_vm`, `mark_suspended`, `mark_active`, `mark_destroyed`, `allocate_ip`, `allocate_ipv6`, `allocate_vmid`, `extend_expiry`, `set_nft_minted`, `update_beacon_info`, `delete_vm`. Plus `release_ip` and `release_ipv6` on `VMDatabase`.

### Storage

- Production: JSON file at path from `db.yaml` → `db_file`, lockfile at `{db_file}.lock`
- Default: `/var/lib/blockhost/vms.json` (lockfile: `vms.json.lock`)
- Mock: in-memory, no file locking

---

## 3. Root Agent Client API

Module: `blockhost.root_agent`

### Protocol

- **Socket**: `/run/blockhost/root-agent.sock`
- **Framing**: 4-byte big-endian length prefix + JSON payload (both directions)
- **Request**: `{"action": "action-name", "params": {...}}`
- **Response**: `{"ok": true/false, "error": "...", ...}`

### Generic Call

```python
call(action: str, timeout: float = 300, **params) -> dict
```

This is the real interface. Everything else is a wrapper around this.

### Convenience Wrappers

| Function | Calls Action | Params |
|----------|-------------|--------|
| `ip6_route_add(address, dev)` | `ip6-route-add` | `{address, dev}` |
| `ip6_route_del(address, dev)` | `ip6-route-del` | `{address, dev}` |
| `addressbook_save(entries)` | `addressbook-save` | `{entries}` |

`dev` is required — there is no default. (Earlier versions defaulted to `"vmbr0"`; that Proxmox-ism was removed.)

### Exceptions

- `RootAgentError` — base exception, agent returned `{"ok": false}`
- `RootAgentConnectionError(RootAgentError)` — socket unreachable

---

## 4. Root Agent Daemon

Location: `/usr/share/blockhost/root-agent/blockhost_root_agent.py`

### Plugin Discovery

- Scans `/usr/share/blockhost/root-agent-actions/` for `.py` files
- Skips files starting with `_` (e.g., `_common.py`)
- Each module must export `ACTIONS: dict[str, callable]`
- Handler signature: `handler(params: dict) -> dict`
- Response must include `"ok": bool`

### Built-in Actions (from common)

Shipped in `root-agent-actions/system.py` and `networking.py`:

| Action | Params | Returns | Module |
|--------|--------|---------|--------|
| `iptables-open` | `{port: int, proto: str, comment: str}` | `{ok, output}` | system.py |
| `iptables-close` | `{port: int, proto: str, comment: str}` | `{ok, output}` | system.py |
| `virt-customize` | `{image_path: str, commands: list[list]}` | `{ok, output}` | system.py |
| `addressbook-save` | `{entries: dict}` | `{ok}` | system.py |
| `broker-renew` | `{}` (none) | `{ok, output}` | system.py |
| `ip6-route-add` | `{address: str, dev: str}` | `{ok, output}` | networking.py |
| `ip6-route-del` | `{address: str, dev: str}` | `{ok, output}` | networking.py |
| `bridge-port-isolate` | `{dev: str}` | `{ok, output}` | networking.py |

### Shared Utilities (`_common.py`)

Available to all action plugins via `from _common import ...`:

**Constants:**
```python
CONFIG_DIR = Path('/etc/blockhost')
STATE_DIR = Path('/var/lib/blockhost')
VMID_MIN = 100
VMID_MAX = 999999
```

**Validation Regexes:**
```python
NAME_RE             = re.compile(r'^[a-z0-9-]{1,64}$')
SHORT_NAME_RE       = re.compile(r'^[a-z0-9-]{1,32}$')
STORAGE_RE          = re.compile(r'^[a-z0-9-]+$')
_HEX_ADDRESS_RE     = re.compile(r'^0x[0-9a-fA-F]{40,128}$')   # internal — use is_valid_address()
_BECH32_ADDRESS_RE  = re.compile(r'^[a-z][a-z0-9_]{0,9}1[02-9ac-hj-np-z]{39,98}$')   # internal
COMMENT_RE          = re.compile(r'^[a-zA-Z0-9-]+$')
IPV6_CIDR128_RE     = re.compile(r'^([0-9a-fA-F:]+)/128$')
TAP_DEV_RE          = re.compile(r'^tap\d+i\d+$')
```

The two address regexes are internal (leading underscore). Plugins should call `is_valid_address(addr)` instead — it handles both hex (EVM/OPNet, with the wider 40–128 char range to cover non-EVM hex addresses) and Bech32 (Cardano `addr1...`/`addr_test1...`, Ergo testnet, etc.).

**Validation Functions:**
- `is_valid_address(addr: str) -> bool` — chain-agnostic structural check (hex OR Bech32). Use this from plugins; do not import `_HEX_ADDRESS_RE` / `_BECH32_ADDRESS_RE` directly.
- `validate_vmid(vmid: int) -> int`
- `validate_ipv6_128(address: str) -> str`
- `validate_dev(dev: str) -> str` — accepts entries from `ALLOWED_ROUTE_DEVS` **or** any device matching `TAP_DEV_RE` (libvirt's per-VM tap interfaces, e.g. `tap0i0`, `tap1i2`)

**Execution:**
- `run(cmd: list, timeout: int = 120) -> tuple[int, str, str]` — returns `(returncode, stdout, stderr)`

**Allowed Sets:**
- `ALLOWED_ROUTE_DEVS = frozenset({'vmbr0', 'virbr0', 'br0', 'br-ext', 'docker0'})`
- `WALLET_DENY_NAMES = frozenset({'admin', 'server', 'dev', 'broker'})`
- `VIRT_CUSTOMIZE_ALLOWED_OPS` — validated operations for virt-customize

---

## 5. Cloud-Init API

Module: `blockhost.cloud_init`

### Functions

| Function | Signature | Returns |
|----------|-----------|---------|
| `render_cloud_init` | `(template_name: str, variables: dict[str, str], extra_dirs: list[Path] = None)` | `str` (rendered YAML) |
| `find_template` | `(name: str, extra_dirs: list[Path] = None)` | `Path` |
| `list_templates` | `(extra_dirs: list[Path] = None)` | `list[str]` |

**Template search order**: `extra_dirs` (first match) → `/usr/share/blockhost/cloud-init/templates/` → `./cloud-init/templates/` (dev)

**Rendering**: `string.Template.safe_substitute()` — unknown `${VAR}` left as-is (no error).

### Shipped Templates

| Template | Variables Required | Purpose |
|----------|-------------------|---------|
| `nft-auth.yaml` | `VM_NAME`, `SIGNING_HOST`, `SIGNING_DOMAIN`, `USERNAME`, `WALLET_ADDRESS`, `NFT_TOKEN_ID`, `OTP_LENGTH`, `OTP_TTL`, `SECRET_KEY` | NFT-authenticated VM with PAM module. Requires both `libpam-web3` (PAM) and engine auth-svc template package on VM. |
| `webserver.yaml` | (none) | nginx + UFW |
| `devbox.yaml` | (none) | Build tools + dev environment |

---

## 5a. Naming Validators

Module: `blockhost.naming`

Shared validators for identifiers used as natural keys across BlockHost. Centralised so that tightening the rules (e.g. removing dots) only touches one site, instead of the 9+ places where the same regex used to be duplicated across libvirt and Proxmox provisioners.

### Constants

| Name | Value | Purpose |
|------|-------|---------|
| `DOMAIN_NAME_RE` | `re.compile(r'^[a-zA-Z0-9][a-zA-Z0-9._-]{0,63}$')` | VM domain name. Used as the natural key in `vms.json` and as the libvirt domain name. |

### Functions

| Function | Signature | Returns | Raises |
|----------|-----------|---------|--------|
| `is_valid_domain_name` | `(name: str) -> bool` | `True` if `name` matches `DOMAIN_NAME_RE`, else `False`. Returns `False` for non-string input or empty string. | — |
| `validate_domain_name` | `(name: str) -> str` | `name` if valid; intended for use at trust boundaries (root agent action handlers, CLI entry points). | `ValueError(f'Invalid domain name: {name!r}')` if invalid. |

### Why this regex?

- **Leading char must be alphanumeric** — bans hostnames starting with `.` or `-` (which break libvirt domain naming and DNS).
- **Allows `.`, `_`, `-`** in the body — matches both libvirt domain conventions and the historical Proxmox hostname rules.
- **64-char ceiling** — Linux `HOST_NAME_MAX` is 64; libvirt domain names are bounded by the same practical limit.

### Re-exports

Available from the package root: `from blockhost import DOMAIN_NAME_RE, is_valid_domain_name, validate_domain_name`.

### Cross-language

Bash CLI wrappers (5 libvirt scripts + Proxmox equivalents) keep an inline regex check matching `DOMAIN_NAME_RE`. Cross-language sharing via a CLI helper was considered and rejected as overkill — bash callers are simple `[[ "$name" =~ ^... ]]` checks at script entry, and the duplication risk is bounded (one regex, two implementations of the same string).

---

## 6. Provisioner Dispatcher

Module: `blockhost.provisioner`

### Factory

```python
get_provisioner() -> ProvisionerDispatcher  # singleton
```

### Class: ProvisionerDispatcher

**Constructor**: `__init__(manifest_path: Path = None)` — defaults to `/usr/share/blockhost/provisioner.json`

**Properties:**

| Property | Type | Source |
|----------|------|--------|
| `name` | `str` | `manifest["name"]` |
| `display_name` | `str` | `manifest["display_name"]` |
| `version` | `str` | `manifest["version"]` |
| `is_loaded` | `bool` | True if manifest exists and parsed |
| `manifest` | `dict` | Raw manifest (empty dict if not loaded) |
| `wizard_module` | `Optional[str]` | `manifest["setup"]["wizard_module"]` |
| `finalization_steps` | `list[str]` | `manifest["setup"]["finalization_steps"]` |
| `first_boot_hook` | `Optional[str]` | `manifest["setup"]["first_boot_hook"]` |
| `session_key` | `str` | `manifest["config_keys"]["session_key"]` |
| `root_agent_actions` | `Optional[str]` | `manifest["root_agent_actions"]` |

**Methods:**

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `get_command` | `(verb: str)` | `str` | Maps verb to CLI command name from manifest |
| `run` | `(verb: str, args: list = None, **kwargs)` | `CompletedProcess` | Runs command via `subprocess.run()` |

**Legacy fallback**: If no manifest exists, uses hardcoded `LEGACY_COMMANDS` dict. This should be removed (see section 9).

---

## 6a. CLI tools

Engine-helper binaries shipped under `/usr/bin/`. Engines call these instead of `python3 -c "<inline>"` to avoid per-call interpreter startup + import cost (~50–150 ms per spawn, 2–4× per subscription event in the worst case).

### Conventions

- `0` = success
- `1` = expected failure (record not found, validation error)
- `2` = unexpected exception
- stderr = human-readable error message
- stdout = structured payload (JSON, host string, etc.)

### `blockhost-vmdb`

Wraps `VMDatabaseBase` methods that engines need at provisioning time.

| Subcommand | Args | Stdout | Exit | Wraps |
|------------|------|--------|------|-------|
| `get-vm` | `<vm_name>` | JSON-encoded VM record | 0 ok / 1 not found | `VMDatabaseBase.get_vm` |
| `mark-nft-minted` | `<vm_name> <token_id>` | (empty) | 0 ok / 1 not found | `VMDatabaseBase.set_nft_minted` |
| `extend-expiry` | `<vm_name> <days>` | line 1: confirmation; line 2: `NEEDS_RESUME` (only if VM was suspended at extend time) | 0 ok / 1 not found | `VMDatabaseBase.extend_expiry` (with status check around it) |

**Subcommands intentionally absent.** `register-vm` and `mark-destroyed` are not exposed via this CLI. They are provisioner-owned (see `§2 VM Lifecycle`) — the provisioner calls them directly via `from blockhost.vm_db import get_database`, since provisioners are Python and don't need a CLI bridge. Adding a `register-vm` subcommand would re-introduce the engine→common bridge that this CLI was meant to eliminate.

**Note on `mark-nft-minted` signature.** Some earlier engine prompts suggested `mark-nft-minted <token_id> <owner_wallet>`. That doesn't match the underlying API (`set_nft_minted(vm_name, token_id)`), and the engine handler already knows `vm_name` at mint time (it just provisioned the VM). The CLI mirrors the Python signature to avoid a parallel wallet→VM lookup.

### `blockhost-network-hook`

Wraps `blockhost.network_hook` for engines that need to compute connection endpoints without spawning Python per call.

| Subcommand | Args | Stdout | Exit | Wraps |
|------------|------|--------|------|-------|
| `resolve` | `<vm_name> <bridge_ip> <mode>` | subscriber-facing host (single line) | 0 ok / 1 error | `network_hook.get_connection_endpoint` |
| `cleanup` | `<vm_name> <mode>` | (empty) | 0 ok / 1 error | `network_hook.cleanup` |

`mode` is `broker` / `manual` / `onion` per §7's network-mode docs.

---

## 7. Config File Schemas

### `/etc/blockhost/db.yaml`

```yaml
db_file: /var/lib/blockhost/vms.json
default_expiry_days: 30
gc_grace_days: 7

ip_pool:
  network: "192.168.122.0/24"
  start: 200              # int (last octet) or full IP string
  end: 250
  gateway: "192.168.122.1"

ipv6_pool:
  start: 2
  end: 254

# Optional — set by provisioner if needed
# vmid_range:
#   start: 100
#   end: 999

# Optional — set by provisioner if needed
# terraform_dir: /var/lib/blockhost/terraform
```

**Owned by**: common (ships template in .deb)
**Written by**: wizard finalization (fills in actual IP pool, expiry values)
**Read by**: every provisioner script, VM database, GC

**Note on `fields:` removal:** earlier versions of common shipped an optional `fields:` block for renaming VM-record keys. The mechanism was identity-only and applied inconsistently (only `register_vm` honored it; every other mutator hardcoded literal keys). It has been removed. The template no longer ships it. Existing installs that customised `db.yaml` keep their `fields:` block on upgrade (conffile semantics) — the code ignores it, so the orphaned block is harmless.

### `/etc/blockhost/web3-defaults.yaml`

```yaml
blockchain:
  chain_id: 11155111
  nft_contract: ""          # Set after contract deployment
  rpc_url: "https://ethereum-sepolia-rpc.publicnode.com"

deployer:
  private_key_file: "/etc/blockhost/deployer.key"

signing_page:
  port: 8443
  html_path: "/usr/share/blockhost/signing-page/index.html"  # Engine-provided

auth:
  otp_length: 6
  otp_ttl_seconds: 300
```

**Owned by**: common (ships template in .deb)
**Written by**: wizard finalization (fills in contract address, chain ID)
**Read by**: mint_nft, vm-create, app.py, engine

### `/etc/blockhost/broker-allocation.json`

```json
{
  "prefix": "2a11:6c7:f04:276::/120",
  "gateway": "2a11:6c7:f04:276::1",
  "broker_pubkey": "...",
  "broker_endpoint": "...",
  "dns_zone": "blockhost.thawaras.org"
}
```

**Owned by**: blockhost-broker-client (writes on allocation)
**Read by**: common's `load_broker_allocation()`, VM database (IPv6 pool), provisioners (FQDN derivation)
**Optional**: Missing = no IPv6 allocation available

**`dns_zone`** (optional string): Broker's authoritative DNS zone. When present, the broker runs an authoritative DNS server mapping `{hex_label}.{dns_zone}` → `{prefix}::{hex_label}`. Provisioners derive per-VM FQDNs by converting the IPv6 offset to lowercase hex: `f"{offset:x}.{dns_zone}"`. Enables Let's Encrypt on VM signing pages (replaces self-signed TLS cert). Missing or empty = signing pages use IP with self-signed cert.

### `/etc/blockhost/blockhost.yaml`

```yaml
public_secret: "..."
server_public_key: "..."
deployer_address: "0x..."
contract_address: "0x..."
```

**Owned by**: installer / init scripts
**Read by**: `load_blockhost_config()`, validate_system.py

### `/etc/blockhost/network-mode`

```
broker
```

Single line containing the network mode: `broker`, `manual`, or `onion`. Written by wizard finalization. Read by first-boot (skips broker-client install in onion mode), engine handler (passes to network hook), and the network hook itself.

**Owned by**: wizard finalization
**Read by**: first-boot, engine handler, network hook

### Network Hook

Module: `blockhost.network_hook`

Provides network-mode-agnostic connection endpoint resolution. The engine handler calls this after `provisioner.create()` to get the endpoint subscribers use to connect.

```python
get_connection_endpoint(vm_name: str, bridge_ip: str, mode: str) -> str
```

| Mode | Behavior | Returns |
|------|----------|---------|
| `broker` | Pass-through (IPv6 from broker-allocation.json) | IPv6 address string |
| `manual` | Pass-through (static IP) | Static IP string |
| `onion` | Calls root agent `tor-hidden-service-add`, pushes `.onion` into VM via `guest-exec`, updates signing URL | `.onion` address |

```python
cleanup(vm_name: str, mode: str) -> None
```

Removes network resources on VM destroy. Onion mode calls root agent `tor-hidden-service-remove`.

### Root Agent — Tor Actions

The root agent (`root_agent_actions/system.py`) provides two actions for hidden service lifecycle:

**`tor-hidden-service-add`:** Params: `vm_name`, `bridge_ip`, `port=22`. Creates `/var/lib/tor/blockhost-{name}/`, appends `HiddenServiceDir` and `HiddenServicePort` to `/etc/tor/torrc`, reloads tor, reads the generated `.onion` from the `hostname` file, returns it.

**`tor-hidden-service-remove`:** Params: `vm_name`. Removes matching lines from `/etc/tor/torrc`, reloads tor, deletes `/var/lib/tor/blockhost-{name}/`.

---

## 8. Installed File Locations

### From blockhost-common .deb

```
/etc/blockhost/
  ├── db.yaml                    # Config template (conffile)
  └── web3-defaults.yaml         # Config template (conffile)

/usr/bin/
  ├── blockhost-vmdb             # CLI wrapper for VMDatabaseBase (see §6a)
  └── blockhost-network-hook     # CLI wrapper for network_hook (see §6a)

/usr/lib/python3/dist-packages/blockhost/
  ├── __init__.py                # Package entry, re-exports (incl. naming validators)
  ├── config.py                  # Configuration loading
  ├── vm_db.py                   # VM database abstraction
  ├── provisioner.py             # Provisioner dispatcher
  ├── root_agent.py              # Root agent client
  ├── cloud_init.py              # Template rendering
  ├── naming.py                  # Domain-name validators (see §5a)
  └── network_hook.py            # Connection endpoint resolution (see §7)

/usr/share/blockhost/
  ├── root-agent/
  │   └── blockhost_root_agent.py
  ├── root-agent-actions/
  │   ├── _common.py             # Shared utilities (not an action module)
  │   ├── system.py              # iptables, virt-customize, wallet
  │   └── networking.py          # IPv6 routes
  └── cloud-init/templates/
      ├── nft-auth.yaml
      ├── webserver.yaml
      └── devbox.yaml

/var/lib/blockhost/              # Data directory (created by postinst, mode 750)
```

**CLI tools** (see §6a): `/usr/bin/blockhost-vmdb` and `/usr/bin/blockhost-network-hook` are thin wrappers over the Python API for engines that prefer one exec over `python3 -c "<inline>"`.

---

## 9. Contract Violations & Cleanup Status

| # | Item | Location | Problem | Status |
|---|------|----------|---------|--------|
| 1 | ~~`qm_start/stop/shutdown/destroy`~~ | ~~`root_agent.py`~~ | ~~Proxmox-specific wrappers in common~~ | RESOLVED: removed from codebase |
| 2 | ~~`get_terraform_dir()`~~ | ~~`config.py`~~ | ~~Terraform is Proxmox-specific~~ | RESOLVED: removed from codebase |
| 3 | ~~`TERRAFORM_DIR` constant~~ | ~~`config.py`, `__init__.py`~~ | ~~Same~~ | RESOLVED: removed from codebase |
| 4 | ~~`mint_nft` module~~ | ~~`blockhost/mint_nft.py`~~ | ~~Minting is engine responsibility~~ | RESOLVED: moved to blockhost-engine-evm |
| 5 | ~~`LEGACY_COMMANDS` fallback~~ | ~~`provisioner.py`~~ | ~~Hardcoded Proxmox commands when no manifest~~ | RESOLVED: removed from codebase |
| 6 | ~~`ALLOWED_ROUTE_DEVS = {'vmbr0'}`~~ | ~~`_common.py`~~ | ~~vmbr0 is Proxmox-specific bridge name~~ | RESOLVED: expanded to include virbr0, br0, br-ext, docker0 |
| 7 | ~~`QM_SET_ALLOWED_KEYS`, `QM_CREATE_ALLOWED_ARGS`~~ | ~~`_common.py`~~ | ~~Proxmox constants in shared code~~ | RESOLVED: removed from codebase (live in provisioner-proxmox's qm.py) |
| 8 | `vmid_range`/`allocate_vmid()` | `vm_db.py` | VMID is Proxmox-specific (libvirt uses domain names) | RESOLVED in API surface: present and functional, dormant for libvirt (raises `RuntimeError` if `vmid_range` not configured, so libvirt never trips it). Acceptable as long as it remains optional to call. |
| 9 | ~~`ip6_route_add/del()` default `dev="vmbr0"`~~ | ~~`root_agent.py`~~ | ~~Proxmox-ism in shared wrapper~~ | RESOLVED: defaults removed; `dev` is now a required argument. |

---

## 10. Consumers

| Package | Imports From Common | Config Files Read |
|---------|--------------------|--------------------|
| **blockhost-provisioner-proxmox** | config (5 functions), vm_db, root_agent (ip6 wrappers + errors), cloud_init | db.yaml, web3-defaults.yaml, broker-allocation.json |
| **blockhost-provisioner-libvirt** | config (3 functions), vm_db, root_agent (`call()` direct), cloud_init | db.yaml, web3-defaults.yaml, broker-allocation.json |
| **blockhost (installer)** | config, provisioner dispatcher | web3-defaults.yaml |
| **blockhost-engine-evm** | config, vm_db, root_agent | db.yaml, web3-defaults.yaml |
| **blockhost-broker** | config (broker allocation) | broker-allocation.json |
