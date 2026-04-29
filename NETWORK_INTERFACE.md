# Network Interface Specification

> Authoritative contract for the network-layer plugin boundary.
> Network modes (onion, broker, manual, none, …) are plugins discovered via
> the apache-style `available/enabled` symlink pattern. Common ships only
> the dispatcher CLI; mode-specific logic lives in plugins.
>
> The dispatcher takes a `vm_name` and resolves its `network_mode` from
> `vm-db.json`. Today, every VM gets the same mode (the one symlinked into
> `enabled/` at finalization). Tomorrow, multiple modes can run
> concurrently — the contract does not change, only the activation set
> grows beyond a single symlink.

---

## 1. Overview

| Concern | Owner |
|---------|-------|
| `network_mode` field on each VM record | `blockhost-common` (vm-db schema) |
| Dispatcher CLI (`blockhost-network-hook`) | `blockhost-common` |
| Plugin manifests (`network-modes.available/`) | Main repo / installer (ships them) |
| Plugin scripts (`/usr/share/blockhost/network/<mode>/`) | Main repo / installer |
| Active-mode symlinks (`network-modes.enabled/`) | Wizard finalization |

Engines and provisioners do not import network code. They invoke the dispatcher CLI for everything network-shaped.

---

## 2. Discovery and activation

apache `sites-available`/`sites-enabled` pattern. Manifests live in `/etc/blockhost/network-modes.available/`; activation is a symlink under `/etc/blockhost/network-modes.enabled/`.

```
/etc/blockhost/
├── network-modes.available/      # all installed plugins
│   ├── onion.json
│   ├── broker.json
│   ├── manual.json
│   └── none.json
└── network-modes.enabled/        # active modes (symlinks)
    └── onion.json -> ../network-modes.available/onion.json
```

| Operation | Mechanism |
|-----------|-----------|
| Install a plugin | Drop manifest in `network-modes.available/`, drop scripts in `/usr/share/blockhost/network/<mode>/`. |
| Uninstall a plugin | Remove from both. Removing the manifest while it's symlinked from enabled/ leaves a dangling link — the dispatcher rejects dispatch via dangling links. |
| Activate a plugin | Create symlink in `network-modes.enabled/` pointing into `available/`. Wizard does this on user choice. |
| Deactivate | Remove the symlink. |
| Discover available | List `network-modes.available/`. |
| Discover active | List `network-modes.enabled/`. Symlink target tells you the manifest. |

Why symlinks: package install/uninstall, operator manual edits, and wizard automation all converge on the same primitive (`ln -s` / `rm`). Operators familiar with apache/systemd already know the pattern.

---

## 3. Manifest schema

**Path** (when installed): `/etc/blockhost/network-modes.available/<mode>.json`.

```json
{
  "name": "onion",
  "display_name": "Onion Routing (Tor)",
  "description": "Each VM gets its own .onion hidden service.",
  "package": "blockhost-installer",
  "exclusive_with": ["*"],
  "commands": {
    "public-address": "/usr/share/blockhost/network/onion/public-address.py",
    "push-vm-config": "/usr/share/blockhost/network/onion/push-vm-config.py",
    "cleanup":        "/usr/share/blockhost/network/onion/cleanup.py"
  },
  "finalize_d": "/usr/share/blockhost/network/onion/finalize.d/",
  "validation": {
    "required_packages": ["tor"]
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Mode key. Must match the manifest's basename and the `network_mode` string written into vm-db. |
| `display_name` | yes | Human-readable label for the wizard. |
| `description` | yes | One-sentence pitch shown in the wizard. |
| `package` | no | Debian package providing this plugin (informational). |
| `exclusive_with` | yes | Coexistence rules. See §4. |
| `commands` | yes | Map of subcommand → executable path. Missing entry = unsupported subcommand → dispatcher exits non-zero with `plugin does not implement <subcommand>`. |
| `finalize_d` | no | Directory of finalization-step executables. Wizard splices these into its progress UI when this mode is enabled. See §5. |
| `validation` | no | Hooks for `validate_system.py` to assert the plugin is functional. |

---

## 4. Coexistence (`exclusive_with`)

Three forms:

```json
"exclusive_with": []                    // coexists with anything
"exclusive_with": ["*"]                  // fully exclusive — cannot coexist with any other mode
"exclusive_with": ["onion", "manual"]   // partially exclusive — names specific conflicts
```

The wizard validates the proposed enabled-set pairwise:

1. For every pair `(A, B)` in the new set:
   - If either side's `exclusive_with` contains `*`, conflict.
   - If `A.exclusive_with` contains `B.name` or `B.exclusive_with` contains `A.name`, conflict.
2. On any conflict, reject the change.

**Symmetric closure.** A plugin may declare `exclusive_with` against a sibling that itself declares `[]`. The wizard treats the relation as symmetric — declaring it on one side suffices.

**Today:** all four built-in modes set `["*"]`. The wizard renders as radio buttons (one symlink at a time).
**Tomorrow:** a new plugin compatible with another sets `[]` or names specific conflicts; the wizard renders compatible subsets as checkboxes.

Built-in plugins MUST set `exclusive_with` explicitly. There is no default — leaving it absent makes the manifest invalid.

---

## 5. Finalization hooks (`finalize.d/`)

Each plugin may declare a `finalize.d/` directory of executable finalization steps. The wizard's finalization pipeline iterates **enabled** modes and walks each one's `finalize.d/`, splicing the executables into its step list at the configured insertion point.

```
/usr/share/blockhost/network/onion/finalize.d/
├── 01-install-tor
├── 02-host-hidden-service
└── 03-write-https-json
```

Each executable runs as one wizard step. Discovery rules:

- Lexical order (NN-name convention recommended).
- Filename → step ID. Strip a leading `NN-` prefix if present.
- Step label: read from a `# label:` line in the first 10 lines of the script. Falls back to humanised step ID.

Each script receives:

| Source | Variable | Description |
|--------|----------|-------------|
| Env | `BH_MODE` | The plugin's mode name. |
| Env | `BH_CONFIG_JSON` | Path to a JSON file containing the wizard's session config (whatever finalize.py would have passed to a Python step). The script reads what it needs. |

Scripts exit 0 on success, non-zero on retryable failure. Stdout and stderr are captured for the wizard UI. A script may write its own `_step_result` shape to a known temp path (`$BH_STEP_RESULT_FILE`); the wizard merges it into the step's data block (see `installer/web/templates/wizard/summary.html` for renderer).

**Splicing point in `finalize.py`.** The pipeline today is roughly: server keypair → engine wallet → contracts → write configs → provisioner steps → **[network finalization]** → signup page → nginx → mint NFT. The network slot is where each enabled mode's `finalize.d/` runs, in mode order (lexical by mode name when multiple are enabled).

Modes that have no `finalize.d/` simply contribute nothing to the step list.

---

## 6. `network_mode` field

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
| Source | only one mode is enabled, so the provisioner reads its name from `network-modes.enabled/` and uses that. | plan-id → mode mapping, or wizard-time per-VM choice. |
| Mutability | immutable after vm-create | immutable after vm-create |
| Validation | provisioner enforces the value matches an enabled plugin | unchanged |
| Default for missing field | hard error | hard error |

Once a VM is provisioned with mode X, its public address is baked into its NFT. Switching modes for an existing VM is out of scope — operators destroy and re-provision instead.

---

## 7. Plugin commands

Each subcommand the dispatcher exposes maps 1:1 to a plugin command. Plugins implement only the ones they need; the rest are absent from the manifest.

### 7.1 `public-address`

```
<plugin> public-address <vm-name>
```

Returns the publicly-routable address for the VM. Stdout: a single line, no trailing whitespace. Exit 0 = success, non-zero = no address.

Idempotent. No fallback — if the plugin can't determine an address, exit non-zero. Engines treat this as a hard failure (no NFT mint with garbage data).

### 7.2 `push-vm-config`

```
<plugin> push-vm-config <vm-name>
```

Pushes mode-specific configuration *inside* the VM via guest-exec. Idempotent. Best-effort; engines retry via reconciler.

Exit 0 = config now correct. Non-zero = retry needed.

Plugins for which no in-VM push is needed (broker, manual, none) implement this as a no-op exit 0. Plugins MUST declare this command in their manifest; the dispatcher does not no-op missing commands.

### 7.3 `cleanup`

```
<plugin> cleanup <vm-name>
```

Releases per-VM resources allocated by `public-address` and `push-vm-config`. Reverses both host-side and guest-side state.

Idempotent. Called by the engine handler during subscription cancellation, before the provisioner's `vm-destroy`.

### 7.4 `pre-provision` (optional, future-facing)

```
<plugin> pre-provision <plan-id>
```

Allocates / reserves any values that must exist *before* `vm-create` runs. Stdout: JSON of name → value pairs that the engine forwards to the provisioner.

Today no engine calls this; the hook is named for forward compatibility.

---

## 8. Dispatcher CLI: `blockhost-network-hook`

Shipped by `blockhost-common`. Single entry point for all network-layer operations.

```
blockhost-network-hook <subcommand> [args...]
```

| Subcommand | Args | Description |
|------------|------|-------------|
| `public-address` | `<vm>` | Resolve VM's `network_mode` from vm-db, dispatch to plugin's `public-address`. |
| `push-vm-config` | `<vm>` | Same resolution, plugin's `push-vm-config`. |
| `cleanup` | `<vm>` | Same resolution, plugin's `cleanup`. |
| `pre-provision` | `<mode> <plan-id>` | Forward to that mode's `pre-provision`. Used pre-vm-create. |
| `mode` | `<vm>` | Echo `vm-db.network_mode[<vm>]` (debugging). |
| `list-available` | — | List manifests in `network-modes.available/`. |
| `list-enabled` | — | List symlinks in `network-modes.enabled/`. |
| `enable` | `<mode>` | Create the symlink. Validates `exclusive_with` against currently-enabled modes; rejects on conflict. |
| `disable` | `<mode>` | Remove the symlink. |

Resolution rules for VM-keyed subcommands (`public-address`, `push-vm-config`, `cleanup`):

1. Read `vm-db.network_mode[<vm>]`. Missing → exit non-zero.
2. Resolve `/etc/blockhost/network-modes.enabled/<mode>.json`. Missing or dangling → exit non-zero ("mode not enabled").
3. Read manifest, look up `commands.<subcommand>`. Missing → exit non-zero.
4. Exec the plugin command with `BH_VM_NAME=<vm>` (or `BH_PLAN_ID=<id>` for `pre-provision`) in the environment plus the same args on argv. Forward stdout/stderr/exit code unchanged.

The dispatcher does no semantic logic of its own. All semantics live in plugins.

---

## 9. validate_system.py contributions

Each plugin manifest may declare validation requirements via the `validation` block. `validate_system.py` reads each enabled manifest and applies its rules.

Per-VM validation (run for every active VM):

- `vm-db.network_mode` is non-empty.
- The named plugin is enabled (symlink exists in `network-modes.enabled/`).
- The plugin's `public-address <vm>` command exits 0 and returns non-empty stdout.

Plugin-declared validation (from manifest `validation` block):

- `required_files`: each path must exist.
- `required_packages`: each `dpkg -s` reports `installed`.

System-level validation:

- `network-modes.enabled/` is non-empty (host has at least one active network mode), unless the wizard is configured for a deliberate `none`-only deployment.
- Every symlink in `network-modes.enabled/` resolves into `network-modes.available/`.

---

## 10. Forward compatibility

This contract is built so the day multi-mode lands, no consumer changes:

- Per-VM `network_mode` in vm-db means dispatch is per-VM from day one. Adding a new write source (plan-id mapping, multi-select wizard) doesn't touch the dispatcher.
- `exclusive_with` already specifies coexistence semantics. Compatible plugins just declare it; the wizard learns to multi-select without contract change.
- `available/enabled` is the natural set-shape for activation. One symlink today, multiple tomorrow.
- `pre-provision` is named but optional. Plugins that need it implement it; the engine starts calling it when plans grow `network_mode` declarations.
- `finalize.d/` directories scale: each enabled plugin's hooks run, in stable order, regardless of how many modes are enabled.

The dispatcher rejects modes with no symlink, rejects VMs with no `network_mode`, rejects exclusivity conflicts. An unsupported configuration is impossible to construct accidentally.
