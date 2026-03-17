# Page Template Interface

Defines the contract between chain-specific JS bundles and user-facing HTML/CSS templates (signing page, signup page). Templates are replaceable — the plugin ships a default, anyone can drop in their own as long as it honors this contract.

> **Ownership:** Signing pages and auth-svc are owned by libpam-web3 chain plugins (`libpam-web3-<chain>`), not by the engine. Signup pages remain engine-owned (they handle subscription purchase logic). This contract applies to both.

## Separation Principle

```
template (HTML/CSS)     — layout, branding, copy, styles
engine bundle (JS)      — wallet connection, signing, chain interaction
generator (Python/Bash) — injects config variables, combines template + bundle → output
```

The template never contains wallet or chain logic. The bundle never contains layout or styling. The generator is the glue.

## Directory Convention

```
Signing page (in libpam-web3-<chain> plugin):
  signing-page/
    template.html        ← replaceable (HTML/CSS only)
    engine.js            ← plugin-owned (wallet + signing logic)
    index.html           ← generated (template + bundle + variables)
  scripts/
    signup-template.html ← replaceable (HTML/CSS only)
    signup-engine.js     ← engine-owned (wallet + purchase + decrypt logic)
    signup.html          ← generated output
    generate-signup-page ← generator script
```

## Template Variables

Injected by the generator as `{{VARIABLE}}` placeholders. Templates should use all shared variables and conditionally include engine-specific sections.

### Shared (all engines)

| Variable | Type | Description |
|----------|------|-------------|
| `PAGE_TITLE` | string | Page heading text |
| `PRIMARY_COLOR` | CSS color | Accent color (buttons, links, active states) |
| `PUBLIC_SECRET` | string | Message text the user signs |
| `SERVER_PUBLIC_KEY` | hex string | secp256k1 public key for ECIES encryption |
| `RPC_URL` | URL | Chain RPC endpoint |
| `NFT_CONTRACT` | hex string | NFT contract address |
| `SUBSCRIPTION_CONTRACT` | hex string | Subscription contract address |
| `ENGINE_NAME` | string | Engine identifier (`evm` or `opnet`) |

### Engine-specific (conditionally present)

| Variable | Engine | Type | Description |
|----------|--------|------|-------------|
| `CHAIN_ID` | evm | integer | EVM chain ID |
| `USDC_ADDRESS` | evm | hex string | Payment token (USDC) contract address |
| `PAYMENT_TOKEN` | opnet | hex string | OP_20 payment token address |

Templates must handle absent variables gracefully — either hide the section or use a default.

## Required DOM Elements

The engine JS bundle finds elements by `id`. Templates must include all of these.

### Signing Page

| Element ID | Type | Purpose |
|------------|------|---------|
| `btn-connect` | button | Triggers wallet connection |
| `btn-sign` | button | Triggers message signing |
| `wallet-address` | span/div | Displays connected wallet address |
| `status-message` | div | Shows status/error messages |
| `step-connect` | div | Connect wallet step container |
| `step-sign` | div | Sign message step container |

### Signup Page

| Element ID | Type | Purpose |
|------------|------|---------|
| `btn-connect` | button | Triggers wallet connection |
| `btn-sign` | button | Triggers message signing |
| `btn-purchase` | button | Triggers subscription purchase |
| `wallet-address` | span/div | Displays connected wallet address |
| `plan-select` | select | Plan selection dropdown |
| `days-input` | input | Subscription duration (days) |
| `total-cost` | span/div | Computed cost display |
| `status-message` | div | Shows status/error messages |
| `step-connect` | div | Connect wallet step container |
| `step-sign` | div | Sign message step container |
| `step-purchase` | div | Purchase step container |
| `step-servers` | div | View servers step container |
| `server-list` | div | Container for decrypted server details |

## Engine Bundle API

The bundle exposes no global API. It self-initializes on `DOMContentLoaded`, finds the required DOM elements, and attaches event listeners. The bundle reads its configuration from a `CONFIG` object embedded in the page by the generator:

```html
<script>
  const CONFIG = {
    publicSecret: "{{PUBLIC_SECRET}}",
    serverPublicKey: "{{SERVER_PUBLIC_KEY}}",
    rpcUrl: "{{RPC_URL}}",
    nftContract: "{{NFT_CONTRACT}}",
    subscriptionContract: "{{SUBSCRIPTION_CONTRACT}}",
    // Engine-specific fields (present or absent):
    chainId: {{CHAIN_ID}},           // EVM only
    usdcAddress: "{{USDC_ADDRESS}}", // EVM only
    paymentToken: "{{PAYMENT_TOKEN}}" // OPNet only
  };
</script>
<script src="engine.js"></script>
```

## CSS Contract

The template owns all styling. The engine bundle adds/removes these CSS classes on elements:

| Class | Applied to | Meaning |
|-------|-----------|---------|
| `hidden` | any step container | Step not yet active |
| `active` | step container | Currently active step |
| `completed` | step container | Step finished |
| `disabled` | button | Button not yet clickable |
| `loading` | button | Operation in progress |
| `error` | `#status-message` | Error state |
| `success` | `#status-message` | Success state |

The template defines what these classes look like. The bundle only toggles them.

## Signing Page Callback

The plugin's JS bundle handles the full callback to auth-svc. The template is not involved — the bundle POSTs the signature to `/auth/callback/{sessionId}` (session ID from URL query parameter).

## Customization Guide

To create a custom template:

1. Copy the default `template.html` / `signup-template.html`
2. Modify HTML structure, CSS, copy, images — anything visual
3. Keep all required DOM element IDs intact
4. Keep the `CONFIG` script block and `engine.js` include
5. Rebuild: run the generator script

The template can add any extra elements, sections, or styling. It must not remove or rename the required IDs.
