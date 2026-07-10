## Why

This role is currently the unmodified `gh-template-ansible-role` scaffold. It needs to become a working installer for HashiCorp Vault (community or enterprise binaries) so it can be consumed like the existing `bradfordwagner.ansible.role.go.releaser.install` and `azure-blob-cli` roles.

## What Changes

- Replace the template scaffold (`defaults/main.yml`, `vars/main.yml`, `tasks/main.yml`, `README.md`) with a real Vault binary installer.
- Support both `community` and `enterprise` Vault builds via a `vault_type` switch that toggles the `+ent` version suffix used in HashiCorp's download URLs.
- Download the platform-appropriate `.zip` from `https://releases.hashicorp.com/vault`, verify it against the published `SHA256SUMS` file fetched at runtime (no hand-maintained checksum tables), and unarchive it into a versioned install directory (`.../vault/<version>/vault`).
- Symlink the versioned binary into a link directory, with `force: true` so re-running with a new version repoints the link.
- Support two install scopes via a single `vault_scope` switch (`system` or `local`): `system` installs under `/usr/local` with `become: true`; `local` installs under `${HOME}/.local` with no privilege escalation.
- Extract the *entire* downloaded archive (not just the `vault` binary) into the versioned install directory, so HashiCorp's bundled license/notice files (`LICENSE.txt` for community; the `LA_*`/`LI_*`/`notices` files for enterprise — confirmed by inspecting real release zips) land alongside the binary with no extra download step or guessed URL.
- Default `vault_version` to a pinned, currently-latest Community release (`2.0.3` as of 2026-07-10) so the role works out of the box without requiring the caller to look up a version first.
- Add a `CLAUDE.md` with an **Upgrades** section documenting how to check for newer Vault releases (`https://releases.hashicorp.com/vault/index.json`) and newer `andrewrothstein.unarchivedeps` releases, and recording that this role's default tracks the latest Community version.
- Bump `meta/requirements.yml`'s `andrewrothstein.unarchivedeps` pin from `3.0.1` to the current latest, `3.2.0`.
- Depend on `andrewrothstein.unarchivedeps` for `unzip` availability, matching the existing role's `meta/requirements.yml`.
- Test matrix in `test.yml` covering all four combinations of `vault_type` (`community`/`enterprise`) x `vault_scope` (`system`/`local`), each with real verification steps (version output, symlink target, license artifact presence, become/no-become behavior).
- `README.md` MUST document every exposed variable (name, default, description) — kept in sync with `defaults/main.yml`.
- Out of scope: license *activation* (`VAULT_LICENSE`/`license_path`/autoload config), systemd unit/service management, GPG signature verification of `SHA256SUMS.sig`, and HSM/FIPS enterprise variants.

## Capabilities

### New Capabilities
- `vault-install`: Download, checksum-verify, and install the HashiCorp Vault binary (community or enterprise) to a versioned directory, then symlink it into a system or user-local bin directory.

### Modified Capabilities
(none — this is the first real implementation of this role)

## Impact

- `defaults/main.yml`, `vars/main.yml`, `tasks/main.yml`, `README.md` — rewritten for Vault instead of the `azure-blob-cli` template leftovers.
- `meta/requirements.yml` — version bump only, `andrewrothstein.unarchivedeps` `3.0.1` → `3.2.0`.
- `CLAUDE.md` — new file, documents the upgrade procedure for both the pinned Vault version and the `andrewrothstein.unarchivedeps` dependency.
- `tests/test.yml` / `test.yml` — updated to exercise all four `vault_type` x `vault_scope` combinations instead of `abc_ver`.
- No external system impact beyond the target host(s) this role is applied to (downloads from `releases.hashicorp.com`, writes to `/usr/local` or `${HOME}/.local`).
