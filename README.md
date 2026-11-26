# vps-provisioning-ansible

Turn a freshly created Ubuntu VPS into a hardened, Docker-ready host in one run.

Five roles, applied in order, covering the things that get skipped when a server
is set up by hand: base packages, a non-root deploy user, SSH hardening, a
default-deny firewall, Docker from the official repository with a sane daemon
config, and Let's Encrypt issuance with automatic renewal.

Built to be re-run. Every task is idempotent, so `site.yml` doubles as a drift
correction pass against a box that has been running for months.

## Requirements

- Ubuntu 22.04 or 24.04 (checked at runtime; the play aborts on anything else)
- Ansible 2.16 or newer on the control node
- Root (or sudo) SSH access for the first run
- DNS pointing at the host if you enable certbot

```bash
ansible-galaxy collection install -r requirements.yml
```

## Quick start

```bash
# 1. Copy the example inventory and edit it
cp -r inventory/example inventory/prod
$EDITOR inventory/prod/hosts.yml
$EDITOR inventory/prod/group_vars/vps.yml     # ssh keys, port, domains

# 2. Dry run first. This is always worth the extra minute.
ansible-playbook site.yml -i inventory/prod/hosts.yml --check --diff

# 3. Apply
ansible-playbook site.yml -i inventory/prod/hosts.yml
```

Run a single role:

```bash
ansible-playbook site.yml -i inventory/prod/hosts.yml --tags docker
```

## Roles

| Role | What it does |
|---|---|
| `common` | apt cache, base packages, timezone, chrony, unattended security upgrades |
| `hardening` | deploy user + sudo, sshd config, fail2ban, kernel sysctl baseline |
| `firewall` | ufw default-deny inbound, rate-limited SSH, explicit app ports |
| `docker` | official apt repo, engine + compose plugin, daemon.json, deploy user in `docker` group |
| `certbot` | certbot install, renewal timer, initial standalone issuance |

## Variables

Set in `inventory/<env>/group_vars/vps.yml`.

### Access

| Variable | Default | Notes |
|---|---|---|
| `ssh_port` | `22` | Opens the firewall port and writes `Port` into sshd |
| `deploy_user` | `deploy` | Created on first run, added to `sudo` and `docker` |
| `deploy_ssh_keys` | `[]` | Authorised keys. The play is useless without at least one |
| `disable_root_login` | `true` | `PermitRootLogin no` |
| `disable_password_auth` | `true` | `PasswordAuthentication no`, `AuthenticationMethods publickey` |
| `allowed_tcp_ports` | `[80, 443]` | Additional inbound ports |

### Docker

| Variable | Default | Notes |
|---|---|---|
| `docker_log_max_size` | `50m` | Per-container json-file rotation |
| `docker_log_max_file` | `5` | Rotated files kept |
| `docker_live_restore` | `true` | Containers survive a daemon restart |
| `docker_tcp_bind` | `""` | Leave empty; the socket is the only intended interface |

### Certbot

| Variable | Default | Notes |
|---|---|---|
| `certbot_enabled` | `false` | Off until DNS resolves |
| `certbot_domains` | `[]` | First domain becomes the cert path |
| `certbot_staging` | `false` | Set `true` while testing to avoid rate limits |

## Design notes

**Order matters.** `hardening` depends on `common` for `sudo` and `fail2ban`'s
prerequisites; `docker` depends on `common` and writes to the `deploy_user`
created by `hardening`. Dependencies are declared in each role's `meta/main.yml`
as well as in `site.yml`, so running a role standalone still resolves.

**sshd is validated before it is installed.** The template task uses
`validate: /usr/sbin/sshd -t -f %s`, so a bad config fails the run *before*
`/etc/ssh/sshd_config` is replaced. There is a `backup: true` as a second net.
Lock yourself out of a box with a broken sshd config once and this stops being
theoretical.

**SSH port changes are the one genuinely risky operation.** Raising `ssh_port`
writes the new port into sshd and the firewall in the same run. If your provider
has a network firewall in front of the host, open the new port there *first*, or
you will lose access between the ufw reload and your next connection.

**Kernel forwarding stays off deliberately.** `net.ipv4.ip_forward: 0` is the
baseline; Docker sets the per-interface forwarding it needs. Turning it on
globally is the usual mistake.

**No secrets in the repo.** Nothing here needs a vault. Certbot runs standalone
and does not need DNS credentials; if you later move to a DNS-01 challenge, add
`community.sops` or `ansible-vault` rather than committing tokens.

## Validation

CI runs `yamllint`, `ansible-lint` (production profile) and
`ansible-playbook --syntax-check` on every push.

Locally:

```bash
yamllint .
ansible-lint
ansible-playbook site.yml --syntax-check
```

Linting is not a substitute for `--check` against a real host. The dry run is
the real test.

## Testing against a throwaway host

The cheapest way to exercise this end to end:

```bash
# Any provider: create the smallest instance, add your key, then
ansible-playbook site.yml -i inventory/prod/hosts.yml
# Verify, then destroy
ssh -p 22 deploy@<host> 'sudo ufw status verbose && sudo fail2ban-client status sshd'
```

## Licence

MIT
