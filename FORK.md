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
| `Cargo.toml` `workspace.package.version`, `Cargo.lock` | `0.3.38` | `0.3.38-herdr.1`, the fork's release version (see Releases) |

Not changed, on purpose: `install.ps1` (Windows; ACFS does not use it),
`COSIGN_IDENTITY` (used only for upstream's releases before v0.3.31), and the
workflows (see Releases).

Tests:

| File | Change |
|---|---|
| `crates/mcp-agent-mail-core/src/pane_identity.rs` | the caller pane's herdr fallback, tmux first and fail-closed, and off in the HTTP daemon; a herdr pane resolves the record nhm wrote and leaves it unchanged; cleanup keeps herdr records while tmux reports live panes |
| `crates/mcp-agent-mail-guard/tests/guard_env_tests.rs` | `guard_check_full` resolves the committer of a herdr pane, and another agent's reservation still conflicts |
| `crates/mcp-agent-mail-guard/src/lib.rs` | the hook reads a JSON record's name; in a herdr pane its own reservation passes and another agent's blocks |
| `crates/mcp-agent-mail-cli/src/lib.rs` | the self-update fallback hint names the fork's `herdr` installer |
| `FORK.md` | this file |

## Branches

`herdr` is the fork's release line and its default branch: upstream's
latest release tag plus the delta above. `main` mirrors upstream's `main`
and carries none of it. Upstream releases v0.3.31 and later from its own
release infrastructure, not from `main`, so the fork follows upstream's
release tags: at each new upstream release, merge its tag into `herdr` and
cut the next fork release.

## Releases

The fork tags `v<upstream version>-herdr.<n>` (for example
`v0.3.38-herdr.1`) on `herdr`, after the commit that sets that version in
`Cargo.toml` and in `install.sh`'s `FORK_PINNED_VERSION`.

Only `dist.yml` is enabled on the fork; every other workflow is disabled,
so a tag push starts no container, crates or schedule job. `dist.yml` is
upstream's, unchanged: the tag push builds the six targets, signs them with
the workflow's keyless Sigstore identity and stages the 30 assets in a
draft release. Its last step then refuses to publish, because upstream's
ACFS does not pin this fork's installer. The draft is published by hand
after checking what that step would have checked: the tag resolves to the
built revision, the draft holds exactly the 30 staged assets with the
digests the run recorded, and `install.sh` equals the tag's.

Then one more commit on `herdr` sets `FORK_PINNED_SHA256SUMS_SHA256` to the
SHA-256 of the published `SHA256SUMS`. The ACFS fork pins that commit's
`install.sh` in `checksums.yaml`, so its pin covers the binaries. The
installer inside a release, like the one at its tag, has no digest pinned
and refuses to install.

## Removal

Drop a row when upstream resolves herdr panes itself, or keys "the caller's
own pane" on a variable herdr sets. Drop the `read_identity_file()` row when
upstream's hook reads JSON records. When no pane-identity row is left, the
fork ends: ACFS pins upstream's installer again, and the installer, update
and version rows go with the fork.
