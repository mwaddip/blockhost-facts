# Network Interface Specification

> Authoritative contract for the network-layer plugin boundary.
> Network modes (onion, broker, manual, none, …) are plugins discovered at
> runtime. Common ships only the dispatcher CLI; mode-specific logic lives in
> plugin packages.
>
> The dispatcher takes a `vm_name` and resolves its `network_mode` from
> `vm-db.json`. Today, every VM gets the same mode (the global one chosen at
> finalization). Tomorrow, different VMs can run different modes — the
> contract does not change, only the source of `network_mode` does.

---

## 1. Overview

Three responsibilities, three places:

| Concern | Owner |
|---------|-------|
| `network_mode` field on each VM record | `blockhost-common` (vm-db schema) |
| Dispatcher CLI (`blockhost-network-hook`) | `blockhost-common` |
| Mode plugins (onion, broker, manual, none) | Main repo / installer |

Engines and provisioners do not import network code. They invoke the dispatcher CLI for everything network-shaped.

---

## 2. Lifecycle

```
finalization (operator picks mode):
  /etc/blockhost/network-mode      ← single line, e.g. "onion"
  /usr/share/blockhost/network/onion.json present
  plugin's host-setup hook runs (optional)

vm-create (per subscription):
  provisioner reads /etc/blockhost/network-mode
  provisioner calls register_vm(..., network_mode=<mode>)
  vm-db record gets network_mode field

post-vm-create (engine handler):
  blockhost-network-hook public-address <vm-name>     → returns address
  blockhost-network-hook push-vm-config <vm-name>     → idempotent VM-side push
  engine encrypts address into NFT, mints

reconciler (engine, periodic):
  blockhost-network-hook push-vm-config <vm-name>     → retried until success

vm-destroy:
  blockhost-network-hook cleanup <vm-name>            → release per-VM resources
```

The CLI's contract is per-VM from day one. Today the lookup is trivial (every VM has the same mode); the day plans declare modes or operators offer multi-mode, only the *source* of `network_mode` changes — not the CLI's signature.

---

## 3. `network_mode` field

Every VM record in `vm-db.json` carries a `network_mode` string, written by the provisioner at `register_vm` time and never mutated afterward.

```json
{
  "vms": {
    "blockhost-001": {
      "vm_name": "blockhost-001",
      "network_mode": "onion",
      ...
    }
  }
}
```

| Aspect | Today | Tomorrow |
|--------|-------|----------|
| Source | global `/etc/blockhost/network-mode` | plan ID → mode mapping, or per-VM operator choice |
| Mutability | immutable after vm-create | immutable after vm-create |
| Validation | provisioner enforces value matches an installed plugin manifest | unchanged |
| Default for missing field | hard error | hard error |

Once a VM is provisioned with mode X, its public address is baked into its NFT. Changing the mode of an existing VM is out of scope — operators destroy and re-provision instead.

---

## 4. Plugin Discovery

Plugins are discovered from `/usr/share/blockhost/network/<mode>.json`. The dispatcher refuses to dispatch to a mode that has no manifest.

The wizard's connectivity step lists every manifest it finds and lets the operator choose one. Adding a mode = installing its package = the wizard sees it.

---

## 5. Plugin Manifest

**Path:** `/usr/share/blockhost/network/<mode>.json`

```json
{
  "name": "onion",
  "display_name": "Onion Routing (Tor)",
  "description": "Each VM gets its own .onion hidden service.",
  "package": "blockhost-network-onion",
  "exclusive_with": [],
  "commands": {
    "public-address": "/usr/share/blockhost/network/onion/public-address.sh",
    "push-vm-config": "/usr/share/blockhost/network/onion/push-vm-config.sh",
    "cleanup":        "/usr/share/blockhost/network/onion/cleanup.sh",
    "host-setup":     "/usr/share/blockhost/network/onion/host-setup.sh",
    "host-teardown":  "/usr/share/blockhost/network/onion/host-teardown.sh",
    "pre-provision":  "/usr/share/blockhost/network/onion/pre-provision.sh"
  },
  "validation": {
    "required_files": [],
    "required_packages": ["tor"]
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Mode key. Must match the file's basename and the `network_mode` string written into vm-db. |
| `display_name` | yes | Human-readable label for the wizard. |
| `description` | yes | One-sentence pitch shown in the wizard. |
| `package` | no | Debian package providing this plugin (informational). |
| `exclusive_with` | no | Other modes that cannot run simultaneously on the same host. Today unused (single-mode); reserved. |
| `commands` | yes | Map of subcommand → executable path. Missing entries = unsupported subcommand → dispatcher exits non-zero. |
| `validation` | no | Hooks for `validate_system.py` to assert the plugin is functional. |

All commands receive `BH_VM_NAME=<vm-name>` (or `BH_PLAN_ID=<id>` for `pre-provision`) in the environment, plus relevant args on the command line. Convention is to read these env vars first and fall back to `argv` for testability.

---

## 6. Plugin Commands

Each subcommand the dispatcher exposes maps 1:1 to a plugin command. Plugins implement only the ones they need; the rest are absent from the manifest.

### 6.1 `public-address`

```
<plugin> public-address <vm-name>
```

Returns the publicly-routable address for the VM. Stdout: a single line, no trailing whitespace, the address (e.g. `xyz...xyz.onion`, `2a11:6c7:f04:276::201`, `192.168.1.5`). Stderr: human-readable diagnostics. Exit 0 = success.

The address format is opaque to consumers. It's whatever a customer pastes into their tooling to reach the VM.

**Idempotent**: calling repeatedly returns the same address. Plugins that allocate at first call (e.g. onion) cache the result and return the cached value on subsequent calls.

**No fallback**: if the plugin can't determine an address, exit non-zero. The engine treats this as a hard failure (no NFT mint). Garbage-in-NFT is worse than no NFT.

### 6.2 `push-vm-config`

```
<plugin> push-vm-config <vm-name>
```

Pushes mode-specific configuration *inside* the VM via guest-exec. Examples (onion mode): write `/run/libpam-web3/signing_host`, set `signing_url` and `use_tls` in `/etc/pam_web3/config.toml`, append `/etc/hosts` entry.

Idempotent. Best-effort. Engines call this from the create handler and from the reconciler — guest-agent timing means the first call may fail and the reconciler retries until it succeeds.

Exit 0 = config is now correct. Exit non-zero = retry needed.

### 6.3 `cleanup`

```
<plugin> cleanup <vm-name>
```

Releases per-VM resources allocated by `public-address` and `push-vm-config`. Examples: remove tor hidden service dir, free the IPv6 from the broker pool, delete /etc/hosts entry from the host. **Crucially, also reverses VM-side config** — the simplify-review-flagged "one-way teardown" lives here. Plugins must clean up both host *and* guest state.

Idempotent. Called by the provisioner during vm-destroy.

### 6.4 `host-setup` (optional)

```
<plugin> host-setup
```

One-time host initialisation at finalization. Examples: install/configure tor's host-side hidden service for the signup page, allocate a broker prefix, configure dummy interface. Called once when the operator activates this mode.

Plugins without host-side state omit this command.

### 6.5 `host-teardown` (optional)

```
<plugin> host-teardown
```

Reverses `host-setup`. Called when the operator deactivates the mode (rare). Plugins omit if not applicable.

### 6.6 `pre-provision` (optional, future-facing)

```
<plugin> pre-provision <plan-id>
```

Allocates / reserves any values that must exist *before* `vm-create` runs (e.g. an IPv6 prefix earmarked for the VM, a pre-generated tor key). Stdout: JSON of name → value pairs that the engine forwards to the provisioner.

Today no engine calls this; the flow is direct. The hook is named in the contract so the day plans declare modes, plugins can implement it without a contract change.

---

## 7. Dispatcher CLI: `blockhost-network-hook`

Shipped by `blockhost-common`. Single entry point for all network-layer operations.

```
blockhost-network-hook <subcommand> [args...]
```

| Subcommand | Resolves mode from | Description |
|------------|--------------------|-------------|
| `public-address <vm>` | `vm-db.network_mode[<vm>]` | Forward to plugin's `public-address` command. |
| `push-vm-config <vm>` | `vm-db.network_mode[<vm>]` | Forward to plugin's `push-vm-config`. |
| `cleanup <vm>` | `vm-db.network_mode[<vm>]` | Forward to plugin's `cleanup`. |
| `host-setup <mode>` | `<mode>` arg | Forward to that plugin's `host-setup`. Used at finalization. |
| `host-teardown <mode>` | `<mode>` arg | Forward to that plugin's `host-teardown`. |
| `pre-provision <mode> <plan-id>` | `<mode>` arg | Forward to that plugin's `pre-provision`. Future. |
| `mode <vm>` | `vm-db.network_mode[<vm>]` | Echo the resolved mode (debugging / tooling). |
| `list-modes` | — | List installed plugin manifests. |

Resolution rules for VM-keyed subcommands:

1. Read `vm-db.network_mode[<vm>]`.
2. If absent: exit non-zero with a clear error. **No fallback to global mode** — the field is required by the schema.
3. Look up `/usr/share/blockhost/network/<mode>.json`. If missing: exit non-zero.
4. Look up the manifest's `commands.<subcommand>`. If missing: exit non-zero ("plugin does not implement <subcommand>").
5. Exec the plugin command, forwarding stdout/stderr/exit code.

The dispatcher does no logic of its own beyond lookup. All semantics live in the plugin.

---

## 8. validate_system.py contributions

Each plugin's manifest can declare validation requirements via the `validation` block. `validate_system.py` reads each installed manifest and applies its rules.

Per-VM validation (run for every active VM):

- `vm-db.network_mode` is non-empty
- The named plugin manifest exists at `/usr/share/blockhost/network/<mode>.json`
- The plugin's `public-address <vm>` command exits 0 and returns non-empty stdout

Plugin-declared validation (from manifest `validation` block):

- `required_files`: each path must exist
- `required_packages`: each `dpkg -s` must report `installed`
- Plugins may extend with custom checks via a `validate` command in their manifest

---

## 9. Forward compatibility

This contract is built so the day multi-mode lands, no consumer changes:

- Per-VM `network_mode` in vm-db means no global lookup at dispatch time. Adding a new write source (plan-id mapping, multi-select wizard) doesn't touch the dispatcher.
- `pre-provision` is named but optional. Plugins that need it implement it; the engine starts calling it when plans grow `network_mode` declarations.
- `exclusive_with` in the manifest reserves multi-mode coordination logic without forcing its implementation today.
- The dispatcher rejects modes with no manifest, rejects VMs with no `network_mode`. An unsupported configuration is impossible to construct accidentally.
