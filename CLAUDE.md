# Vault Ansible Role

Installs HashiCorp Vault (Community or Enterprise) as a versioned binary with a symlink. See `README.md` for variables and usage.

## Upgrades

Two independent things can go stale and should be checked when revisiting this role:

1. **Vault version default** (`defaults/main.yml`'s `vault_version`): this role defaults to the latest final (non-`alpha`/`beta`/`rc`), non-HSM/FIPS Vault **Community** release. Check `https://releases.hashicorp.com/vault/index.json` for a newer final Community version (a version with no `+` suffix and no `alpha`/`beta`/`rc` in the string) and bump `vault_version` if one exists.
2. **`andrewrothstein.unarchivedeps` role dependency** (`meta/requirements.yml`): check its current latest release/tag and bump the pinned `version` if newer.

Ansible role dependencies are part of "upgrades" for this repo, not just the software the role installs.
