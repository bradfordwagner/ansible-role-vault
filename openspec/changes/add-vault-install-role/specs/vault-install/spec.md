## ADDED Requirements

### Requirement: Version and edition selection
The role SHALL install the Vault version given by `vault_version`, and SHALL append a `+ent` suffix to the version used in all download URLs and install paths when `vault_type` is `enterprise`. When `vault_type` is `community` (the default), no suffix SHALL be applied. `vault_version` SHALL default to the latest known-good Vault Community release at the time this role was last updated, so the role is usable with no required variables.

#### Scenario: Community version resolves without suffix
- **WHEN** `vault_version: 1.17.2` and `vault_type: community`
- **THEN** the resolved download URL and install path use the literal version `1.17.2`

#### Scenario: Enterprise version resolves with +ent suffix
- **WHEN** `vault_version: 1.17.2` and `vault_type: enterprise`
- **THEN** the resolved download URL and install path use the version `1.17.2+ent`

#### Scenario: Defaults install the latest known Community release
- **WHEN** the role is applied with no `vault_version` or `vault_type` override
- **THEN** it installs the pinned default Community version (`2.0.3` as of 2026-07-10) without error

### Requirement: Download and checksum verification
The role SHALL download the Vault `.zip` archive for the host's OS and architecture from `{{ vault_mirror }}/<resolved-version>/vault_<resolved-version>_<os>_<arch>.zip`, SHALL fetch the corresponding `SHA256SUMS` file from the same release directory at runtime, SHALL extract the checksum line matching the archive filename, and SHALL pass that checksum to the download task so an integrity mismatch fails the run.

#### Scenario: Checksum fetched at runtime, not hand-maintained
- **WHEN** the role runs for a Vault version not previously seen by this role
- **THEN** it fetches `SHA256SUMS` for that version from HashiCorp's release server and verifies the downloaded archive against it, without requiring any code change to the role

#### Scenario: Checksum mismatch fails the run
- **WHEN** the downloaded archive's SHA-256 digest does not match the digest published in `SHA256SUMS` for that archive's filename
- **THEN** the download task fails and no install/link steps execute

### Requirement: Versioned install directory
The role SHALL unarchive the downloaded `.zip` into a directory that includes the resolved version (e.g. `<parent_install_dir>/vault/<resolved-version>/`), so multiple versions can coexist on the same host.

#### Scenario: Install directory includes the version
- **WHEN** `vault_version: 1.17.2`, `vault_type: community`, `vault_scope: system`
- **THEN** the `vault` binary is unarchived to `/usr/local/vault/1.17.2/vault`

### Requirement: Bundled license artifacts preserved
The role SHALL extract the entire contents of the downloaded archive into the versioned install directory (not just the `vault` binary), so any license or notice files HashiCorp bundles in that release's archive are present on disk alongside the binary. The role SHALL NOT fetch a license file from any other source (e.g. GitHub) or hard-code an enterprise EULA URL.

#### Scenario: Community license file present after install
- **WHEN** a `community` version is installed (its archive contains `vault` and `LICENSE.txt`)
- **THEN** `<parent_install_dir>/vault/<resolved-version>/LICENSE.txt` exists after the role completes

#### Scenario: Enterprise license/notice files present after install
- **WHEN** an `enterprise` version is installed (its archive contains `vault`, per-language `LA_*`/`LI_*` files, and `notices`)
- **THEN** those files exist in `<parent_install_dir>/vault/<resolved-version>/` alongside `vault` after the role completes

### Requirement: Idempotent install
The role SHALL check whether the versioned binary already exists before downloading, verifying, or unarchiving, and SHALL skip those steps when it does.

#### Scenario: Re-running with the same version is a no-op
- **WHEN** the role is run a second time with the same `vault_version` and `vault_type` as a prior successful run
- **THEN** no download, checksum fetch, or unarchive task reports a change

#### Scenario: Re-running with a new version installs it
- **WHEN** the role is run with a `vault_version` that has no existing install directory on the host
- **THEN** the role downloads, verifies, and unarchives that version to its own versioned directory

### Requirement: Symlink to link directory
The role SHALL create (or update) a symlink named `vault` in `vault_link_dir` pointing at the versioned binary, overwriting any existing symlink at that path.

#### Scenario: Symlink points at the installed version
- **WHEN** the role installs `vault_version: 1.17.2` with `vault_scope: system`
- **THEN** `/usr/local/bin/vault` is a symlink to `/usr/local/vault/1.17.2/vault`

#### Scenario: Upgrading repoints the symlink
- **WHEN** the role is re-run with a different `vault_version` than the currently linked one
- **THEN** the existing symlink at `vault_link_dir` is replaced to point at the newly installed version

### Requirement: System vs. local install scope
The role SHALL expose a single `vault_scope` variable (`system` or `local`) that together determines the parent install directory, the link directory, and whether install/link tasks escalate privilege via `become`. `system` SHALL install under `/usr/local` with `become: true`; `local` SHALL install under `{{ ansible_env.HOME }}/.local` with `become: false`. Each of the derived variables (`vault_parent_install_dir`, `vault_link_dir`, `vault_become`) SHALL remain individually overridable.

#### Scenario: System scope requires no manual become configuration
- **WHEN** `vault_scope: system` (the default)
- **THEN** the role installs to `/usr/local/vault/...`, links at `/usr/local/bin/vault`, and all install/link tasks run with `become: true`

#### Scenario: Local scope requires no privilege escalation
- **WHEN** `vault_scope: local`
- **THEN** the role installs to `{{ ansible_env.HOME }}/.local/vault/...`, links at `{{ ansible_env.HOME }}/.local/bin/vault`, and no install/link task uses `become`

#### Scenario: Individual override wins over scope default
- **WHEN** `vault_scope: system` and `vault_link_dir: /opt/vault/bin` is also set
- **THEN** the symlink is created at `/opt/vault/bin/vault`, while the install directory and `become` still follow the `system` scope default

### Requirement: Documented variables
`README.md` SHALL document every variable defined in `defaults/main.yml`, including its default value and a description of what it controls.

#### Scenario: Every default variable appears in the README
- **WHEN** a variable is added to or changed in `defaults/main.yml`
- **THEN** `README.md`'s variable table is updated to match, so no exposed variable is left undocumented
