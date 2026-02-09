# blockhost-facts

Single source of truth for interface contracts across the BlockHost ecosystem.

Every BlockHost repo adds this as a submodule. When an interface changes, it changes here. No copies, no drift.

## Contracts

| Document | Covers |
|----------|--------|
| `PROVISIONER_INTERFACE.md` | Provisioner contract: manifest, CLI commands, wizard plugin, root agent actions, first-boot hook |
| `COMMON_INTERFACE.md` | blockhost-common public API: config, VM database, root agent protocol, cloud-init, provisioner dispatcher |

## Usage

Added as a submodule in each repo:

```bash
git submodule add https://github.com/mwaddip/blockhost-facts facts
```

Referenced in each repo's `CLAUDE.md`:

```markdown
## Interface Contracts (REFERENCE)

Read and internalize the relevant contract in `facts/` before modifying any interface boundary.
```
