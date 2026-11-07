# Security Policy

This repository is a sanitized deployment template. It does not provide a
production-hardening guarantee, and operators are responsible for reviewing
Terraform plans, Ansible changes, firewall exposure, runtime configuration, and
secret handling before use.

## Security Model

This template assumes:

- a single operator or small trusted-admin group, not a multi-tenant host;
- a trusted controller machine for Terraform and Ansible;
- a trusted Tailscale tailnet for ongoing OpenSSH access;
- loopback-bound optional services unless exposure is explicitly accepted;
- NoMachine access over Tailscale rather than public GUI ports;
- secret-bearing runtime files that remain outside Git; and
- private, encrypted backups when artifacts leave the VPS.

It does not protect against a compromised controller, malicious infrastructure
operator, compromised upstream artifact, hostile tailnet administrator, or an
operator who intentionally exposes a private service.

## Non-Negotiable Boundaries

- Restrict bootstrap SSH to the controller's current `/32`, then move OpenSSH
  behind Tailscale in both UFW and the Hetzner firewall.
- Validate non-root access before disabling root SSH or changing firewall rules.
- Keep password authentication disabled.
- Bind Firecrawl, Windmill, and remote desktop to loopback or the Tailscale
  network unless a separate public-ingress design is reviewed.
- Do not apply Terraform when a plan unexpectedly replaces the VPS.
- Do not let coding agents write directly to production runtime directories,
  env files, or databases.

## Secrets and Local Files

Do not commit or share:

- `templates/ansible/inventory.ini`
- `templates/ansible/vars/local.yml`
- `templates/infra/terraform.tfvars`
- `templates/infra/terraform.tfstate*`
- `templates/infra/tfplan*`
- `templates/infra/.terraform/`
- `templates/ansible/ansible-run.log`
- backup archives, OAuth profiles, pairing state, runtime env files, or SSH keys

These paths should be ignored by `.gitignore`. Runtime credentials belong in
protected controller storage or service-specific VPS env files. Do not print
secret values while checking whether they are present.

## Supply Chain

- Review Terraform provider lockfile changes.
- Firecrawl consists of coordinated upstream images that may default to moving
  tags; pin them in ignored local vars for production rollouts.
- Review remote install scripts for Tailscale and Claude CLI, plus OS/npm
  package sources used by optional Codex CLI support.
- Treat NoMachine as an explicit proprietary-package decision.
- Treat every new public route, image bump, and remote install source as a
  reviewable change.

## Sharing Checklist

Before publishing a fork or support bundle:

```sh
git status --short --ignored .
git grep -n -I -E 'BEGIN .*PRIVATE KEY|OPENAI_API_KEY=.+|ANTHROPIC_API_KEY=.+|GITHUB_TOKEN=.+|GH_TOKEN=.+|TELEGRAM_BOT_TOKEN=.+|SLACK_.*TOKEN=.+|FIRECRAWL.*KEY=.+|HCLOUD_TOKEN=.+' -- . || true
git ls-files . | rg '(^|/)(inventory\.ini|terraform\.tfstate|terraform\.tfvars|tfplan|ansible-run\.log|local\.yml)$|\.terraform/' || true
```

The first command should show private deployment files only as ignored. The
last two commands should not reveal real secrets or tracked deployment state.
Extend the search with your usernames, hostnames, account labels, and local
path fragments before publishing.

## Supported Versions

Security fixes are expected on the default branch and the latest tagged release.
Older commits and forks are not actively supported.

## Reporting a Vulnerability

Prefer a private contact method listed on the maintainer's GitHub profile for
vulnerability reports. If no private contact is available, open a GitHub issue
with only non-sensitive reproduction details and ask for a private channel.

Expected initial response time: best effort within 7 days.

Do not include secrets, tokens, private IPs, account IDs, hostnames, chat IDs,
user IDs, OAuth payloads, Terraform state, Ansible inventory, or live
infrastructure details in public reports.

## Operator Responsibilities

Before using this template, operators should review:

- Terraform plans and provider lockfile changes.
- Hetzner firewall exposure and UFW rules.
- SSH bootstrap CIDRs and Tailscale access controls.
- Remote install script tasks in Ansible roles.
- Docker image tags, especially optional Firecrawl images that default to upstream `latest`.
- Runtime env files and backup handling.

Read [Private Access](docs/access.md) and
[Operations and Recovery](docs/operations.md) before enabling their respective
profiles.
