# arosl/mcp_agent_mail_rust: the fork

This fork of
[Dicklesworthstone/mcp_agent_mail_rust](https://github.com/Dicklesworthstone/mcp_agent_mail_rust)
resolves Agent Mail's pane identity in panes of [herdr](https://herdr.dev),
the terminal multiplexer that [nhm](https://github.com/arosl/nhm) drives in
place of tmux and ntm. It is part of the
[ACFS fork](https://github.com/arosl/agentic_coding_flywheel_setup), which
replaces ntm with nhm (ACFS bead acfs-7mu.4).

The fork follows upstream closely. It carries only the changes below, keeps
them small, and merges upstream regularly. It adds no features of its own.

## The delta

Upstream keys "the caller's own pane" on `$TMUX_PANE` alone. In a herdr pane
that variable is unset, so the pre-commit guard cannot name the committer
(agents must export `AGENT_NAME`), `am agents resolve-pane` and
`resolve_pane_identity` need an explicit pane, and a stdio caller's bound
pane cannot stand in for its registration token.

nhm already writes Agent Mail's canonical identity file for every binding it
makes, keyed by herdr's pane id: `w1:p2` is the file
`~/.config/agent-mail/identity/<project_hash>/w1-p2`, holding
`{"name":…,"pane_id":"w1:p2","written_at":…}`. The fork reads that file. It
never writes a herdr pane's file: nhm is its one writer, and it rewrites a
file that disagrees with its binding.

| Where | Upstream | The fork |
|---|---|---|
| core `get_caller_pane_id` (new) | | the tmux pane (`get_composite_tmux_pane_id`), else `$HERDR_PANE_ID`. Like upstream it fails closed when neither is set (the `serve-http` daemon, GH#177) |
| core `mark_serving_http` (new), called by the server's `prepare_http_startup` | | the HTTP daemon (`am serve`, `am serve-http`) never takes its own `$HERDR_PANE_ID` for a caller's pane: an `am serve` started inside a herdr pane would otherwise give that pane's name to every HTTP caller that names no pane. Upstream keeps that behavior for `$TMUX_PANE`, and so does the fork |
| core `resolve_identity_current_pane` (the guard's `am guard check`, `macro_start_session` and the stdio lifecycle check without a pane) | `get_composite_tmux_pane_id` | `get_caller_pane_id` |
| core `write_identity_current_pane` | `get_composite_tmux_pane_id` | unchanged: writes never target a herdr pane |
| core cleanup (`cleanup_stale_identities`, `cleanup_all_stale_identities`) | an unverifiable record whose file name matches no live tmux pane is stale | a record whose `pane_id` is not a tmux `%N` id (nhm's) is never stale by tmux's pane list; nhm removes it when it retires or moves the binding |
| tools `resolve_pane_identity` without `pane_id` | `$TMUX_PANE` | `get_caller_pane_id`; the `MISSING_PANE_ID` message names both variables |
| CLI `am agents resolve-pane` without `--pane` | `$TMUX_PANE` | `get_caller_pane_id`; the error and the `--pane` help name both variables |
| guard hook template, `current_pane_ids()` | `$TMUX_PANE`, else nothing | else `$HERDR_PANE_ID` |
| guard hook template, `read_identity_file()` | the whole file is the name | a JSON record's name is its `name` field. This also fixes upstream: core writes JSON records since GH#252, and the hook took their whole text as the name |

The HTTP transport is unchanged. The `am` CLI still sends only a tmux
`X-Tmux-Pane` header (`%N`), so CLI calls routed to the daemon carry no herdr
pane, and over HTTP only the registration token authorizes a lifecycle
change, as upstream. Agents in herdr panes that call the daemon's tools pass
`pane_id` (their `$HERDR_PANE_ID`) when they need a pane.

An installed hook is a rendered copy of the template: after upgrading, run
`am guard install` again in each clone.

Installer, updates and version:

| Where | Upstream | The fork |
|---|---|---|
| `install.sh` `OWNER` | `Dicklesworthstone` | `arosl`: it downloads the fork's releases |
| `install.sh` `FORK_PINNED_VERSION`, `FORK_PINNED_SHA256SUMS_SHA256` (new) | | the one fork release this installer installs, and the SHA-256 of that release's `SHA256SUMS`. Without `--version` it installs the pinned release, not GitHub's "latest" |
| `install.sh` trust model | minisign over `SHA256SUMS` (v0.3.31 and later), Sigstore before | `pinned` for `vX.Y.Z-herdr.N`: `SHA256SUMS` must have the pinned digest, then each archive must match it. Any other fork version fails closed unless `--no-verify`. Upstream versions keep upstream's models |
| `install.sh` usage comments | upstream's one-liner | the fork's `herdr` branch installer |
| CLI self-update (`am self-update`, `am update`) and its "update available" check | upstream's releases | the fork's releases, so an update never replaces the fork with upstream. The fork's releases carry no minisign signature, so self-update refuses them: update through the installer (ACFS: its pinned one) |
| CLI update hints, the self-update fallback hint, core `AM_INSTALL_REPAIR_COMMAND` | upstream's installer | the fork's `herdr` branch installer |
| `Cargo.toml` `workspace.package.version`, `Cargo.lock` | `0.3.39` | `0.3.39-herdr.3`, the fork's release version (see Releases) |

Not changed, on purpose: `install.ps1` (Windows; ACFS does not use it), and
`COSIGN_IDENTITY` (used only for upstream's releases before v0.3.31). The
release workflow's changes are under Releases.

Tests:

| File | Change |
|---|---|
| `crates/mcp-agent-mail-core/src/pane_identity.rs` | the caller pane's herdr fallback, tmux first and fail-closed, and off in the HTTP daemon; a herdr pane resolves the record nhm wrote and leaves it unchanged; cleanup keeps herdr records while tmux reports live panes |
| `crates/mcp-agent-mail-guard/tests/guard_env_tests.rs` | `guard_check_full` resolves the committer of a herdr pane, and another agent's reservation still conflicts |
| `crates/mcp-agent-mail-guard/src/lib.rs` | the hook reads a JSON record's name; in a herdr pane its own reservation passes and another agent's blocks |
| `crates/mcp-agent-mail-cli/src/lib.rs` | the self-update fallback hint names the fork's `herdr` installer |
| `FORK.md` | this file |

The bead queue:

| Where | Upstream | The fork |
|---|---|---|
| `.beads/` | tracked | not tracked, except its `.gitignore`, which ignores everything. The files stay on disk, and the fork's own work is tracked in this checkout's br. br writes the local user name and checkout path into every exported bead, and the fork is public (the ACFS operator, 2026-10-14). Not part of a release |

## Branches

`herdr` is the fork's release line and its default branch: upstream's
latest release tag plus the delta above. `main` mirrors upstream's `main`
and carries none of it. Upstream releases v0.3.31 and later from its own
release infrastructure, not from `main`, so the fork follows upstream's
release tags: at each new upstream release, merge its tag into `herdr` and
cut the next fork release.

## Releases

The fork tags `v<upstream version>-herdr.<n>` (for example
`v0.3.39-herdr.3`) on `herdr`, after the commit that sets that version in
`Cargo.toml` and in `install.sh`'s `FORK_PINNED_VERSION`.

Only `dist.yml` is enabled on the fork; every other workflow is disabled,
so nothing starts a container, crates or schedule job. A tag push on the
fork starts no run at all (tried twice for `v0.3.38-herdr.1`, which has no
release), so a release is dispatched on its tag. The tagged commit's
`dist.yml` must have the dispatch trigger:

    gh workflow run dist.yml -R arosl/mcp_agent_mail_rust --ref v0.3.39-herdr.3

The run builds the six targets, signs them with the workflow's keyless
Sigstore identity, stages the 30 assets in a draft, verifies the draft and
publishes it as a prerelease.

Then one more commit on `herdr` sets `FORK_PINNED_SHA256SUMS_SHA256` to the
SHA-256 of the published `SHA256SUMS`. The ACFS fork pins that commit's
`install.sh` in `checksums.yaml`, so its pin covers the binaries. The
installer inside a release, like the one at its tag, has no digest pinned
and refuses to install.

Workflow changes:

| File | Change |
|---|---|
| `.github/workflows/dist.yml` | `workflow_dispatch` (no inputs). `release_contract` still requires a `refs/tags/vX.Y.Z[-pre]` ref that matches `Cargo.toml`, so a dispatch on a branch fails there. The `sign` and `release` jobs run on a tag ref, not only on a tag push. Before publishing, upstream checks that upstream's `main` serves this installer and that upstream's ACFS pins its hash; the fork drops that check, since the ACFS fork pins the installer after the release (above) |
| `.github/workflows/dist.yml`, `test` job | a hosted runner could not build the workspace tests (run 38446923873 was killed while compiling, 39 minutes in; upstream's own `ci.yml` has failed on hosted runners since 2026-08). The job first frees runner disk (the preinstalled .NET, Android, GHC and CodeQL), builds the tests without debug info (`CARGO_PROFILE_DEV_DEBUG=0`, `CARGO_INCREMENTAL=0`) and may run 240 minutes. It still runs `cargo test --locked --workspace` |
| `.github/workflows/dist.yml`, `test` job | it installs `br` (beads_rust `=0.6.0`, the version the workspace pins), because the CLI's beads tests run a real `br` and a hosted runner has none. `cargo test` gets `--no-fail-fast`, so every test binary runs, and skips by name the tests that fail or hang at pristine upstream v0.3.39 (run 39979901955, and each skipped test alone at the v0.3.39 tag): the CLI's `doctor_read_only_probe_still_fails_on_unopenable_file` (upstream's br-uwvwz); the server's `ack_ttl` admission tests `promotion_owner_does_not_bypass_ack_scan_admission`, `recovery_contention_preserves_ack_tail_and_resumes_real_escalation` and `shutdown_after_ack_admission_releases_the_generation_lease`, which deadlock (fixed on upstream's main in 04172904); its `ack_ttl::worker` tests `failed_open_retries_the_restored_mailbox_and_publishes_a_real_escalation`, `failed_read_reopens_without_losing_the_tail_or_missing_a_new_ack` and `reopened_pool_restarts_a_changed_generation_before_publishing_a_claim`; and `opt_in_http_sessions_carry_identity_across_requests`; and its TUI tests `action_menu_message_receipts_reach_the_mailbox` and `summarize_action_shows_key_points_and_actions_for_the_selected_thread`, which pass alone but fail in v0.3.39's parallel server suite once another test leaves the process-wide storage write-back queue with unrecoverable errors. Drop each skip when an upstream release fixes it. One more is skipped on the hosted runner only: the CLI's `cli_json_snapshots::cli_json_snapshots`. Its `doctor_check` snapshot differs there (run 40007255333), while the test passes on the fork's tree locally, in the full `it` binary and alone, with this job's br 0.6.0 too. Drop that skip once the runner's output matches |
| `.github/workflows/dist.yml`, `build` job | two steps that never ran on hosted runners before run 40025131151 (v0.3.39-herdr.3): "Install pinned cargo-zigbuild" checks the version with `cargo-zigbuild --version`, since 0.23.3 rejects `cargo zigbuild --version`; "Build release binaries" runs with `shell: bash`, since its bash script failed under Windows' default pwsh. The same run's musl build lost its runner mid-compile (a shutdown signal, exit 143), so on Linux the job first frees runner disk, as the test job does, and the release build runs two rustc jobs at a time (`CARGO_BUILD_JOBS=2`). The release profile is unchanged |
| `tests/docs_drift_ci.rs` | the dist contract matches: `workflow_dispatch` once and without inputs, the tag-ref job condition twice, no ACFS or upstream-main check, 20 `set -euo pipefail`; mutations that remove the tag condition or add dispatch inputs are refused |

## Removal

Drop a row when upstream resolves herdr panes itself, or keys "the caller's
own pane" on a variable herdr sets. Drop the `read_identity_file()` row when
upstream's hook reads JSON records. When no pane-identity row is left, the
fork ends: ACFS pins upstream's installer again, and the installer, update
and version rows go with the fork.
