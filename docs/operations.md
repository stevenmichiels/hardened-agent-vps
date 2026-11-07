# Operations and Recovery

This guide covers steady-state checks, backups, and recovery after the initial
Terraform and Ansible deployment. Run `agent-vps ...` commands on the VPS;
run Terraform and Ansible from the controller machine.

## Readiness Model

`agent-vps status` is the operator-facing source of truth. It checks Tailscale,
backup state, off-box freshness, the retention-pruning gate, disk use, timers,
and the health of enabled optional services. When configured,
`agent-vps healthcheck` sends Telegram alerts only on health-state changes.

## Operator Commands

```sh
agent-vps status
agent-vps timers
agent-vps healthcheck
agent-vps backup
agent-vps backup-offbox
agent-vps docker-cleanup
```

- `status` checks the host, backup state, and enabled optional services.
- `timers` lists scheduled backup, cleanup, and health jobs.
- `healthcheck` runs status checks and sends transition alerts when configured.
- `backup` creates an immediate local archive.
- `backup-offbox` copies and checksum-verifies the latest archive when enabled.
- `docker-cleanup` removes unused Docker artifacts without pruning volumes.

## Backup Model

The backup chain is designed as a closed loop:

1. Runtime configuration and optional service state are collected into the VPS
   backup.
2. Archives are copied off-box with rsync over SSH.
3. The remote copy is verified end-to-end with SHA-256.
4. Local retention pruning is allowed only while the verified off-box state is
   fresh.

Until off-box transport is enabled and verified, `agent-vps status` reports
`backup_retention_prune_disabled_until_offbox_copy` and, when applicable,
`backup_offbox_disabled`. If the off-box target is unavailable, the local
archive remains in place and a retry-pending marker tells the next run which
archive to retry.

A Mac on the same Tailscale network can be a suitable personal off-box target
when it receives already encrypted artifacts. Keep backup archives private;
general VPS archives can contain runtime env files and service credentials.

## Recovery Model

The VPS is replaceable infrastructure. Persistent state should live in Git
remotes, backups, or explicitly documented storage paths.

The high-level recovery sequence is:

1. Recreate the server and firewall with Terraform.
2. Reapply host configuration and service layout with Ansible.
3. Restore optional service state from backup while services are stopped.
4. Validate health, timers, private access, and optional integrations.
5. Close bootstrap SSH and prove ongoing OpenSSH access over Tailscale.

The full runbook includes file ownership and Firecrawl volume caveats:
[Restore runbook](../references/restore.md).

## Optional Ubuntu Pro / ESM Apps

Ubuntu Pro is an optional manual host-maintenance step, not an Ansible default.
Never store its token in Ansible vars, Terraform vars, committed files, support
bundles, screenshots, chat logs, or persistent shell history.

```sh
sudo agent-vps status
sudo agent-vps backup

read -rsp "Ubuntu Pro token: " UBUNTU_PRO_TOKEN; echo
sudo pro attach "$UBUNTU_PRO_TOKEN"
unset UBUNTU_PRO_TOKEN

pro status
sudo apt update
sudo apt upgrade
pro security-status --esm-apps
sudo agent-vps status
sudo agent-vps healthcheck

if [ -f /var/run/reboot-required ]; then cat /var/run/reboot-required; else echo no; fi
```

Reboot only when `/var/run/reboot-required` exists, then rerun `status` and
`healthcheck`. If a token appears in chat, screenshots, logs, or support
artifacts, revoke or rotate it in the Ubuntu Pro dashboard.

## Release Context

See [CHANGELOG.md](../CHANGELOG.md) and the
[v1.0.0 release notes](https://github.com/stevenmichiels/hardened-agent-vps/releases/tag/v1.0.0)
for the deployment-ready workbench baseline.
