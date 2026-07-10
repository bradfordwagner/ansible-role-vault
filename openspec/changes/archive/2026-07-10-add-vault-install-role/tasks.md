## 1. Defaults and vars

- [x] 1.1 Rewrite `defaults/main.yml`: `vault_version: "2.0.3"` (latest known-good Community release as of 2026-07-10), `vault_type: community`, `vault_scope: system`, `vault_mirror: https://releases.hashicorp.com/vault`, `vault_arch_map`, `vault_checksum_algo: sha256`, `vault_become_method: sudo`, and scope-derived `vault_parent_install_dir` / `vault_link_dir` / `vault_become` (Jinja conditionals on `vault_scope`, individually overridable).
- [x] 1.2 Rewrite `vars/main.yml`: computed `vault_full_version` (append `+ent` when `vault_type == enterprise`), `vault_os` (`ansible_system | lower`), `vault_arch` (via `vault_arch_map[ansible_architecture]`), archive filename, download URL, checksum URL, versioned install dir, installed exe path, installed link path, temp zip path.

## 2. Tasks

- [x] 2.1 Rewrite `tasks/main.yml`: `include_role: andrewrothstein.unarchivedeps`, then `stat` the versioned binary and register it.
- [x] 2.2 Add a `when: not <binary>.stat.exists` block: fetch `SHA256SUMS` via `uri`, parse out the line/digest matching the archive filename, `get_url` the zip with that checksum (respecting `vault_become`), `mkdir` the versioned install dir, `unarchive` the **entire** zip into it (do not scope `unarchive` to a single member — this is what makes `LICENSE.txt` / `LA_*`/`LI_*`/`notices` land alongside `vault` automatically).
- [x] 2.3 Add `always:` cleanup of the temp zip file.
- [x] 2.4 Add the final `file: state=link, force=true` task linking `vault_link_dir/vault` to the versioned binary (respecting `vault_become`).

## 3. Role metadata and docs

- [x] 3.1 Update `meta/main.yml` description/tags for Vault (keep `andrewrothstein.unarchivedeps` dependency).
- [x] 3.2 Bump `meta/requirements.yml`'s `andrewrothstein.unarchivedeps` pin from `3.0.1` to `3.2.0` (current latest as of 2026-07-10).
- [x] 3.3 Rewrite `README.md` with a complete variable table (`vault_version`, `vault_type`, `vault_scope`, `vault_parent_install_dir`, `vault_link_dir`, `vault_become`, `vault_become_method`, `vault_mirror`, `vault_arch_map`, `vault_checksum_algo`) — every variable in `defaults/main.yml` MUST appear here with its default and description — plus community/enterprise/local-scope usage examples.
- [x] 3.4 Clear `handlers/main.yml`'s stale `azure-blob-cli` comment.
- [x] 3.5 Create `CLAUDE.md` with an **Upgrades** section documenting:
  - This role defaults `vault_version` to the latest final (non-`alpha`/`beta`/`rc`), non-HSM/FIPS Vault **Community** release. Check `https://releases.hashicorp.com/vault/index.json` for a newer one and bump `defaults/main.yml`.
  - Ansible role dependencies are also part of "upgrades": check `meta/requirements.yml`'s `andrewrothstein.unarchivedeps` pin against its current latest tag/release and bump if newer.

## 4. Tests

- [x] 4.1 Rewrite `test.yml` to run all four combinations of `vault_type` (`community`/`enterprise`) x `vault_scope` (`system`/`local`), each as its own **play** (not just role entry) so verification runs immediately after its role invocation, before a later scenario sharing the same `link_dir` overwrites the symlink. Deviation from the original wording: deleted the stale `tests/test.yml` + `tests/inventory` (leftover `azure-blob-cli`/`remote_user: root` scaffold, not used by any sibling role) and added `tests/verify.yml` as a shared `include_tasks` verification file, matching `go.releaser.install`'s actual convention (`test.yml` + `tests/<helper>.yml`, no duplicate playbook).
- [x] 4.2 Each scenario's `tests/verify.yml` include verifies: `command: <exe> --version` succeeds and its output contains the expected resolved version (including `+ent` for enterprise — confirmed real output is `Vault v2.0.3+ent (...)`); the symlink at `<link_dir>/vault` resolves (via `stat`, checking `lnk_source`) to the expected versioned binary path; the expected bundled license artifact(s) exist in the versioned directory (`LICENSE.txt` for community; `LA_*` files + `notices` for enterprise — confirmed by inspecting real release zips).
- [x] 4.3 For the two `vault_scope: local` scenarios, `tests/verify.yml` asserts the symlink's owner matches `ansible_env.USER` (i.e. not root-owned via unwanted privilege escalation).
- [x] 4.4 Ran `test.yml` twice against this machine (community/system, community/local, enterprise/system, enterprise/local — all 4 real HashiCorp releases downloaded and verified). First run: `changed=20, failed=0`. Second run: `changed=4, failed=0`, and all 4 changes were the symlink-repoint tasks only (expected — community and enterprise share a link dir per scope, so each full test run flips the shared symlink at the end; this is the "Upgrading repoints the symlink" scenario, not an idempotency violation). Download, checksum-fetch, `mkdir`, and `unarchive` tasks showed zero changes on the second run — idempotency confirmed. Test-installed files were removed from this machine afterward.

## 5. Validation

- [x] 5.1 No `.yamllint`/`.ansible-lint` config exists anywhere in this repo or in the sibling `go.releaser.install`/`azure-blob-cli` reference roles (which themselves fail default yamllint's 80-char/truthy-value rules) — nothing configured to run, consistent with established practice, not a new gap.
- [x] 5.2 Confirmed: `defaults/main.yml` and `README.md`'s variable table both list exactly the same 10 variables (`vault_arch_map`, `vault_become`, `vault_become_method`, `vault_checksum_algo`, `vault_link_dir`, `vault_mirror`, `vault_parent_install_dir`, `vault_scope`, `vault_type`, `vault_version`).

## 6. Reduce exposed variable surface

- [x] 6.1 `defaults/main.yml`: keep only `vault_version`, `vault_type`, `vault_scope`, `vault_mirror`. Removed `vault_parent_install_dir`, `vault_link_dir`, `vault_become`, `vault_become_method`, `vault_arch_map`, `vault_checksum_algo` as exposed/overridable variables.
- [x] 6.2 `vars/main.yml`: moved the scope-derived computation (parent install dir, link dir, `become`) here as internal values (no longer individually overridable), and added `vault_arch_map` (constant) and `vault_checksum_algo: sha256` (constant) here instead of `defaults/main.yml`.
- [x] 6.3 `tasks/main.yml`: hardcoded `become_method: sudo` on every task instead of referencing `vault_become_method`.
- [x] 6.4 `README.md`: trimmed the variable table to the 4 remaining variables; usage examples didn't reference any removed variable, so no changes needed there.
- [x] 6.5 Re-ran `test.yml` locally: `ok=106, changed=20, failed=0` — all 4 scenarios (community/enterprise x system/local) still pass identically. Pushed and confirmed CI green.
