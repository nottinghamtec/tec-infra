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
| `pyrigs`              | PyRIGS (`ghcr.io/nottinghamtec/pyrigs`) via compose, deploy script and user, cron jobs |
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

PyRIGS runs from a small docker compose project in `/opt/pyrigs`, driven by one
script, `/usr/local/sbin/pyrigs-deploy [tag]`. It pulls the image, copies its
static files to `/var/www/pyrigs/static`, recreates the container and waits for
the health check. If the new container doesn't become healthy, it restores the
previous tag. The deployed tag is recorded in `/opt/pyrigs/deploy.env`.

Releases normally happen automatically (see
[Automatic deployments](#automatic-deployments)). To deploy by hand:

```bash
ssh <you>@tec-rigs sudo pyrigs-deploy sha-1a2b3c4   # a specific commit
ssh <you>@tec-rigs sudo pyrigs-deploy               # redeploy the current tag
```

To roll back, deploy the previous tag the same way. Re-running the Ansible
`pyrigs` role calls the same script with the recorded tag, so it never reverts a
deploy.

On every container start, `manage.py migrate` runs before gunicorn. Static files
are served by nginx (falling back to the app) and old files are kept, so pages
loaded before a deploy still get their assets. The container restarts
automatically (`restart: always`).

### Automatic deployments

CI in the PyRIGS repo builds and pushes `latest` and `sha-<short commit>` images
to GHCR. A final job SSHes to the server as the `deploy` user, whose key can run
**only** `pyrigs-deploy <tag>`: the key is locked to that forced command with
`restrict`, `sudo` allows that one script, and the script only accepts plain tag
characters.

**4. Update the workflow** in the PyRIGS repo (`.github/workflows/ci.yml`). The
`docker` job exposes the exact commit tag that `docker/metadata-action`
generated, and a new `deploy` job uses it:

```yaml
  docker:
    # ...existing job...
    outputs:
      tag: ${{ steps.sha-tag.outputs.tag }}
    steps:
      # ...existing steps, including the `meta` step and the build/push step...
      - id: sha-tag
        name: Pick the commit tag
        env:
          TAGS: ${{ steps.meta.outputs.tags }}
        run: |
          ref="$(grep -m1 ':sha-' <<< "$TAGS")"
          test -n "$ref"
          echo "tag=${ref##*:}" >> "$GITHUB_OUTPUT"

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: [docker]
    if: github.event_name == 'push' && github.ref == 'refs/heads/master'
    environment: production
    concurrency: deploy-production
    steps:
      - name: Deploy the image built for this commit
        env:
          DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
          DEPLOY_KNOWN_HOSTS: ${{ secrets.DEPLOY_KNOWN_HOSTS }}
          TAG: ${{ needs.docker.outputs.tag }}
        run: |
          install -m 700 -d ~/.ssh
          printf '%s\n' "$DEPLOY_SSH_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          printf '%s\n' "$DEPLOY_KNOWN_HOSTS" > ~/.ssh/known_hosts
          ssh deploy@rigs.nottinghamtec.co.uk "$TAG"
```

The tag comes from the metadata step's own `tags` output rather than being
rebuilt from `GITHUB_SHA`, so it stays correct if the tag format changes. It is
passed through an environment variable, not interpolated into the script. The job
fails if the deploy fails, including when the health check fails and the script
rolls back.

Caveats: a deploy replaces the container, so requests are interrupted for a few
seconds. Migrations run on start and are not reversed by a rollback, so keep
them backward-compatible.

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
journalctl -t pyrigs-deploy        # deploy history
cat /opt/pyrigs/deploy.env         # currently deployed tag
docker inspect --format '{{.State.Health.Status}}' pyrigs
```

The environment is in `/etc/pyrigs/pyrigs.env` (root only). The app listens on
`127.0.0.1:8000` and reaches PostgreSQL through the mounted unix socket
(`/var/run/postgresql`). If a migration fails during a deploy, the health
check fails and `pyrigs-deploy` rolls back to the previous tag; the cause will be
in the deploy output or `docker logs pyrigs`. A container that crashes later
restarts automatically.

### Scheduled tasks

`pyrigs_cron_jobs` (in `roles/pyrigs/defaults/main.yml`) defines the cron jobs,
written to `/etc/cron.d/pyrigs`. Each runs `docker exec pyrigs ...` and logs to
the journal. Times are server time (UTC).

| Job       | When            | Command                                                  |
| --------- | --------------- | -------------------------------------------------------- |
| Cleanup   | daily at 00:00  | `manage.py cleanupregistration` then `manage.py usercleanup` |
| Reminders | daily at 08:00  | `manage.py send_reminders`                               |

```bash
journalctl -t pyrigs-cleanup
journalctl -t pyrigs-reminders
```

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
