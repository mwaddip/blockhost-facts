# Admin Panel Interface Specification

> Authoritative reference for the admin panel: built-in API surface, authentication,
> config dependencies, and the plugin interface for provisioner-contributed pages.
>
> The admin panel is a Flask app behind nginx. It provides built-in system management
> pages and discovers provisioner plugins at startup for hypervisor-specific pages.

---

## 1. Architecture

```
Browser → nginx (:443, TLS) → location {path_prefix} → proxy_pass 127.0.0.1:8443 (Flask)
```

- **Service:** `blockhost-admin.service` (systemd, `User=blockhost`)
- **Bind:** `127.0.0.1:8443` (localhost only — never exposed directly)
- **Entry:** `python3 -m admin.app --host 127.0.0.1 --port 8443`
- **Config:** `/etc/blockhost/admin.json`
- **Code:** `/opt/blockhost/admin/`

### Path Prefix

All routes live under a configurable path prefix (default `/admin`). The prefix is:
1. Read from `/etc/blockhost/admin.json` → `path_prefix` key
2. Stored in `app.config["PATH_PREFIX"]`
3. Injected into all templates as `{{ admin_prefix }}`
4. Used by nginx `location` block for prefix-stripping reverse proxy

Changing the prefix requires updating `admin.json`, nginx config, and restarting both services. The built-in admin path API handles this via root agent action `admin-path-update`.

---

## 2. Authentication

NFT-gated wallet signature verification. Access is restricted to the current holder of the admin credential NFT.

### Flow

1. Browser loads `/login` → server generates challenge code
2. User signs challenge with their wallet (MetaMask, signing page, or manual paste)
3. `POST /api/auth/verify` with `{code, signature}`
4. Backend recovers signer address from signature
5. Backend calls `bw who admin` → queries `ownerOf(credential_nft_id)` on-chain
6. If signer == NFT owner → session created, redirect to `/`
7. If not → 401

### Session

- Flask server-side session with `auth_token` key
- Token maps to in-memory session store (validated per request)
- Lifetime: 1 hour (`PERMANENT_SESSION_LIFETIME`)
- Cookie: `HttpOnly`, `SameSite=Lax`, `Secure`

### `login_required` Decorator

All authenticated routes use `@login_required` from `admin.auth`. Unauthenticated requests redirect to `{PATH_PREFIX}/login`.

### NFT Transfer

If NFT #0 is transferred, the new holder becomes admin. No config changes needed — `bw who admin` resolves the current owner on-chain (cached 60s).

---

## 3. Built-in Pages

| Page | Route | Template | Active Page ID | Cards |
|------|-------|----------|----------------|-------|
| System & Storage | `/` | `system.html` | `system` | System, Storage |
| Network & Security | `/network` | `network.html` | `network` | Network, Security, Admin Path |
| Wallet Management | `/wallet` | `wallet.html` | `wallet` | Wallet, Addressbook |
| VMs & Accounts | `/vms` | `vms.html` | `vms` | VMs, Accounts (placeholder) |

All page routes pass `active_page` and `wallet` (short address) to the template via `_page_context()`.

---

## 4. Built-in API

All API routes require `@login_required`. Request/response is JSON.

### System

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| GET | `/api/system` | — | `{hostname, uptime_seconds, os, kernel}` |
| POST | `/api/system/hostname` | `{hostname: str}` | `{ok: bool, error?: str}` |

### Network

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| GET | `/api/network` | — | `{ipv4, gateway, dns: str[], ipv6_broker: {prefix, status}?}` |
| POST | `/api/network/broker/renew` | — | `{ok: bool, error?: str}` |

Broker renew goes through root agent action `broker-renew`.

### Security

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| GET | `/api/security` | — | `{enabled, knock_command, knock_ports: int[], knock_timeout}` |
| POST | `/api/security` | `{knock_command?, knock_ports?, knock_timeout?}` | `{ok: bool, error?: str}` |

Reads/writes `/etc/blockhost/admin-commands.json` directly.

### Admin Path

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| GET | `/api/admin/path` | — | `{path_prefix: str}` |
| POST | `/api/admin/path` | `{path_prefix: str}` | `{ok: bool, new_path?: str, error?: str}` |

POST writes `admin.json`, then calls root agent `admin-path-update` which updates nginx and schedules admin service restart (3s delay).

### Storage

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| GET | `/api/storage` | — | `{devices: lsblk[], usage: [{mount, total, used, free}], boot_device}` |

### VMs

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| GET | `/api/vms` | — | `[{name, status, ip, created}]` |
| POST | `/api/vms/<name>/start` | — | `{ok: bool, error?: str}` |
| POST | `/api/vms/<name>/stop` | — | `{ok: bool, error?: str}` |
| POST | `/api/vms/<name>/kill` | — | `{ok: bool, error?: str}` |
| POST | `/api/vms/<name>/destroy` | — | `{ok: bool, error?: str}` |

VM commands resolved through provisioner manifest `commands.*` → CLI executables.

### Wallet

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| GET | `/api/wallet` | — | `[{role, address, can_sign}]` |
| GET | `/api/wallet/balance/<role>` | — | `{ok, output?, error?}` |
| POST | `/api/wallet/send` | `{amount, token, from, to}` | `{ok, output?, error?}` |
| POST | `/api/wallet/withdraw` | `{to, token?}` | `{ok, output?, error?}` |

### Addressbook

| Method | Route | Request | Response |
|--------|-------|---------|----------|
| POST | `/api/addressbook/add` | `{name, address}` | `{ok, output?, error?}` |
| POST | `/api/addressbook/remove` | `{name}` | `{ok, output?, error?}` |
| POST | `/api/addressbook/generate` | `{name}` | `{ok, output?, error?}` |

Wallet/addressbook operations shell out to `bw` and `ab` CLIs with env from `/opt/blockhost/.env`.

---

## 5. Config Dependencies

### Reads

| File | What | Used by |
|------|------|---------|
| `/etc/blockhost/admin.json` | `path_prefix` | `app.py` (startup), `system.py` (path API) |
| `/etc/blockhost/admin-commands.json` | Knock settings | Security API |
| `/etc/blockhost/addressbook.json` | Wallet entries | Wallet/addressbook API |
| `/etc/blockhost/broker-allocation.json` | IPv6 broker status | Network API |
| `/opt/blockhost/.env` | `RPC_URL`, `BLOCKHOST_CONTRACT` | Wallet CLI env |
| `/usr/share/blockhost/provisioner.json` | Manifest (plugin discovery) | `app.py` (startup) |

### Writes

| File | What | Written by |
|------|------|-----------|
| `/etc/blockhost/admin.json` | Path prefix | Admin path API |
| `/etc/blockhost/admin-commands.json` | Knock settings | Security API |

### Ownership

| File | Owner | Mode |
|------|-------|------|
| `/etc/blockhost/admin.json` | `blockhost:blockhost` | 0644 |
| `/etc/blockhost/admin-commands.json` | `root:blockhost` | 0644 |

---

## 6. CLI Dependencies

| Command | Used by | Purpose |
|---------|---------|---------|
| `bw balance <role>` | Wallet API | Check wallet balance |
| `bw send <amount> <token> <from> <to>` | Wallet API | Transfer tokens |
| `bw withdraw [token] <to>` | Wallet API | Contract withdrawal |
| `bw who admin` | Auth | Resolve NFT owner address |
| `ab add <name> <address>` | Addressbook API | Add entry |
| `ab del <name>` | Addressbook API | Remove entry |
| `ab new <name>` | Addressbook API | Generate wallet |
| `hostnamectl set-hostname <name>` | System API | Set hostname |
| `blockhost-vm-list --format json` | VM API | List VMs |
| `blockhost-vm-{start,stop,kill,destroy} <name>` | VM API | VM lifecycle |

All `blockhost-vm-*` commands resolved through provisioner manifest.

---

## 7. Root Agent Actions

| Action | Params | Purpose | Defined in |
|--------|--------|---------|-----------|
| `broker-renew` | (none) | Renew IPv6 broker lease | `blockhost-common` system.py |
| `admin-path-update` | `{path_prefix: str}` | Update nginx location + schedule admin restart | `admin/root-agent-actions/admin_panel.py` |

---

## 8. Template Architecture

### Base Template (`base.html`)

All pages extend `base.html`. It provides:

**Blocks:**

| Block | Purpose | Default |
|-------|---------|---------|
| `title` | `<title>` tag | `BlockHost Admin` |
| `page_title` | Topbar `<h1>` | `Dashboard` |
| `extra_css` | Additional styles | empty |
| `content` | Main page content | empty |
| `extra_js` | Page-specific JavaScript | empty |

**Shared JavaScript** (available in all pages via base.html `<script>`):

| Function | Signature | Purpose |
|----------|-----------|---------|
| `apiGet(url)` | `async (string) → object` | GET `API_PREFIX + url`, parse JSON |
| `apiPost(url, body)` | `async (string, object?) → object` | POST JSON to `API_PREFIX + url` |
| `showAlert(msg, type)` | `(string, 'success'\|'error') → void` | Flash message in `#global-alert` |
| `formatUptime(seconds)` | `(number?) → string` | `"2d 5h 30m"` format |
| `formatBytes(bytes)` | `(number?) → string` | `"1.5 GB"` format |
| `toggleCard(header)` | `(HTMLElement) → void` | Collapse/expand card |
| `updateStatus(data)` | `(object) → void` | Update topbar status from system data |
| `loadStatus()` | `async () → void` | Fetch `/api/system` and update topbar |

**Constants:**
- `API_PREFIX` — the admin path prefix (e.g., `/admin`)

**Topbar status:** `loadStatus()` is NOT auto-called. Each page calls it in its DOMContentLoaded handler (or feeds `updateStatus()` directly if it already fetches system data).

**Sidebar nav:** Currently hardcoded in base.html. Will become dynamic when plugin support is implemented (see section 9).

**CSS classes:** See base.html `<style>` block. Key classes: `.card`, `.card-header`, `.card-body`, `.kv-row`, `.kv-label`, `.kv-value`, `.btn`, `.btn-*`, `.badge`, `.badge-*`, `.form-inline-edit`, `.progress-bar`, `.progress-fill`, `.placeholder-msg`, `.spinner-sm`, `.alert`.

---

## 9. Provisioner Admin Plugin

Provisioners can contribute pages to the admin panel. The pattern mirrors the wizard plugin system.

### Manifest Extension

Add to `provisioner.json`:

```json
{
  "admin": {
    "module": "<python.module.path>"
  }
}
```

Example: `"module": "blockhost.provisioner_proxmox.admin"`

### Discovery

At startup, the admin app:
1. Reads `/usr/share/blockhost/provisioner.json`
2. If `admin.module` exists, imports the module
3. Registers the module's `blueprint`
4. Merges the module's `PAGES` into sidebar navigation

### Required Exports

| Export | Type | Description |
|--------|------|-------------|
| `blueprint` | `flask.Blueprint` | Routes and templates for provisioner pages |
| `PAGES` | `list[dict]` | Sidebar navigation entries |

### `PAGES` Schema

```python
PAGES = [
    {
        "id": "hypervisor",       # Matches active_page for sidebar highlighting
        "path": "/hypervisor",    # Route path (relative to admin prefix)
        "label": "Hypervisor",    # Sidebar display text
        "icon": "&#9881;",        # HTML entity for nav icon
    },
]
```

Pages appear in the sidebar after the built-in entries, in list order.

### Blueprint

```python
from flask import Blueprint, render_template

blueprint = Blueprint(
    "admin_provisioner",
    __name__,
    template_folder="templates",
)

@blueprint.route("/hypervisor")
def hypervisor_page():
    return render_template("hypervisor.html", active_page="hypervisor")
```

**Auth:** The admin app wraps provisioner blueprints with `before_request` auth enforcement. Plugin routes do NOT need to import or apply `@login_required` — the host handles it.

**Context:** `admin_prefix` and `wallet` are injected by the app's context processor. Plugin templates get them for free.

### Templates

Plugin templates extend `base.html`:

```html
{% extends "base.html" %}

{% block title %}Hypervisor - BlockHost Admin{% endblock %}
{% block page_title %}Hypervisor{% endblock %}

{% block content %}
<div class="card" id="pools">
    <div class="card-header" onclick="toggleCard(this)">
        <h2><span class="section-icon">&#9744;</span> Storage Pools</h2>
        <span class="card-arrow">&#9660;</span>
    </div>
    <div class="card-body">
        <!-- ... -->
    </div>
</div>
{% endblock %}

{% block extra_js %}
<script>
// All base.html JS helpers available: apiGet, apiPost, showAlert, etc.
async function loadPools() {
    const data = await apiGet('/api/hypervisor/pools');
    // ...
}

document.addEventListener('DOMContentLoaded', () => {
    loadPools();
    loadStatus();
});
</script>
{% endblock %}
```

### API Routes

Plugin API routes are part of the blueprint. Convention: `/api/<plugin-id>/...`

```python
@blueprint.route("/api/hypervisor/pools")
def api_pools():
    return jsonify(get_pool_info())
```

The admin app registers the blueprint at the root of the Flask app (no URL prefix on the blueprint itself — the nginx prefix-stripping handles the path prefix).

### Install Location

Plugin Python module: `/usr/lib/python3/dist-packages/blockhost/provisioner_<name>/admin.py`
Plugin templates: `/usr/lib/python3/dist-packages/blockhost/provisioner_<name>/templates/`

The `.deb` package installs these alongside the existing provisioner module.

---

## 10. Known Issues

### Root agent action not in package

`admin/root-agent-actions/admin_panel.py` is currently manually installed to `/usr/share/blockhost/root-agent-actions/`. Needs to be included in the admin panel deployment (systemd unit or package postinst).

### No provisioner admin plugins exist yet

The plugin discovery and registration system is implemented in `app.py`. No provisioner has shipped an admin plugin yet. To test: add `"admin": {"module": "..."}` to the provisioner manifest and export `blueprint` + `PAGES` from the module.
