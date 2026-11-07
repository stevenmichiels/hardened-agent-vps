# hardened-agent-vps

A reproducible, private-by-default VPS setup for running coding agents and
automation away from my laptop.

I built this because I wanted an always-on environment for Codex/Claude-style
workflows without turning a VPS into an opaque snowflake. The project focuses
on repeatable provisioning, safe access, backups, health checks, and recovery.

**Stack:** Terraform, Ansible, Docker, Tailscale, Hetzner, GitHub Actions.

The main engineering goal is simple: if the VPS disappears tomorrow, I should
be able to rebuild it predictably and understand exactly what is exposed,
persisted, and backed up.

This repository is a sanitized deployment template, not a copy of a live VPS.
For agent-guided setup and operation, use [SKILL.md](SKILL.md).

## Engineering Decisions

- **Terraform owns infrastructure; Ansible owns host configuration.** Keeping
  that boundary explicit makes changes easier to review and recovery easier to
  reason about.
- **Bootstrap SSH narrowly, then move to Tailscale-only access.** Public SSH is
  a temporary provisioning path, not the steady-state access model.
- **Backups are only useful when they can be restored.** The backup flow includes
  off-box verification and documented restore checks.
- **Services stay on loopback or private networks by default.** Public ingress
  is added only for a specific, reviewed use case.
- **Secrets never belong in Git.** Tracked templates contain placeholders;
  credentials and runtime state stay outside the repository.

## Architecture

```text
Controller machine
|-- Terraform -> Hetzner server + firewall
|-- Ansible   -> hardening, Docker, operator tools, optional services
`-- ignored local config
    |-- templates/infra/terraform.tfvars
    |-- templates/ansible/inventory.ini
    `-- templates/ansible/vars/local.yml

Hetzner VPS
|-- OpenSSH restricted to the Tailscale network
|-- /home/<admin_user>/repos and agent-workspaces
|-- /srv/apps for staged and production application zones
|-- /opt and /var/lib for optional service runtimes
|-- hermes-vps health, backup, release, and cleanup commands
`-- /var/backups/hermes-vps
```

The controller holds deployment intent and private local configuration. The VPS
holds runtime state, private services, timers, and backups. Terraform owns the
server and firewall shape; Ansible owns host configuration and service layout.
The included operator CLI is named `hermes-vps`; this historical runtime name
is retained for compatibility with existing commands, timers, paths, and
automation.
See [Private Integrations](docs/integrations.md) for the detailed directory and
service boundaries.

## What It Includes

- Hetzner VPS and firewall provisioning with Terraform.
- Host hardening, UFW, fail2ban, Docker, and Tailscale-aware access with Ansible.
- Separated zones for applications, repositories, agent workspaces, services,
  and backups.
- `hermes-vps` commands for health, backups, off-box verification, timers,
  release checks, alerts, and Docker cleanup.
- Encrypted n8n backups and a documented, verifiable recovery path.
- Optional private n8n, Firecrawl, Windmill, agent gateway, and remote desktop
  profiles.
- Source-controlled Codex, Claude, and runtime skill templates.

## Quick Start

Start with the private headless workbench. Add optional services only after the
base host, Tailscale access, backups, and health checks are working.

### Prerequisites

You need:

- Terraform and Ansible on a trusted controller machine;
- a Hetzner Cloud account and API token;
- an SSH key and your current public `/32` for bootstrap; and
- a Tailscale account for ongoing private access.

Cloudflare, NoMachine, Telegram, and optional service credentials are needed
only when their corresponding profiles are enabled.

### Create Local Configuration

```sh
cp templates/ansible/inventory.ini.example templates/ansible/inventory.ini
cp templates/ansible/vars/local.yml.example templates/ansible/vars/local.yml
cp templates/infra/terraform.tfvars.example templates/infra/terraform.tfvars
```

Replace the examples locally. These files are ignored because they may contain
hostnames, addresses, account details, or deployment-specific settings. Keep
provider tokens, bot tokens, OAuth state, runtime env files, backups, and SSH
keys out of Git. See [SECURITY.md](SECURITY.md).

### Review Without Changing Infrastructure

```sh
terraform -chdir=templates/infra fmt -check -diff
terraform -chdir=templates/infra init -backend=false -input=false -lockfile=readonly
terraform -chdir=templates/infra validate

cd templates/ansible
ansible-playbook -i inventory.ini.example site.yml --syntax-check
```

These checks validate formatting, Terraform configuration, and Ansible syntax
without contacting Hetzner or a live VPS.

### Deploy

1. Set Terraform variables and restrict bootstrap SSH to your current public
   `/32`.
2. Run `terraform plan` and inspect every change.
3. Apply only when the plan matches the intended server and firewall shape.
4. Point the Ansible inventory at the bootstrap IP and apply the base playbook.
5. Prove non-root SSH works before disabling root login.
6. Join the VPS to Tailscale and prove ordinary OpenSSH works over its Tailscale
   IP or MagicDNS name.
7. Remove the public bootstrap CIDR from UFW and the Hetzner firewall.
8. Create the first backup, check timers, and run the status command.
9. Enable optional services one at a time and validate each private boundary.

Do not apply a Terraform plan that unexpectedly replaces an existing VPS. The
exact staged procedure and configuration gates live in [SKILL.md](SKILL.md).

### Verify the Workbench

Run on the VPS:

```sh
sudo hermes-vps backup
sudo hermes-vps status
sudo hermes-vps timers
```

The default access model is OpenSSH over Tailscale, with Tailscale SSH itself
disabled unless deliberately selected. See [Private Access](docs/access.md) for
the transition and Termius setup.

## Recovery and Operations

The VPS is treated as replaceable infrastructure. Terraform rebuilds the cloud
resources, Ansible rebuilds host configuration, and explicit backups restore
persistent service state.

Local retention pruning remains disabled until an off-box archive has been
copied and checksum-verified. Recovery includes an isolated n8n database and
credential-decryption test without printing secret values.

Read [Operations and Recovery](docs/operations.md) and the detailed
[restore runbook](references/restore.md).

## Security Model

The steady-state model is private by default: OpenSSH is limited to Tailscale,
service ports bind to loopback or a Tailscale address, and public ingress is a
separate design decision. The template assumes a trusted controller and a
single operator or small trusted-admin group; it is not a multi-tenant hosting
platform.

Read [SECURITY.md](SECURITY.md) for the threat model, secret boundaries, supply
chain notes, and pre-sharing checks.

## Optional Components

- **n8n and public webhooks:** private editor plus narrowly routed Cloudflare
  Tunnel ingress. [Guide](docs/n8n-cloudflare.md)
- **Firecrawl and MCP integrations:** private host/container endpoints and
  user-local Codex configuration. [Guide](docs/integrations.md)
- **Windmill:** private workflow automation backed by Postgres.
  [Role guide](templates/ansible/roles/windmill/README.md)
- **NoMachine/XFCE:** opt-in GUI access restricted to Tailscale.
  [Guide](docs/remote-desktop.md)
- **Codex/Claude cross-review:** one agent implements while the other reviews
  without edit tools. [Guide](docs/agent-workflows.md)
- **Agent gateway:** optional containerized runtime with state isolated under
  `/var/lib/hermes`; see [SKILL.md](SKILL.md) for enablement and validation.

## Repository Structure

```text
.
|-- SKILL.md                  agent-facing operating procedure
|-- SECURITY.md               threat model and secret boundaries
|-- CHANGELOG.md              release history
|-- docs/                     human-facing guides
|-- references/restore.md     detailed recovery runbook
`-- templates/
    |-- infra/                Terraform server and firewall templates
    |-- ansible/              playbook, roles, and example local config
    |-- codex-skills/         Codex review skill templates
    |-- claude-skills/        Claude review skill templates
    `-- hermes-skills/        optional runtime skill templates
```

## Documentation

- [Private Access](docs/access.md)
- [Operations and Recovery](docs/operations.md)
- [n8n and Cloudflare Tunnel](docs/n8n-cloudflare.md)
- [Private Integrations](docs/integrations.md)
- [Remote Desktop](docs/remote-desktop.md)
- [Agent Workflows](docs/agent-workflows.md)

## What This Is Not

This is not a hosted service, a turnkey SaaS product, or a substitute for
reviewing infrastructure plans and security boundaries. It is an opinionated
deployment template for operators who want reproducible automation and are
comfortable inspecting Terraform, Ansible, and runtime configuration.

## License

Released under the [MIT License](LICENSE).
