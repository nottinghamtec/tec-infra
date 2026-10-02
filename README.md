# tec-infra

Ansible configuration for TEC's servers. It currently manages one host,
`tec-rigs`, which runs [PyRIGS](https://github.com/NottinghamTEC/PyRIGS)
(`rigs.nottinghamtec.co.uk`) behind nginx with a Let's Encrypt certificate.

> This repository is **public**. Never commit plaintext secrets: anything
> sensitive goes in the Ansible Vault file (see [Secrets](#secrets)).

## What gets configured

| Role                  | Purpose                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------ |
| `users`               | Local users with SSH key login only (passwords locked), optional passwordless sudo         |
| `ssh`                 | Disables SSH password authentication via a drop-in in `/etc/ssh/sshd_config.d/`            |
| `unattended_upgrades` | Automatic reboot at 03:00 (server time) and allows Docker's apt repo                       |
| `docker`              | Docker Engine from Docker's official apt repository                                        |
| `postgres`            | PostgreSQL, the `rigs` role and database, and password login over the local unix socket    |
| `pyrigs`              | The PyRIGS container (`ghcr.io/nottinghamtec/pyrigs:latest`), migrations, static files     |
| `nginx`               | nginx reverse proxy; serves `/static/` directly                                            |
| `certbot`             | Requests the certificate (webroot), renews via `certbot.timer`, reloads nginx on renewal   |

Roles run in the order above (see `playbooks/site.yml`). `users` runs before
`ssh` on purpose, so keys exist before password login is disabled.

## Layout

```
ansible.cfg                       Ansible settings (inventory, vault password file)
inventory/hosts.yml               Hosts (group: servers)
inventory/group_vars/all/*.yml    Variables for every host
inventory/group_vars/all/vault.yml  ENCRYPTED secrets (Ansible Vault)
playbooks/site.yml                The playbook that applies every role
roles/<name>/                     One role per concern
requirements.yml                  Ansible collections
pyproject.toml / uv.lock          Python dependencies (managed with uv)
.pre-commit-config.yaml           Lint hooks (run by prek)
```

## Setup

You need [uv](https://docs.astral.sh/uv/) and SSH access to the host.

```bash
uv sync                                                              # Python deps (ansible-core, ansible-lint, prek)
uv run ansible-galaxy collection install -r requirements.yml -p .ansible/collections
uv run prek install                                                  # git pre-commit hook
```

Then put the **vault password** in `.vault_pass` at the repository root (it is
gitignored). Get it from Bitwarden. `ansible.cfg` points at this
file, so Ansible will refuse to run without it.

## Deploying

Always preview first:

```bash
uv run ansible-playbook playbooks/site.yml --check --diff
```

Apply everything, or just one part with a tag:

```bash
uv run ansible-playbook playbooks/site.yml
uv run ansible-playbook playbooks/site.yml --tags pyrigs
```

Notes on `--check`: some tasks are skipped in a dry run because they depend on
something an earlier task would only create for real (a new apt repo, a newly
installed package, a file that hasn't been written yet). That is expected.

### Releasing a new PyRIGS version

The `pyrigs` role pulls `:latest` and recreates the container only if the image
changed. Re-run:

```bash
uv run ansible-playbook playbooks/site.yml --tags pyrigs
```

On every container start, `manage.py migrate` runs before gunicorn. Static files
are copied out of the image to `/var/www/pyrigs/static` and served by nginx
(falling back to the app). The container restarts automatically (`always`).

## Making changes

1. Edit variables in `inventory/group_vars/all/` or the relevant role.
2. Run `uv run prek run --all-files` (hooks plus `ansible-lint`, `production`
   profile; the commit hook runs the same checks).
3. Preview with `--check --diff` against the host, then apply with the narrowest
   `--tags` that covers the change.
4. Commit.

Common changes:

| To...                     | Edit                                                                                      |
| ------------------------- | ----------------------------------------------------------------------------------------- |
| Add or remove a user      | `users_accounts` in `inventory/group_vars/all/users.yml` (`state: absent` removes a user) |
| Add a docker group member | `docker_users` in `inventory/group_vars/all/docker.yml`                                   |
| Add a website             | `nginx_sites` and `certbot_certificates` in `inventory/group_vars/all/web.yml`            |
| Add a database or DB role | `postgres_roles` / `postgres_databases` in `inventory/group_vars/all/postgres.yml`        |
| Change PyRIGS settings    | `pyrigs_env` (plain) or the vault (secrets); see below                                    |
| Change the reboot time    | `unattended_upgrades_reboot_time`                                                         |
| Change the SSH config     | `roles/ssh/templates/00-ansible-hardening.conf.j2`                                        |

Each role's `defaults/main.yml` documents every variable it accepts.

### Adding a role

```bash
mkdir -p roles/<name>/{tasks,defaults,meta}
```

Add `tasks/main.yml`, `defaults/main.yml` and `meta/main.yml`, and register the
role in `playbooks/site.yml` with a tag. Variables defined in a role need the
role name as a prefix (for example `pyrigs_port`), which `ansible-lint` enforces.

## Secrets

Secrets live in `inventory/group_vars/all/vault.yml`, encrypted with Ansible
Vault. The ciphertext is public, so the vault password must be long and random.

```bash
uv run ansible-vault view inventory/group_vars/all/vault.yml   # read
uv run ansible-vault edit inventory/group_vars/all/vault.yml   # change
```

The vault holds `vault_postgres_rigs_password` (the `rigs` database password) and
`vault_pyrigs_secret_env` (PyRIGS secrets such as `SECRET_KEY`).

- **Rotate the database password:** edit the vault, then run
  `--tags postgres,pyrigs`.
- **Rotate the vault password:** `uv run ansible-vault rekey inventory/group_vars/all/vault.yml`,
  then share the new password with the team and update `.vault_pass`.

## Operations

### PyRIGS container

```bash
docker ps --filter name=pyrigs
docker logs -f pyrigs
docker inspect --format '{{.State.Health.Status}}' pyrigs
```

The environment is in `/etc/pyrigs/pyrigs.env` (root only). The app listens on
`127.0.0.1:8000` and reaches PostgreSQL through the mounted unix socket
(`/var/run/postgresql`). If a migration fails the container will restart in a
loop; the cause will be in `docker logs pyrigs`.

### Certificates

Certbot's systemd timer renews automatically, and a deploy hook reloads nginx.

```bash
certbot certificates
certbot renew --dry-run
systemctl list-timers certbot.timer
```

### Database

```bash
sudo -u postgres psql -d rigs
```

### Automatic updates

`unattended-upgrades` installs security updates (and Docker's) daily and reboots
at 03:00 server time (UTC) when an update requires it. The reboot is skipped if
someone is logged in.

## License

[MIT](LICENSE)
