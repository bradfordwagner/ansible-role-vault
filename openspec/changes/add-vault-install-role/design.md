## Context

This role currently ships as the unmodified `gh-template-ansible-role` scaffold (`azure-blob-cli` placeholder names throughout). Two sibling roles already establish the conventions to follow:

- `bradfordwagner.ansible.role.go.releaser.install`: generic installer for GoReleaser-built binaries, hard-coded to GitHub releases, with runtime checksum fetch, versioned install directories, and a `become`-toggle struct.
- This role's own prior scaffold (`azure-blob-cli`): a single-purpose installer with checksums hand-pinned in `defaults/main.yml`.

Vault is not a GoReleaser artifact on GitHub — HashiCorp publishes `.zip` archives directly at `https://releases.hashicorp.com/vault/<version>/vault_<version>_<os>_<arch>.zip`, with enterprise builds distinguished only by a `+ent` suffix on the version string used in every path segment. This role adopts the runtime-checksum-fetch pattern from `go.releaser.install` (avoids hand-maintaining a checksum table per Vault release) but is purpose-built for HashiCorp's URL scheme rather than generic GitHub releases.

## Goals / Non-Goals

**Goals:**
- Install a specific Vault version (community or enterprise) to a versioned directory and symlink it into a bin directory.
- Support both a system-wide install (`/usr/local`, requires `become`) and a user-local install (`${HOME}/.local`, no `become`) via one switch.
- Verify the downloaded archive against HashiCorp's published `SHA256SUMS`, fetched at runtime.
- Be idempotent: re-running with the same version is a no-op; re-running with a new version installs it and repoints the symlink.
- Preserve whatever license/notice files HashiCorp bundles in the release archive itself, alongside the binary, with no separate fetch step.
- Work with zero required variables: default to the latest known-good Vault Community release.

**Non-Goals:**
- License *activation* (`VAULT_LICENSE` env var, `license_path` config, or autoload directory) — this role installs files, it doesn't configure or run `vault server`.
- systemd unit, config file (`config.hcl`), or data directory management — this is a binary installer only, not a service role.
- GPG signature verification of `SHA256SUMS.sig`.
- HSM/FIPS-1402 enterprise variants (different filenames/artifacts entirely).
- Cleanup of previously installed versions left on disk.

## Decisions

- **One version var, computed suffix**: `vault_version` holds the bare version (e.g. `1.17.2`); a `vault_type: community|enterprise` switch drives a computed `vault_full_version` (`1.17.2` or `1.17.2+ent`) used in every URL and path. Alternative considered: require callers to pass the `+ent` suffix themselves in `vault_version` — rejected because it makes `vault_type` redundant and easy to get out of sync (e.g. `+ent` in the version but `vault_type: community`).
- **Single scope switch drives three behaviors**: `vault_scope: system|local` computes `vault_parent_install_dir`, `vault_link_dir`, and `vault_become` together in `defaults/main.yml` via Jinja conditionals, while still leaving each of those three vars individually overridable. Alternative considered: fully independent vars for each (as raw as `go.releaser.install`'s `gri_parent_install_dir` / `gri_link_dir` / `gri_become_user`) — rejected per user preference to reduce the chance of an inconsistent combination (e.g. local dir with `become: true`).
- **Runtime checksum fetch over pinned table**: fetch `vault_<full_version>_SHA256SUMS`, split lines, match the one containing the archive filename, extract the hex digest. Mirrors `go.releaser.install`'s `multi` checksum resolution. Alternative considered: pin checksums per version in `defaults/main.yml` like the old `azure-blob-cli` scaffold — rejected because it requires an edit to this role for every new Vault release, whereas runtime fetch works forever for any version.
- **Idempotency check on the binary path, not a package-manager fact**: `stat` the versioned `vault` binary; skip download/verify/unarchive when it already exists. Matches both reference roles.
- **`unzip` dependency via existing `meta/requirements.yml`**: Vault ships `.zip` (not `.tar.gz`), and `andrewrothstein.unarchivedeps` already installs `unzip` where needed — bump the pin to the current latest release, `3.2.0` (verified against the `andrewrothstein/ansible-unarchivedeps` GitHub tags as of 2026-07-10; was `3.0.1`).
- **License file via unfiltered extraction, not a separate fetch**: inspecting real release archives confirms `vault_2.0.3_linux_amd64.zip` contains `LICENSE.txt` + `vault`, and `vault_2.0.3+ent_linux_amd64.zip` contains per-language `LA_*`/`LI_*` agreement/information files, a `notices` file, + `vault`. Because the `unarchive` task's `dest` is already the whole versioned directory (not a single extracted member), these ship for free — no GitHub-raw fetch, no guessed enterprise EULA URL, no new task type. Alternative considered: fetch a per-tag `LICENSE` file from `raw.githubusercontent.com/hashicorp/vault` — rejected once we confirmed the archive already carries it, and rejected outright for enterprise since `hashicorp/vault-enterprise` is a private repo with no public per-tag URL to point at.
- **Pin `vault_version` to a specific latest-known Community release rather than leaving it required**: default `vault_version: "2.0.3"` (confirmed latest final, non-prerelease Community version via `releases.hashicorp.com/vault/index.json` as of 2026-07-10). Community is the default `vault_type`. This value goes stale as HashiCorp ships new releases, which is why the upgrade procedure is written down in `CLAUDE.md` rather than left tribal. Alternative considered: no default, force callers to always set `vault_version` — rejected because the user wants the role usable with zero required vars out of the box.
- **`CLAUDE.md` Upgrades section**: document two independent things to check when revisiting this role: (1) `https://releases.hashicorp.com/vault/index.json` for a newer final Community version, to bump the `vault_version` default; (2) the `andrewrothstein/ansible-unarchivedeps` releases/tags, to bump the `meta/requirements.yml` pin. This generalizes the existing `role-update` convention (checking upstream tool versions) to also cover Ansible role dependencies, which it didn't previously call out.
- **Test matrix covers the 2x2, not just one path**: `test.yml` exercises `vault_type` (`community`/`enterprise`) crossed with `vault_scope` (`system`/`local`), i.e. 4 scenarios, each asserting: `vault --version` output contains the expected version (and `+ent`/`Enterprise` marker where applicable), the symlink resolves to the correct versioned path, the bundled license artifact(s) exist in the versioned directory, and — for the `local` scenarios specifically — that no task in the run required `become`. Alternative considered: test only `community`/`system` (cheapest) — rejected per explicit request to cover all four combinations with real verification, not just a happy path.

## Risks / Trade-offs

- [HashiCorp changes the `SHA256SUMS` filename/format or the URL layout] → Low likelihood (stable for years); if it happens, only `vars/main.yml`'s URL templates need updating, isolated from the task logic.
- [`+ent` embedded in a directory name is unusual but valid on Linux] → No mitigation needed; `+` is a legal path character, no escaping required in the `file`/`unarchive` modules.
- [No signature verification means a compromised/mirrored HashiCorp CDN response would still pass checksum-only verification if both were tampered together] → Accepted as non-goal for v1, consistent with the existing `go.releaser.install` role's own scope.
- [`vault_scope: local` on a host without a preexisting `${HOME}/.local/bin` on `$PATH`] → Out of scope for this role; `$PATH` setup is the caller's responsibility (same as any user-local install).
- [Pinned `vault_version` default silently drifts behind HashiCorp's actual latest release over time] → Mitigated by the `CLAUDE.md` Upgrades section; still requires someone to actually run the check periodically (the `role-update` skill convention), not automated within the role itself.
- [Extracting the full archive means enterprise installs bring in ~35 small per-language license files most users won't read] → Accepted; the alternative (filtering `unarchive` to only the binary) would require rebuilding license handling per-edition and re-introduces the guessed-URL problem this decision avoids.

## Migration Plan

Not applicable — this replaces an unused template scaffold with no prior consumers. No rollback beyond reverting the commit.
