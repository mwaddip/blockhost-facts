# Wizard UI Style Guide

Style reference for provisioner wizard templates (and any template extending `base.html`). All CSS is defined in `installer/web/templates/base.html` — there are no external stylesheets.

## Theme

Dark theme. All colors via CSS custom properties:

| Variable | Value | Use |
|----------|-------|-----|
| `--primary` | `#2563eb` | Buttons, links, active states |
| `--primary-dark` | `#1d4ed8` | Primary hover |
| `--secondary` | `#64748b` | Secondary text |
| `--success` | `#22c55e` | Success states, completed steps |
| `--error` | `#ef4444` | Error states |
| `--warning` | `#f59e0b` | Warning states |
| `--bg` | `#0f172a` | Page background |
| `--bg-card` | `#1e293b` | Card/header/footer background |
| `--bg-input` | `#334155` | Input/select background, hover states |
| `--text` | `#f1f5f9` | Primary text |
| `--text-muted` | `#94a3b8` | Secondary text, labels, hints |
| `--border` | `#475569` | All borders |

## Page Structure

Every wizard page follows this skeleton:

```html
{% extends "base.html" %}
{% from "macros/wizard_steps.html" import step_bar %}

{% block title %}Page Title - BlockHost Installer{% endblock %}

{% block content %}
{{ step_bar('step_id') }}

<div class="card">
    <h2>Page Title</h2>
    <p class="text-muted mb-2">Brief description of this step.</p>

    <form method="POST" action="{{ url_for('endpoint_name') }}">
        <!-- content -->

        <div class="flex justify-between" style="margin-top: 2rem;">
            <a href="{{ url_for('previous_endpoint') }}" class="btn btn-secondary">Back</a>
            <button type="submit" class="btn btn-primary">Continue</button>
        </div>
    </form>
</div>
{% endblock %}

{% block extra_js %}
<script>
// Page-specific JavaScript
</script>
{% endblock %}
```

### Step bar

The `step_bar` macro receives `wizard_steps` from the context processor (injected automatically). Pass the current step's `id` string — the macro marks previous steps as completed, current as active, future as pending.

The step ID for a provisioner page is the provisioner `name` field from `provisioner.json` (e.g., `"libvirt"`, `"proxmox"`).

### Navigation

The Back button must point to the previous wizard step. For provisioner pages, the previous step is always `wizard_blockchain`:

```html
<a href="{{ url_for('wizard_blockchain') }}" class="btn btn-secondary">Back</a>
```

On POST success, redirect to the next step (`wizard_ipv6`):

```python
return redirect(url_for("wizard_ipv6"))
```

There is also a `wizard_nav()` template global that returns `{'prev': endpoint, 'next': endpoint}` for any step ID, but most pages hardcode their navigation for clarity.

## Components

### Card (`.card`)

The outermost wrapper for page content. Rounded corners, dark background, subtle border.

```html
<div class="card">
    <h2>Section Title</h2>
    <!-- content -->
</div>
```

- `border-radius: 0.75rem`
- `background: var(--bg-card)`
- `border: 1px solid var(--border)`
- `padding: 2rem`

### Form Section (`.form-section` + `<h3>`)

Groups related form fields within a card. This is the correct way to visually separate sections.

```html
<div class="form-section">
    <h3>Section Heading</h3>
    <p class="text-muted mb-2" style="font-size: 0.875rem;">Optional description.</p>
    <!-- form groups -->
</div>
```

- `border: 1px solid var(--border)`
- `border-radius: 0.5rem`
- `padding: 1.5rem`
- `margin-bottom: 1.5rem`
- `h3`: `font-size: 1rem`, bottom border separator

**Do NOT use `<fieldset>` / `<legend>`.** They produce browser-default styling (bright borders, sharp corners, different padding) that doesn't match the theme.

### Form Group (`.form-group`)

Standard wrapper for label + input pairs.

```html
<div class="form-group">
    <label for="field_id">Label Text</label>
    <input type="text" id="field_id" name="field_name" value="" placeholder="...">
    <p class="text-muted mt-1" style="font-size: 0.75rem;">Optional hint text.</p>
</div>
```

- `margin-bottom: 1.5rem`
- Labels: `color: var(--text-muted)`, `font-size: 0.875rem`
- Inputs/selects: full width, `background: var(--bg-input)`, `border-radius: 0.5rem`

### Two-Column Layout (`.two-col`)

Side-by-side form groups. Collapses to single column on mobile.

```html
<div class="two-col">
    <div class="form-group">...</div>
    <div class="form-group">...</div>
</div>
```

### Form Row (`.form-inline`)

Horizontal form groups on a single line.

```html
<div class="form-inline">
    <div class="form-group">...</div>
    <div class="form-group">...</div>
</div>
```

### Buttons

| Class | Use |
|-------|-----|
| `.btn .btn-primary` | Primary action (Continue, Submit) |
| `.btn .btn-secondary` | Secondary action (Back, Cancel) |
| `.btn .btn-success` | Success action |
| `.btn .btn-warning` | Warning action |
| `.btn .btn-small` | Smaller button variant |
| `.btn:disabled` | Disabled state (opacity 0.5, not-allowed cursor) |

### Button Row (navigation)

Always at the bottom of the form, inside the `.card`:

```html
<div class="flex justify-between" style="margin-top: 2rem;">
    <a href="{{ url_for('...') }}" class="btn btn-secondary">Back</a>
    <button type="submit" class="btn btn-primary">Continue</button>
</div>
```

If there's no Back button (first page), use `<span></span>` as spacer to keep Continue right-aligned.

**Do NOT use `<div class="form-actions">`.** That class doesn't exist in the stylesheet.

### Alerts

```html
<div class="alert alert-info">Informational message.</div>
<div class="alert alert-warning">Warning message.</div>
<div class="alert alert-error">Error message.</div>
<div class="alert alert-success">Success message.</div>
```

- `border-radius: 0.5rem`
- Colored border + tinted background matching the alert type

### Radio Group (`.radio-group` + `.radio-option`)

For mutually exclusive choices with descriptions:

```html
<div class="radio-group mb-2">
    <label class="radio-option" onclick="selectRadio(this)">
        <input type="radio" name="choice" value="a" checked>
        <div class="radio-label">
            <h4>Option Title</h4>
            <p>Option description</p>
        </div>
    </label>
    <label class="radio-option" onclick="selectRadio(this)">
        <input type="radio" name="choice" value="b">
        <div class="radio-label">
            <h4>Option Title</h4>
            <p>Option description</p>
        </div>
    </label>
</div>
```

Requires JavaScript to toggle `.selected` class:

```javascript
function selectRadio(element) {
    const radio = element.querySelector('input[type="radio"]');
    radio.checked = true;
    const group = element.closest('.radio-group');
    group.querySelectorAll('.radio-option').forEach(opt => opt.classList.remove('selected'));
    element.classList.add('selected');
}
```

Initialize on page load:

```javascript
document.querySelectorAll('input[type="radio"]:checked').forEach(radio => {
    radio.closest('.radio-option')?.classList.add('selected');
});
```

### Checkbox Group (`.checkbox-group`)

For toggle options (also used for radio pairs in the network page):

```html
<label class="checkbox-group">
    <input type="checkbox" name="option">
    <div class="checkbox-label">
        <h4>Option Title</h4>
        <p>Description</p>
    </div>
</label>
```

### Input Group (`.input-group`)

Input with adjacent button:

```html
<div class="input-group">
    <input type="text" name="field" placeholder="...">
    <button type="button" class="btn btn-secondary">Action</button>
</div>
```

### Address Box (`.address-box`)

Monospace display of addresses/hashes with copy button:

```html
<div class="address-box">
    <span id="address-value">0x1234...abcd</span>
    <button type="button" class="btn btn-small copy-btn" onclick="copyToClipboard('address-value', this)">Copy</button>
</div>
```

### Status Indicator (`.status-indicator`)

Inline status badges:

```html
<div class="status-indicator pending">Pending</div>
<div class="status-indicator success">Valid</div>
<div class="status-indicator error">Failed</div>
<div class="status-indicator loading"><div class="spinner"></div> Loading...</div>
```

### Collapsible Section (`.collapsible`)

```html
<div class="collapsible">
    <div class="collapsible-header" onclick="this.parentElement.classList.toggle('open')">
        <span>Section Title</span>
        <span class="collapsible-arrow">&#9660;</span>
    </div>
    <div class="collapsible-content">
        <!-- content -->
    </div>
</div>
```

### Summary Section (`.summary-section`)

For read-only key-value display (used on the summary page and provisioner summary templates):

```html
<div class="summary-section">
    <h3>Section Name</h3>
    <div class="summary-item">
        <span class="summary-label">Key</span>
        <span class="summary-value">Value</span>
    </div>
</div>
```

## Utility Classes

| Class | CSS |
|-------|-----|
| `.text-center` | `text-align: center` |
| `.text-muted` | `color: var(--text-muted)` |
| `.text-success` | `color: var(--success)` |
| `.text-error` | `color: var(--error)` |
| `.text-warning` | `color: var(--warning)` |
| `.mt-1` | `margin-top: 0.5rem` |
| `.mt-2` | `margin-top: 1rem` |
| `.mb-1` | `margin-bottom: 0.5rem` |
| `.mb-2` | `margin-bottom: 1rem` |
| `.flex` | `display: flex` |
| `.justify-between` | `justify-content: space-between` |
| `.items-center` | `align-items: center` |
| `.gap-1` | `gap: 0.5rem` |
| `.gap-2` | `gap: 1rem` |
| `.hidden` | `display: none !important` |

## Template Blocks

| Block | Purpose |
|-------|---------|
| `{% block title %}` | Page `<title>` |
| `{% block content %}` | Main page content |
| `{% block extra_css %}` | Additional `<style>` in `<head>` (rarely needed) |
| `{% block extra_js %}` | Additional `<script>` before `</body>` |

## Context Variables

Injected automatically by the app's context processor — available in all templates:

| Variable | Type | Source |
|----------|------|--------|
| `wizard_steps` | `list[dict]` | Step definitions: `[{'id': str, 'label': str, 'endpoint': str}, ...]` |
| `session` | Flask session | All wizard data keyed by step |

The `wizard_nav(step_id)` template global returns `{'prev': endpoint_name, 'next': endpoint_name}` for any step.

## Anti-Patterns

These elements are NOT part of the design system. Do not use them:

| Don't | Do instead |
|-------|-----------|
| `<fieldset>` + `<legend>` | `<div class="form-section">` + `<h3>` |
| `<div class="form-actions">` | `<div class="flex justify-between" style="margin-top: 2rem;">` |
| `{{ prev_step_url }}` | `{{ url_for('wizard_blockchain') }}` or `{{ url_for(wizard_nav('step_id').prev) }}` |
| Bare `<h3>` outside `.form-section` | Wrap in `.form-section` for consistent borders/padding |
| Inline `border`, `border-radius` on containers | Use `.card`, `.form-section`, `.alert` classes |
| `<div class="form-row">` | `<div class="two-col">` or `<div class="form-inline">` |
