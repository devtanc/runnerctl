# runnerctl

A terminal UI and CLI for setting up, monitoring, and managing **self-hosted GitHub Actions runners** on a machine you control.

> **Status:** early planning. Nothing is usable yet. The design and roadmap live in [`.planning/PLAN.md`](.planning/PLAN.md).

## Why

Running GitHub Actions on your own hardware is a great way to get fast, cheap builds (especially macOS builds), but the stock setup is fiddly and easy to get subtly wrong:

- Registering a runner is a manual, copy-and-paste process from a web page, repeated for every repo and every runner.
- The bundled service installer on macOS creates a per-user LaunchAgent. That only starts when the user is logged in, so a headless build machine's runners quietly fail to come back after a reboot.
- Running runners as a dedicated, unprivileged user (which you should) means fighting file permissions, `sudo` prompts, and environment and `PATH` differences between your shell and the background service.
- Getting a consolidated view is hard. "Which runners exist, are they running, are they idle or busy, and what do the logs say?" means jumping between `launchctl`, log files, and the GitHub UI.
- Registration needs GitHub credentials, and it's easy to leave those lying around somewhere the runner user (and therefore every CI job) can read them.

## Goal

`runnerctl` is a single entry point for the whole lifecycle of self-hosted runners on a host. You SSH in, run `runnerctl`, and get:

- **A live dashboard** of every runner on the machine, grouped by repo or organization: service state, GitHub status (idle, busy, or offline), the current job, and host CPU, memory, and disk.
- **Guided setup** for adding runners to a new repo or org: it downloads the correct runner for the host's architecture, verifies the checksum, registers the runner, writes a sane environment, and installs a service that starts at boot with nobody logged in.
- **Day-to-day management:** start, stop, or restart individual runners or whole groups, read logs, and remove runners cleanly from both the machine and GitHub.
- **Adoption** of existing, hand-installed runners.
- **A `doctor` check** that catches permission problems, version mismatches, and drift between what's installed locally and what GitHub knows about.

Every operation is also available as a non-interactive subcommand for scripting.

## Design principles

- **Generic.** Hostnames, users, paths, labels, and toolchains are configuration, not code. Nothing assumes a particular machine or GitHub account.
- **Least privilege.**
  - Runners run as a dedicated unprivileged user.
  - `runnerctl` itself never runs as root.
  - The few operations that need root go through one small, strictly validated helper whose behavior is bounded by root-owned configuration.
  - Day-to-day use doesn't require typing a `sudo` password, but installing or updating the privileged pieces always does.
- **Secrets stay out of reach.** GitHub credentials come from a pluggable secrets provider (an interactive prompt, an environment variable, the `gh` CLI, or [Infisical](https://infisical.com), including self-hosted). They're held only in memory and used only to mint short-lived registration tokens. They never reach the runner user or CI jobs.
- **Stateless and safe to rerun.** All state comes from the service manager, the filesystem, and GitHub, so you can quit and relaunch at any time. Operations are idempotent.

## Platform support

The first target is **macOS (launchd)** on Apple Silicon and Intel. The service-management layer is abstracted so Linux (systemd) support can be added later.

## Roadmap

1. **M1:** core library, privileged helper, bootstrap installer, and CLI (`status`, `add`, `remove`, `start`, `stop`, `adopt`)
2. **M2:** read-only TUI dashboard and log viewer, plus the `gh` and Infisical secrets providers
3. **M3:** TUI actions and the Add and Adopt wizards
4. **M4:** `doctor`, runner update checks, and work-directory cleanup

See [`.planning/PLAN.md`](.planning/PLAN.md) for the full design, the security model, and lessons learned from running runners headless on macOS.

## Security note

Never attach self-hosted runners to **public** repositories. Pull requests from forks could run arbitrary code on your machine. For the same reason, this repository's own CI runs only on GitHub-hosted runners.

## License

TBD.
