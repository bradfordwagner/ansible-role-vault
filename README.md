# Vault

Installs HashiCorp Vault (Community or Enterprise) by downloading the official `.zip` release from `releases.hashicorp.com`, verifying it against the checksums published for that release, and unarchiving it into a versioned directory. A symlink is created in a bin directory pointing at the versioned binary.

This role is a binary installer only — it does not manage a systemd unit, write `config.hcl`, create a data directory, or handle Enterprise license activation (`VAULT_LICENSE` / `license_path`). Whatever license/notice files HashiCorp bundles inside the release archive (`LICENSE.txt` for Community, the per-language `LA_*`/`LI_*`/`notices` files for Enterprise) are extracted alongside the binary as-is.

## Variables

| Variable | Default | Description |
|---|---|---|
| `vault_version` | `2.0.3` | Bare Vault version to install, without any `+ent` suffix. Check `https://releases.hashicorp.com/vault/index.json` for newer releases. |
| `vault_type` | `community` | `community` or `enterprise`. When `enterprise`, `+ent` is appended to `vault_version` in every download URL and install path. |
| `vault_scope` | `system` | `system` or `local`. Drives the install directory, link directory, and `become` together — `system` installs under `/usr/local` with `become: true`; `local` installs under `{{ ansible_env.HOME }}/.local` with no privilege escalation. Not independently overridable; if you need a different combination, `vault_scope` is the supported way to get it. |
| `vault_mirror` | `https://releases.hashicorp.com/vault` | Base URL to download releases and checksums from. |

## Usage

### Community, system-wide install (defaults)

```yaml
- hosts: all
  roles:
    - role: bradfordwagner.vault
```

Installs the pinned default Community version to `/usr/local/vault/<version>/vault`, symlinked at `/usr/local/bin/vault`, using `become`.

### Enterprise, system-wide install

```yaml
- hosts: all
  roles:
    - role: bradfordwagner.vault
      vault_version: '2.0.3'
      vault_type: enterprise
```

Installs to `/usr/local/vault/2.0.3+ent/vault`, symlinked at `/usr/local/bin/vault`.

### Local, user-scoped install (no become required)

```yaml
- hosts: all
  roles:
    - role: bradfordwagner.vault
      vault_scope: local
```

Installs to `{{ ansible_env.HOME }}/.local/vault/<version>/vault`, symlinked at `{{ ansible_env.HOME }}/.local/bin/vault`, with no privilege escalation.

## Upgrades

See `CLAUDE.md` for the procedure to check for newer Vault releases and newer versions of the `andrewrothstein.unarchivedeps` role dependency.
