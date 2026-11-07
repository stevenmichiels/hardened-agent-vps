# Changelog

## Unreleased

## v4.0.1 - 2026-11-07

v4.0.1 is documentation only: lessons from a bootstrap lockout, and a hard
safety rule for the agent user.

- `LEARNINGS.md`: join Tailscale right after the first run, keep the two
  bootstrap `/32` values equal, recover a lockout through the Hetzner web
  console, fix an invalid `hctf` token, and delete saved plans after apply.
- `SKILL.md`: hard safety rule 12 runs coding agents as `agent_user`, never as
  the admin user, which keeps passwordless sudo and Docker on purpose.

Validation:

- `git diff --check`
- Scan of the new `LEARNINGS.md` lines for IP addresses and hostnames

## v4.0.0 - 2026-11-07

v4.0.0 follows a security review: coding agents run as an unprivileged user,
SSH hardening is checked against the effective configuration, Tailscale comes
from its signed apt repository, and the unused Windmill profile is gone. Agent
workspaces and the Claude skill move to the agent user's home, so existing
hosts need the manual steps below.

- Remove the optional Windmill profile: its Ansible role, the `install_windmill`
  and `windmill_*` variables, its status checks, and its backup paths. Its
  worker ran privileged by default. Hosts with Windmill keep `/etc/windmill`,
  `/var/lib/windmill`, and the containers; stop and remove them by hand.
- Add an unprivileged `agent_user` (default `agent`) for Claude Code and Codex,
  without sudo or the `docker` group. The `repos` and `agent-workspaces` zones
  and the `codex-review` Claude skill move to its home. On existing hosts the
  next Ansible run creates the user; checkouts, Claude and Codex logins, and
  `~/.claude/skills/codex-review` under the admin home stay until you move or
  remove them.
- Set SSH hardening in `/etc/ssh/sshd_config.d/00-agent-vps-hardening.conf`,
  validated with `sshd -t`, instead of `lineinfile` edits in
  `/etc/ssh/sshd_config`. sshd keeps the first value it reads, so a drop-in
  such as `50-cloud-init.conf` could override the old edits. The file also sets
  `KbdInteractiveAuthentication no`, and the run fails when `sshd -T` shows
  other effective values.
- Install Tailscale from its signed apt repository instead of piping
  `install.sh` into a root shell. The keyring and source line match what
  `install.sh` writes, so existing hosts see no change.

Validation:

- `ansible-playbook -i inventory.ini.example site.yml --syntax-check` after
  each change
- `bash -n` on the edited `agent-vps` status and backup templates with Jinja
  placeholders replaced
- The SSH drop-in passed a local `sshd -t`, and `sshd -T` printed the lines the
  play asserts
- The Tailscale source line matches the published
  `noble.tailscale-keyring.list`
- `git diff --check`
- Scan for leftover Windmill and admin-home agent paths outside the changelog
  history

## v3.0.1 - 2026-11-07

v3.0.1 keeps fail2ban from locking the operator out of SSH, and records three
bootstrap lessons.

- fail2ban sets `ignoreip` to loopback, `tailscale_ssh_source_cidr`, and
  `bootstrap_public_ssh_cidr` while it is set. SSH is open only to those
  sources, so banning them protected nothing behind key-only login and could
  lock the operator out for an hour. Other sources are still banned.
- `SKILL.md` and `LEARNINGS.md`: check `git status` before an Ansible run, prove
  that root login is off with `sudo sshd -T` instead of a failed root login,
  and disable Tailscale key expiry for the VPS before closing public SSH.
- Existing hosts get the `ignoreip` line at the next Ansible run.

Validation:

- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- The `ignoreip` value rendered with and without `bootstrap_public_ssh_cidr`
- `git diff --check`

## v3.0.0 - 2026-11-07

v3.0.0 makes Claude Code the implementer and Codex the reviewer. The
Codex-to-Claude review direction is gone, so the workbench installs only Claude
skills.

- Make Claude Code the implementer and Codex the reviewer. Remove the
  `claude-review` Codex skill template and the `codex-claude-review` helper,
  with the `install_claude_review_helper`, `claude_review_helper_path`,
  `codex_skills_dir`, and `codex_claude_review_skill_src_dir` variables. The
  `codex-review` Claude skill and `claude-codex-review` stay.
- Hosts provisioned earlier keep `/usr/local/bin/codex-claude-review` and
  `~/.codex/skills/claude-review`; remove them by hand.

Validation:

- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `bash -n templates/claude-skills/codex-review/scripts/claude-codex-review`
- `git diff --check`
- Scan for leftover `codex-claude-review`, `claude-review`, and
  `codex_skills_dir` references outside the changelog history

## v2.0.0 - 2026-11-07

v2.0.0 renames the project to `hardened-agent-vps` and narrows it to a hardened
VPS workbench for coding agents such as Claude Code and Codex. It removes the
optional Hermes Agent, n8n, Cloudflare Tunnel, and OpenClaw pieces and renames
the operator layer, so ignored local vars and hosts from v1.x need changes.

- Rename the repository and README title to `hardened-agent-vps`. GitHub
  redirects the old `dual-agent-vps` URLs.
- Remove the optional Hermes Agent profile: its Ansible role, runtime skill
  templates, URL-prefetch gateway, release check, and status checks.
- Remove the optional n8n profile, its encrypted SQLite backups and restore
  runbook, and the Cloudflare Tunnel role that only served n8n webhooks.
- Remove the `/opt/openclaw` zone directory and the OpenClaw guidance.
- Rename the operator CLI from `hermes-vps` to `agent-vps`, with matching
  systemd units, paths (`/etc/agent-vps`, `/var/backups/agent-vps`,
  `/var/lib/agent-vps-monitor`), and `vps_*` variables instead of `hermes_*`.
- Firecrawl no longer joins an external app Docker network; only its API port
  is published, on loopback.
- Hosts deployed from v1.x keep the old `hermes-vps` units and paths. Rebuild
  them rather than upgrading in place.

Validation:

- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `terraform -chdir=templates/infra fmt -check -diff`
- `terraform -chdir=templates/infra init -backend=false -input=false` and
  `terraform -chdir=templates/infra validate` in CI
- `bash -n` on the operator scripts with Jinja placeholders replaced
- Local Markdown link validation
- `git diff --check`
- Scan for leftover Hermes, n8n, cloudflared, and OpenClaw references outside
  the changelog history

## v1.0.2 - 2026-09-18

v1.0.2 is a documentation-architecture patch that turns the README into a
focused project overview while keeping operational depth in task-specific
guides.

- Reduce the README from 692 lines to a scan-friendly overview centered on the
  motivation, engineering decisions, architecture, capabilities, quick start,
  recovery model, security model, and optional components.
- Move detailed access, operations, n8n/Cloudflare, integration, remote-desktop,
  and dual-agent review guidance into focused documents under `docs/`.
- Expand `SECURITY.md` with the threat model, non-negotiable boundaries, secret
  handling, supply-chain decisions, and pre-sharing checks.
- Use conservative recovery wording, explain why the included operator CLI
  retains the historical `hermes-vps` name, and keep ordinary OpenSSH over
  Tailscale distinct from the optional Tailscale SSH mode.

Validation:

- Local Markdown link validation for `README.md`, `SECURITY.md`, and `docs/*.md`
- `git diff --check`
- `terraform -chdir=templates/infra fmt -check -diff`
- `terraform -chdir=templates/infra init -backend=false -input=false -lockfile=readonly`
- `terraform -chdir=templates/infra validate`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- Public-clean scan for tracked deployment state and secret-like assignments

## v1.0.1 - 2026-05-31

v1.0.1 is a documentation and operator-guidance patch for the v1.0.0 private
workbench baseline. It clarifies the default SSH access model and captures the
manual Ubuntu Pro / ESM Apps maintenance flow proven on the live VPS.

- Document Ubuntu Pro / ESM Apps as an optional manual host-maintenance step,
  including pre-backup, token-handling, update, reboot, and post-healthcheck
  guidance.
- Clarify that the default private access model is ordinary OpenSSH over the
  Tailscale network, with Tailscale SSH disabled unless deliberately chosen.

## v1.0.0 - 2026-05-31

v1.0.0 marks the first deployment-ready shape of `dual-agent-vps` as a private
automation workbench. It keeps the hardened headless VPS and operator layer as
the default, treats Hermes Agent as an optional app profile, and stabilizes the
post-v0.4 architecture around proven backup, restore, off-box verification,
health monitoring, and clear service boundaries.

- Clarify that the base private workbench is the center of the project, with
  Hermes Agent as a first-class optional app profile rather than the default
  conceptual center.
- Split the VPS operator layer into a dedicated `vps_ops` Ansible role while
  keeping the existing `hermes-vps` command and systemd unit names stable.
- Make Hermes Agent opt-in by default (`install_hermes: false`) without losing
  status, backup, healthcheck, off-box verification, release-check, or cleanup
  operator coverage.
- Make `/etc/hermes-vps/.env` the sole source for VPS operator Telegram health
  alerts by removing the legacy fallback to the Hermes app env file.
- Keep the existing `hermes-vps` command and systemd timer names as the runtime
  operator surface, avoiding live-system churn while cleaning up the Ansible
  role tree.
- Reduce Hermes-first wording in README/SKILL docs while keeping historical
  `hermes-vps` runtime names, variables, paths, and app-specific instructions
  stable.
- Live-validate the `vps_ops` split on the VPS with healthy status/healthcheck
  runs, fresh off-box backup verification, and guarded retention pruning.

Validation:

- `git diff --check`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `ansible-playbook -i inventory.ini site.yml --syntax-check`
- `ansible-playbook -i inventory.ini site.yml --check --tags vps_ops`
- `ansible-playbook -i inventory.ini site.yml --tags vps_ops`
- `hermes-vps status`
- `hermes-vps healthcheck`
- Public-clean scan for private domains, Tailscale host IPs, age keys,
  private-key headers, and non-placeholder n8n encryption keys.

## v0.4.0 - 2026-05-31

v0.4.0 closes the n8n public-webhook/private-editor deployment path for
personal workloads. It adds webhook-only Cloudflare Tunnel ingress, private n8n
runtime defaults, encrypted online SQLite backups, off-box backup verification,
guarded local pruning, restore/decrypt proof, expanded status monitoring, and
the remaining WAF/cold-backup/operator documentation.

- Clarify the README with the current public-webhook-only n8n posture,
  encrypted off-box backup chain, guarded prune gate, restore proof, and
  `hermes-vps status` readiness surface.
- Document manual n8n cold full backups for upgrade rollback windows.
- Document manual Cloudflare WAF rate-limit guidance for webhook-only ingress.
- Document that controller-local Cloudflare tunnel credentials should be hidden
  from agent reads with local Codex/Claude deny or exclude rules before
  `cloudflared tunnel login`.
- Align the Cloudflare Tunnel site default binary path with package installs.
- Document the locally managed Cloudflare Tunnel bootstrap flow, including
  hostname prerequisites, controller-side `cloudflared tunnel login/create/route
  dns`, sensitive file handling, first-pass hello-world validation, and
  webhook-only steady state.
- Add an opt-in `cloudflared_install_method: apt_repo` path that configures
  Cloudflare's official apt repository before installing the `cloudflared`
  package.
- Add disabled-by-default Cloudflare Tunnel support for n8n public production
  webhooks while keeping the n8n editor/API private and the VPS closed to
  public `80`, `443`, and `5678`.
- Add disabled-by-default n8n SQLite online backup support using a pinned
  SQLite sidecar image, SQLite `.backup`, integrity checks, encrypted age
  artifacts, and an optional systemd timer.
- Add a disabled-by-default Hermes backup off-box gate with rsync-over-SSH
  upload, remote checksum verification, status freshness checks, and a visible
  retention-prune-disabled sentinel until off-box copy is configured.
- Add an off-box retry-pending marker so a temporarily unavailable personal
  target, such as a Mac over Tailscale, does not discard the local archive.
- Restore Hermes backup retention pruning only when a fresh verified off-box
  state file exists.
- Document a manual Cloudflare WAF rate-limit rule for n8n `/webhook/` ingress
  and a cold full-backup runbook for pre-upgrade n8n safety.
- Tighten n8n templates with a digest-pinned image default, explicit container
  user, explicit backup service user, and an opt-in binary-data inclusion flag
  for online SQLite backups.
- Extend `hermes-vps status` with gated n8n health/container checks and a
  gated `cloudflared` service-active check.
- Document n8n encrypted SQLite restore dry runs, live restore steps, and a
  disposable-credential decrypt proof that avoids printing credential values.
- Add an opt-in private n8n role that is disabled by default, binds to loopback
  or an explicit Tailscale IP only, persists `/var/lib/n8n`, creates
  `/etc/n8n/.env` without overwriting it, and refuses to start until
  `N8N_ENCRYPTION_KEY` is set on the VPS. Include `/etc/n8n` and
  `/var/lib/n8n` in Hermes backups when present.

Validation:

- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `git diff --check`
- `docker manifest inspect docker.n8n.io/n8nio/n8n@sha256:9f1f8e4c093c9924338bd168e3f813f746041d13b337753af0dbdd329e7b50f7`
- Public-clean scan for private domains, Tailscale IPs, age keys, private-key
  headers, and non-placeholder n8n encryption keys.

## v0.3.5 - 2026-05-22

v0.3.5 adds an opt-in NoMachine/XFCE desktop profile while preserving the
headless, private-by-default VPS posture.

- Add an opt-in NoMachine profile that installs the official NoMachine DEB,
  disables UPnP/NAT-PMP port mapping and NoMachine firewall autoconfiguration,
  removes installer-created public UFW rules, requires NX private-key
  authentication, forces virtual desktops to start XFCE, and allows TCP/UDP
  `4000` only from the Tailscale CIDR.
- Make `install_remote_desktop` and `install_nomachine` a paired profile,
  and remove the previous SSH-desktop package defaults.
- Document concrete NoMachine connection values, the expected virtual-display
  prompt, direct Tailscale-only access, and 4 GB minimum / 8 GB preferred
  resource guidance.
- Rename the public repository and README title to `dual-agent-vps`.

Validation:

- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `git diff --check`

## v0.3.4 - 2026-05-15

v0.3.4 makes the Codex-to-Claude review helper more robust in sandboxed
environments.

- Fall back from the default `.codex/reviews/` output path to
  `${TMPDIR:-/tmp}/codex-claude-review-*.md` when the default location is not
  writable.
- Document `CLAUDE_REVIEW_FALLBACK_DIR` for choosing a different fallback
  directory.

Validation:

- `bash -n templates/codex-skills/claude-review/scripts/codex-claude-review`
- `bash templates/codex-skills/claude-review/scripts/codex-claude-review -h`
- Local fake-Claude smoke test for default output
- Local fake-Claude smoke test for read-only default output fallback
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `ansible-playbook -i inventory.ini site.yml --tags agent-review`
- Remote verification of `/usr/local/bin/codex-claude-review -h`
- `git diff --check`

## v0.3.3 - 2026-05-15

v0.3.3 adds the reverse dual-agent review workflow: Claude Code can ask Codex
CLI for a read-only Markdown review through Codex's native review command.

- Add a `codex-review` Claude skill template with a bundled
  `claude-codex-review` helper.
- Install the reverse helper into the VPS operator's `~/.claude/skills` and
  expose `claude-codex-review` on the system PATH.
- Move admin-user creation before user-owned skill installs so fresh VPS
  applies do not depend on the user already existing.
- Add a sharper README positioning sentence for the hardened agent-workbench
  use case.

Validation:

- `bash -n templates/claude-skills/codex-review/scripts/claude-codex-review`
- `bash templates/claude-skills/codex-review/scripts/claude-codex-review -h`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `ansible-playbook -i inventory.ini site.yml --tags agent-review`
- Remote verification of `/home/steven/.claude/skills/codex-review/SKILL.md`
- Local and remote fake-Codex smoke tests for `claude-codex-review`
- `git diff --check`

## v0.3.2 - 2026-05-15

v0.3.2 tightens the release notes for the dual-agent review workflow.

- Explain why `codex-claude-review` matters: it makes review workflows explicit
  and auditable instead of relying on an agent that both changes and judges the
  same code.

## v0.3.1 - 2026-05-15

v0.3.1 fixes the Claude review workflow installation so Codex CLI on the VPS
can discover the `claude-review` skill, not just call the helper command.

- Install the `claude-review` Codex skill into the VPS operator's
  `~/.codex/skills` instead of only installing the `codex-claude-review`
  helper command.

Validation:

- `bash -n templates/codex-skills/claude-review/scripts/codex-claude-review`
- `bash templates/codex-skills/claude-review/scripts/codex-claude-review -h`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- Skill validation for `templates/codex-skills/claude-review`
- `ansible-playbook -i inventory.ini site.yml --tags agent-review`
- Remote verification of `/home/steven/.codex/skills/claude-review/SKILL.md`
- `git diff --check`

## v0.3.0 - 2026-05-15

v0.3.0 adds practical agent-workbench tooling on top of the v0.2.0 VPS model:
Python tool execution with uv/uvx, a controlled Codex-to-Claude review helper,
LangChain docs MCP guidance, and clearer operator access notes.

Highlights:

- Document the preferred Termius phone/tablet SSH key flow: create a
  device-specific Ed25519 key in Termius, add only the public key to
  `admin_authorized_keys`, then rerun Ansible.
- Add Termius key post-add verification guidance: label the key, record its
  fingerprint, verify presence without printing `authorized_keys`, and rename
  the Termius key after a successful login.
- Document that Termius should use OpenSSH over the Tailscale network, and that
  Tailscale SSH intercept may need to be disabled with
  `sudo tailscale set --ssh=false` if Termius hangs on authentication.
- Install the distro `bubblewrap` package when `install_codex_cli=true` so
  Codex CLI can find `bwrap` on Linux VPS hosts instead of warning and falling
  back to its bundled sandbox helper.
- Document adding the official LangChain docs MCP server to user-local Codex
  CLI config on Hermes workbench hosts.
- Add a pinned `uv`/`uvx` workbench baseline with `install_uv`, `refresh_uv`,
  `uv_version`, and `uv_install_dir` knobs; keep Poetry repo-specific.
- Add a `codex-claude-review` helper for the controlled dual-agent workflow:
  one agent develops, Claude Code writes a read-only Markdown review, and the
  reviewer never commits. This makes review workflows explicit and auditable
  instead of relying on an agent that both changes and judges the same code.

Validation:

- `bash -n templates/ansible/roles/base/templates/codex-claude-review.j2`
- `bash templates/ansible/roles/base/templates/codex-claude-review.j2 -h`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `git diff --check`
- `git status --short`

## v0.2.0

v0.2.0 turns the template from a Hermes-only VPS deploy skill into a broader
private AI-agent workbench model. It adds explicit VPS zoning for live apps,
repos, agent workspaces, and service runtimes, plus optional Claude Code CLI,
Codex CLI, and private Firecrawl integration.

This release is the first version where the repository clearly behaves as an
opinionated AI-agent VPS operating model: a safe, rebuildable, agent-ready VPS
for personal or small-team workflows.

Highlights:

- Add an agent-workbench zoning model for app runtimes, staging, repos,
  disposable agent workspaces, OpenClaw, Hermes, Firecrawl, and backups.
- Add optional Claude Code CLI and Codex CLI install toggles to the Ansible base
  role.
- Add a source-controlled Hermes runtime skill for the private Firecrawl stack.
- Add a Hermes gateway URL-prefetch wrapper that hands public URL content to
  Hermes through local artifacts.
- Document Codex CLI access to private Firecrawl through a local MCP server.
- Document Termius access over Tailscale-only SSH.
- Bump the default pinned Hermes image to
  `nousresearch/hermes-agent:v2026.5.7`.
- Add architecture, safe-review, and AI-agent workbench framing to the public
  README.

Validation:

- `terraform -chdir=templates/infra fmt -check -diff`
- `terraform -chdir=templates/infra init -backend=false -input=false -lockfile=readonly`
- `terraform -chdir=templates/infra validate`
- `ansible-playbook -i inventory.ini.example site.yml --syntax-check`
- `git diff --check`
- Secret-pattern scan reviewed; only documentation/helper references were
  present.

## v0.1.0

Initial sanitized deployment template for provisioning and operating Hermes
Agent on a Hetzner VPS with Terraform, Ansible, private-by-default access,
backups, health checks, and restore notes.
