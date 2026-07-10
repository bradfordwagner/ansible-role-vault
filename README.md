# Vault

Installs HashiCorp Vault (Community or Enterprise) by downloading the official `.zip` release from `releases.hashicorp.com`, verifying it against the checksums published for that release, and unarchiving it into a versioned directory. A symlink is created in a bin directory pointing at the versioned binary.

This role is a binary installer only — it does not manage a systemd unit, write `config.hcl`, create a data directory, or handle Enterprise license activation (`VAULT_LICENSE` / `license_path`). Whatever license/notice files HashiCorp bundles inside the release archive (`LICENSE.txt` for Community, the per-language `LA_*`/`LI_*`/`notices` files for Enterprise) are extracted alongside the binary as-is.

## Variables

| Variable | Default | Description |
|---|---|---|
| `vault_version` | `2.0.3` | Bare Vault version to install, without any `+ent` suffix. Check `https://releases.hashicorp.com/vault/index.json` for newer releases. |
| `vault_type` | `community` | `community` or `enterprise`. When `enterprise`, `+ent` is appended to `vault_version` in every download URL and install path. |
| `vault_scope` | `system` | `system` or `local`. Drives `vault_parent_install_dir`, `vault_link_dir`, and `vault_become` together (each remains individually overridable). |
| `vault_parent_install_dir` | `/usr/local` (system) or `{{ ansible_env.HOME }}/.local` (local) | Base directory under which `vault/<version>/vault` is installed. |
| `vault_link_dir` | `/usr/local/bin` (system) or `{{ ansible_env.HOME }}/.local/bin` (local) | Directory where the `vault` symlink is created. |
| `vault_become` | `true` (system) or `false` (local) | Whether install/download/link tasks escalate privilege via `become`. |
| `vault_become_method` | `sudo` | `become_method` used when `vault_become` is `true`. |
| `vault_mirror` | `https://releases.hashicorp.com/vault` | Base URL to download releases and checksums from. |
| `vault_arch_map` | `{x86_64: amd64, aarch64: arm64, arm64: arm64}` | Maps `ansible_architecture` to the architecture name HashiCorp uses in its release filenames. |
| `vault_checksum_algo` | `sha256` | Checksum algorithm passed to `get_url`'s `checksum:` parameter. |

## Usage

### Community, system-wide install (defaults)

```yaml
- hosts: all
  roles:
    - role: vault
```

Installs the pinned default Community version to `/usr/local/vault/<version>/vault`, symlinked at `/usr/local/bin/vault`, using `become`.

### Enterprise, system-wide install

```yaml
- hosts: all
  roles:
    - role: vault
      vault_version: '2.0.3'
      vault_type: enterprise
```

Installs to `/usr/local/vault/2.0.3+ent/vault`, symlinked at `/usr/local/bin/vault`.

### Local, user-scoped install (no become required)

```yaml
- hosts: all
  roles:
    - role: vault
      vault_scope: local
```

Installs to `{{ ansible_env.HOME }}/.local/vault/<version>/vault`, symlinked at `{{ ansible_env.HOME }}/.local/bin/vault`, with no privilege escalation.

## Upgrades

See `CLAUDE.md` for the procedure to check for newer Vault releases and newer versions of the `andrewrothstein.unarchivedeps` role dependency.
