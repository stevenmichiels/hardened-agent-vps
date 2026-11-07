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
- Join Tailscale right after the first successful Ansible run. Until then SSH
  depends on one public `/32` in both the Hetzner firewall and UFW, and a new
  home IP or another network locks the operator out with a connect timeout.
- Keep `ssh_source_ips` in `terraform.tfvars` and `bootstrap_public_ssh_cidr`
  in `vars/local.yml` on the same `/32`. When they differ, no address passes
  both layers. Editing tfvars changes nothing until Terraform applies it.
- Recover from a lockout through the Hetzner web console: reset the root
  password under Rescue, log in on the console, run `tailscale up`, test SSH
  over Tailscale, then lock the password again with `passwd -l root`. The
  console often expects a US keyboard layout. Never paste that password into a
  chat or ticket.
- `hctf` reads Keychain item `hcloud-token`. "The token you have provided is
  invalid" means Hetzner does not know the token in that item; replace it
  through the interactive `security add-generic-password -U ... -w` prompt.
- A saved plan (`plan -out=tfplan`) contains the Hetzner token, because the
  provider reads `var.hcloud_token`. Apply it, then delete it right away.
- Check `git status` before an Ansible run. A run that started while another
  session removed a template folder failed midway on the missing source.
- Firecrawl's upstream `nuq-postgres` image expects `POSTGRES_DB=postgres`;
  using another database can leave NUQ tables missing during initialization.
