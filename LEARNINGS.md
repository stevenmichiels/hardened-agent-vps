# agent-vps Learnings

This file is for sanitized operational lessons only.

It must not contain real tokens, user IDs, channel IDs, hostnames, OAuth
payloads, IP addresses, private account labels, or local paths.

## Launch Notes

- Load `vars/local.yml` from the controller, not the target VPS. Otherwise
  Ansible may silently fall back to defaults during bootstrap.
- Close bootstrap SSH in two layers after Tailscale is approved and tested:
  update the Hetzner firewall with Terraform, then remove the old bootstrap
  `/32` from UFW with Ansible.
- If a rebuilt VPS reuses an old public IP, SSH may correctly block on a stale
  `known_hosts` entry. Confirm the rebuild, then remove only the stale host
  entry.
- Do not test "root login refused" with a failed root login during bootstrap.
  fail2ban bans an address for an hour after 5 failures in 10 minutes, and the
  bootstrap `/32` is the only allowed SSH source, so a ban locks the operator
  out. Check `sudo sshd -T | grep -i permitrootlogin` as the admin user instead.
- Disable Tailscale key expiry for the VPS node before closing public SSH.
- Check `git status` before an Ansible run. A run that started while another
  session removed a template folder failed midway on the missing source.
- Firecrawl's upstream `nuq-postgres` image expects `POSTGRES_DB=postgres`;
  using another database can leave NUQ tables missing during initialization.
