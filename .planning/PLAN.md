# Plan: `runnerctl`, a TUI for managing self-hosted GitHub Actions runners

## Goal

Build **`runnerctl`**, a **generic, open-ended tool** (not tied to any one machine) that is the single entry point for reviewing and managing self-hosted GitHub Actions runners on a host. An admin SSHes in, runs `runnerctl`, and gets a live dashboard of every runner (service-manager state plus GitHub state). From there they can:
- add runners for a repo or org
- start, stop, or restart them
- read their logs
- cleanly remove them

Adding runners covers everything: download, verify, register, write environment files, and install run-at-boot services. The same operations are also available as non-interactive subcommands for scripting and testing.

Key constraints:
- **No repeated sudo password prompts, but it must stay reasonably secure.**
- **GitHub credentials come from a pluggable secrets provider.** Infisical is one supported backend; credentials never live in files the runner user can read.
- **Generic and configurable.** Hostnames, users, paths, labels, toolchain paths, and secrets backends are configuration, never hard-coded. macOS/launchd is the first (and v1-only) platform, behind an abstraction so systemd/Linux can be added later.

The design below is a proposal; please critique it before building.

---

## Local project and versioning

- The project is a git repository whose GitHub remote already exists and is **public**. Nothing in the code, defaults, or config should hard-code a repo URL, owner, hostname, or username.
- **Because the repo is public, its own CI must never run on self-hosted runners.** Fork PRs could execute code on the host. Checks and releases use GitHub-hosted runners only.
- **Versioning:** SemVer starting at `0.1.0`, one shared version for the Cargo workspace (`[workspace.package] version`), annotated git tags like `v0.1.0`, and Conventional Commits.
  - Use **local-first release tooling** that works with no remote, for example `cargo-release` (bump, commit, tag) plus `git-cliff` (generates `CHANGELOG.md` from commits).
  - Tag `v0.1.0` at the end of M1.
- `runnerctl --version` and the TUI header show the version, git short SHA, and build date (via `vergen` or `build.rs`). The root helper also reports a **protocol version** so `doctor` can detect a mismatched install.
- **Include GitHub Actions workflows** (`ci.yml`, `release.yml`) so they work as soon as the repo is pushed, written generically:
  - Checks (`fmt`, `clippy -D warnings`, `test`, `cargo-deny`, `shellcheck`, and a `gitleaks` secret scan) run on GitHub-hosted runners.
  - The release job builds macOS binaries, packages a tarball plus `SHA256SUMS`, and attaches them to the GitHub Release created from a pushed tag.
  - The runner label for macOS release builds is a single variable, default `macos-latest`. Forks or private copies may point it at their own self-hosted runners; this public repo must not.
- `.gitignore` covers local config and credential files (`*.env`, `identity*`, and similar). Add a `gitleaks` pre-commit hook. Fixtures must contain no real tokens, IDs, hostnames, or usernames.

**Layout (suggested):**

```
runnerctl/
  Cargo.toml  Cargo.lock  rust-toolchain.toml  deny.toml  cliff.toml  release.toml
  crates/runnerctl-core/  crates/runnerctl/  crates/runnerctl-helper/
  packaging/
    sudoers.d/runnerctl.in            # template, filled in by bootstrap
    launchd/runner.plist.template     # used only by the root helper
  scripts/bootstrap.sh                # install/update a built or released version
  .planning/PLAN.md                   # this document
  docs/adr/                           # decision records
  docs/lessons-macos.md               # the "hard-won lessons" below, kept current
  CHANGELOG.md  README.md  .gitignore
  .github/workflows/ci.yml  .github/workflows/release.yml
```

---

## Configuration model (everything host-specific lives here)

Two config files with deliberately different trust levels:

**1. Helper config: root-owned, defines the security boundary.**
- Location: `/etc/runnerctl/helper.toml`, owned `root:wheel`, mode `644`, written only by `bootstrap.sh` under interactive sudo.
- It holds:
  - the **service user** that runners run as (for example `ci`)
  - the **runners root** directory (for example `/Users/ci/runners`)
  - the launchd label prefix (default `actions.runner.`)
  - log locations
- The root helper reads **only** this file. It never accepts these values from arguments or from user-writable config, so a compromised admin-side config can't redirect it.

**2. User config: owned by the admin user, holds everything else.**
- Location: `~/.config/runnerctl/config.toml`, mode `600`.
- It holds:
  - runner version pinning and the update policy
  - default labels
  - the runner naming pattern (default `{hostname}-{n}`)
  - the toolchain `PATH` and environment variables written into each runner's `.path`/`.env`
  - polling intervals
  - the secrets provider and its settings
  - scopes the tool knows about (owner, repo or org, and which secret holds each scope's PAT)
- On first run, `runnerctl` offers a setup wizard that writes this file. `doctor` validates it.

The runner directory layout is a convention under the runners root: `<runners_root>/<scope-slug>/<runner-name>/`. The scope slug is the repo name, or the lowercased org name. The runner directory path is baked into the service definition, so it must be final before install.

---

## Privilege model

Three identities: the **admin user** who runs `runnerctl` (has sudo), the **service user** that runners run as (configured), and **root** (only through the helper).

1. **Admin → service user (a step down in privilege):** all download, extract, `config.sh`, `.path`/`.env`, and `runsvc.sh` work, plus narrow reads of runner metadata and logs. This needs a passwordless run-as rule. It's low risk, since the admin already has root and the service user has less.
2. **Admin → root, only through one small root-owned helper binary** (`runnerctl-helper`), installed `root:wheel 755` with every parent directory root-owned and not writable by others (bootstrap verifies this).

The sudoers drop-in is generated from a template by bootstrap, validated with `visudo -cf`, and installed with mode `440`:

```
<admin> ALL=(<service_user>) NOPASSWD: ALL
<admin> ALL=(root) NOPASSWD: /usr/local/libexec/runnerctl-helper
```

- `runnerctl` always uses `sudo -n` so it fails fast instead of prompting.
- **The passwordless rule covers only *running* the helper.** Installing or updating the helper, its template, its config, or the sudoers file always requires interactive sudo through `bootstrap.sh`. `runnerctl` must never be able to replace the helper, or the passwordless rule would become a path to root.
- The TUI and main binary never run as root and never appear in sudoers.

**Root helper rules (this is where the security lives):**
- Fixed subcommands only: `install <scope-slug> <runner-name>`, `remove <label>`, `start|stop|restart <label>`, `list` (JSON), and `version`.
- Strict validation of every argument (for example `^[A-Za-z0-9._-]+$`), rejecting `..`, slashes, and empty values. The label is **derived** by the helper, not passed in.
- **Never trust a service definition written by the service user.** Generate the plist from the root-owned template, with `UserName=<service_user from helper.toml>`, `SessionCreate=true`, `RunAtLoad`, `WorkingDirectory`, log paths, `ProgramArguments=[<runner-dir>/runsvc.sh]`, and `EnvironmentVariables ACTIONS_RUNNER_SVC=1`.
- The runner directory must `realpath` to inside the configured runners root and be owned by the service user. The daemon runs service-user-owned code **as the service user**, so this doesn't escalate privileges. The only escalation risks are the plist's `UserName` and `Label`, which the helper controls.
- Write the plist with `install -o root -g wheel -m 644`, then `launchctl bootstrap system`. Log every action through `logger` or the unified log.
- Never execute, source, or read configuration from service-user-writable or admin-writable files as root (other than `helper.toml`).
- Minimal dependencies, no network access, no secrets.
- Rejected alternatives: Touch ID for sudo (doesn't work over SSH), a longer sudo `timestamp_timeout` (too broad), and running the whole tool as root.

---

## Secrets providers (pluggable)

There's a `SecretsProvider` trait in core. PATs are used **only** to mint short-lived registration or removal tokens and to read runner state from the GitHub API. Only those short-lived tokens ever cross to the service user.

**Providers for v1:**
- **Prompt** (default): hidden input, with nothing stored.
- **Environment variable:** for scripting and CI.
- **`gh` auth:** reuses the admin's `gh auth token`, if present.
- **Infisical:**
  - Universal Auth machine identity against a **configurable domain** (self-hosted or cloud).
  - The bootstrap credential is in a `600` file in the admin's config directory. `runnerctl` refuses to start if the file is group- or world-readable.
  - The provider calls the Infisical REST API directly from Rust, re-authenticating when the short-lived access token expires, and fetches each secret only when it's needed.
  - The `infisical run -- runnerctl …` CLI pattern is documented as an alternative. Direct API access is preferred because the TUI is long-running and `infisical run` only injects secrets once, at startup.
  - **Gotcha:** a machine-identity login doesn't save the instance URL. With no domain set, the CLI silently targets Infisical's US Cloud. Always pass the configured domain explicitly, and fail if it's missing.

**Rules for every provider:**
- Keep secrets in memory only (`secrecy`/`zeroize`). Never put them in the process environment, logs, or crash reports.
- Spawn every child process (service-user calls and the helper) with `env_clear()` plus an explicit allow-list.
- Secrets never go to the service user, runner jobs (which execute repo code), or the helper.

**PAT guidance for the README:**
- Fine-grained PATs target a single resource owner, so plan one per user or org.
- Repo-level runners need repository **Administration: read & write**.
- Org-level runners need organization **Self-hosted runners: read & write**.
- Have the implementation verify the exact permission names.
- Registration tokens expire in about an hour and can be reused within that window. Removal tokens are separate.

---

## `runnerctl` architecture

**Workspace:**
- **`runnerctl-core`** (library): every operation, with side effects behind traits. That includes a `ServiceManager` (launchd now, systemd later), a `CommandRunner` for sudo calls, a `GitHubClient`, a `SecretsProvider`, a filesystem layer, and an `Arch`/`Platform` detector. The core is unit-tested with fakes on any OS.
- **`runnerctl`** (binary): `clap` subcommands for scripting (`status [--json]`, `add`, `remove`, `start`/`stop`/`restart`, `logs`, `doctor`, `config`), and the TUI when run with no arguments.
- **`runnerctl-helper`** (binary): the root helper described above.

**Suggested crates:** `ratatui` + `crossterm`, `tokio`, `clap`, `serde`, `toml`, `plist`, `reqwest` (or `octocrab`), `sysinfo`, `color-eyre`, `tracing` + `tracing-appender` (log to a file, never to the terminal while the TUI is active), and `secrecy`/`zeroize`.

**Core operations:**
- **Add:** choose the scope (repo or org) and how many runners or which names (from the naming pattern), plus extra labels. Then:
  1. Mint the registration token through the secrets provider and the GitHub API.
  2. Download the runner **for the detected architecture** (arm64 vs x64) to a cache, verified against a pinned version and SHA256. Optionally look up the latest release, but always verify the checksum before using it.
  3. For each runner, as the service user:
     - extract
     - `config.sh --url … --token … --name … --labels … --work _work --unattended --replace`
     - write `.path` and `.env` from config
     - copy `bin/runsvc.sh` to the runner root (do **not** use `svc.sh`)
  4. Call the helper's `install` for each runner.
  - Idempotent: skip registration if `.runner` exists, and skip install if the service is already loaded.
- **Remove:** stop and remove the service (helper), then run `config.sh remove` with a removal token, falling back to a REST `DELETE`, then delete the directory.
- **Start, stop, and restart** a runner or a whole scope (useful for capacity).
- **Adopt:** take over existing runners that already follow the directory and label conventions (pre-existing manual installs).
- **Doctor:**
  - sudoers rules present and valid
  - helper and template ownership and modes
  - the helper's protocol version
  - `helper.toml` against the user config
  - secrets provider reachability (including the Infisical domain)
  - PAT scopes
  - whether the service user's toolchain paths exist
  - drift between GitHub and launchd (registered with no service, or a service with no registration)
  - runner version updates

**How the TUI gets its data** (the admin usually can't read the service user's home):
- Service state comes from the helper's `list` JSON (label, PID, last exit status, loaded or not).
- Runner metadata and logs come from narrow reads through `sudo -n -u <service_user>`. Note that `.runner` is JSON **with a UTF-8 BOM**. Logs live at `~<service_user>/Library/Logs/<label>/stdout.log` and `stderr.log`, plus `<runner>/_diag/Runner_*.log`.
- GitHub state (online/offline, busy, labels, ID, and optionally the current job via `runner_name` on in-progress jobs) comes from the REST API.
- Host health comes from `sysinfo` (CPU, RAM, swap, disk) plus each runner's `_work` size.
- Poll the service manager every few seconds and GitHub every 15–30 seconds (configurable), with a manual refresh key.

**Screens:**
- **Dashboard:** runners grouped by scope. Columns for name, service state and PID, GitHub status (Idle/Busy/Offline), labels, and current job. A header with the host name, CPU, RAM, and disk, the secrets provider status, the runner version, and the `runnerctl` version.
- **Detail pane:** paths, labels, ID, version, and the `.path` contents, with a scrollable log tail (stdout, stderr, `_diag`).
- **Actions:** start, stop, or restart; start or stop a whole scope; remove (with confirmation); the Add wizard with a progress view; and Adopt.
- **Later:** a doctor view, a runner update flow, and `_work` cleanup.
- Long operations run asynchronously with progress so the UI stays responsive. Destructive actions always ask for confirmation.
- It's interactive over SSH, not a daemon, and safe to quit and rerun at any time, since all state comes from the service manager, the filesystem, and GitHub.

---

## Hard-won lessons (macOS/launchd; the tool must handle these, and they belong in `docs/lessons-macos.md`)

1. **Use the runner tarball for the host's architecture.** GitHub's "New runner" page defaults to x64. On Apple Silicon without Rosetta, `config.sh` dies with `Bad CPU type in executable`, so nothing gets registered. Detect the architecture; never assume it. Cross-compiling for Intel (for example Tauri's `--target x86_64-apple-darwin` or `universal-apple-darwin`) needs no x64 runner.
2. **`svc.sh install` creates a LaunchAgent, not a daemon.** It writes to `~/Library/LaunchAgents`, refuses to run with sudo, and LaunchAgents only load in a logged-in GUI session, so a never-logged-in service user never starts. Run runners as LaunchDaemons in `/Library/LaunchDaemons` (`root:wheel`, `644`), loaded with `launchctl bootstrap system`. The upstream plist template already includes `UserName` and `SessionCreate`, and `SessionCreate` is needed for keychain and codesign work from a daemon.
3. **A CLI-created, never-logged-in user may be missing `~/Library/LaunchAgents`, and other `~/Library` subfolders may be missing too.** Create what's needed (for example `~/Library/Keychains`, which Tauri uses for its temporary signing keychain).
4. **Globs on the service user's home expand in the caller's shell.** The admin can't read inside it, which produces "no matches found." Do privileged path work inside the target identity's process; never glob across the permission boundary.
5. **`.path` is a snapshot** of PATH taken when `config.sh` runs, and the service uses exactly that. Always write `.path` and `.env` from config (including `HOME=<service home>` and `LANG`), because cargo, rustup, mise, and keychain tooling all depend on them.
6. **Version-manager shims** (for example mise's `~/.local/share/mise/shims`) give a stable PATH entry. Newer mise versions require opting in to `.nvmrc`/`.node-version` (`idiomatic_version_file_enable_tools`).
7. **Once a runner runs as a daemon, `svc.sh start/stop/status` no longer apply.** Use `launchctl kickstart -k`, `bootout`, and `print`.
8. **`config.sh remove` refuses while a `.service` file exists**, and it 404s if the repo was **renamed** after registration (the registration endpoint doesn't follow renames). The REST API does follow renames: list with `GET …/actions/runners` and remove with `DELETE …/actions/runners/{id}`. GitHub also auto-removes runners after 14 days offline.
9. **Scopes:** personal accounts support **repo-level runners only**. Orgs (including Free) support org-level runners through runner groups, and the Default group allows private repos only. Labels look like `actions.runner.<owner>-<repo>.<name>` for repos and `actions.runner.<Org>.<name>` for orgs, preserving the org's casing. The same name can exist in different scopes.
10. **Capacity:** many runners can all start heavy builds at once and exhaust RAM. Make scope-wide stop and start easy, and show memory pressure on the dashboard.
11. **Security:** never attach self-hosted runners to public repos, since fork PRs could run code on the host.
12. **Signing secrets don't belong on the host.** For example, Tauri imports base64 certificates from secrets into a randomly named temporary keychain for each build. Document this pattern in the README; `runnerctl` itself doesn't manage signing.
13. **FileVault and headless operation:** with FileVault on, the host waits for an unlock after a reboot (newer macOS supports pre-boot SSH unlock). LaunchDaemons start right after unlock with nobody logged in. Anything else the host relies on that runs under a hidden user (for example a container VM hosting a secrets server) has the same LaunchAgent-vs-LaunchDaemon problem.

---

## Reference deployment (first real target, used for acceptance testing; **not** baked into the tool)

My first deployment is a headless Apple Silicon Mac mini. The tool must support this setup through configuration alone:

- **Admin user:** has sudo and reaches the machine over SSH with a key. FileVault is on, unlocked over pre-boot SSH. Nobody stays logged in.
- **Service user:** a hidden, standard (non-admin) account with no SecureToken, blocked from SSH and Screen Sharing. Its home is `drwxr-x---`, and `~/Library` is `drwx------`.
- **Toolchains:** rustup (arm64 and x86_64 Apple targets) and mise (Node via shims, corepack enabled), both installed per user. Xcode CLT is system-wide, and Homebrew is owned by the admin.
- **Existing runners:** 10, manually installed (pinned version `2.337.0`, osx-arm64, LaunchDaemons running as the service user), across one org scope and one personal repo scope. They're under `<service home>/runners/<scope-slug>/<name>-N`, all with an extra label shared by every runner. These are the **Adopt** test case.
- **Secrets:** a self-hosted Infisical on the same host, running under its own hidden user in a container VM, reachable on loopback. That's the Infisical provider's test case. Machine identity scoped read-only to one folder, trusted IPs restricted to loopback, short token TTL.
- Do **not** put any of these names, paths, or IDs into code, defaults, or fixtures. They belong only in my local config files.

---

## Milestones and deliverables

0. **Project scaffold:** the layout above, local-first release tooling, CI/release workflows on GitHub-hosted runners, `.gitignore` plus `gitleaks`. The README and this plan (`.planning/PLAN.md`) already exist.
1. **M1:** core, the helper, bootstrap, the sudoers template, the config model, and CLI `status`/`add`/`remove`/`start`/`stop`/`adopt` with the prompt and env providers. Tag `v0.1.0`.
2. **M2:** a read-only TUI dashboard plus log viewer, and the `gh` and Infisical providers.
3. **M3:** TUI actions and the Add and Adopt wizards.
4. **M4:** doctor, runner update checks, and `_work` cleanup.

**First, give me** a critique of this design and a short build plan. Code should be `clippy`-clean, with core logic unit-tested using fakes.

**Acceptance test on the reference deployment:**
- Adopt the existing runners.
- Add runners for a throwaway private repo.
- Confirm the daemons run as the service user (`ps -o user`).
- Run a smoke workflow (`whoami`, `HOME`, `uname -m`, `which cargo node yarn`, `security list-keychains -d user`, `xcrun --find notarytool`).
- Reboot, unlock with nobody logged in, and confirm the dashboard shows everything Idle.
- Remove the throwaway runners and confirm GitHub and launchd are both clean.
