# GitLab + GitLab Runner with Docker

GitLab (optionally with its container registry) and a GitLab Runner, run by
Docker Compose behind [Traefik](https://github.com/ali-avani/traefik-docker-kubernates),
which provides HTTPS. Install Traefik first.

## Requirements

- Docker with Docker Compose v2
- Traefik on the same server (Docker provider, host network)
- DNS records for the GitLab domain, and the registry domain if used
- Firewall open for ports 80, 443 and the SSH port (default 22)

## Install

```bash
bash ./install.sh
```

It asks for the install type (`gitlab` or `runner`), the GitLab domain and
version, registry domain, time zone, SMTP settings and the `root` password. Then it
writes `.env` and `omnibus_config.rb`, creates the data volumes, starts the stack
and sets the `root` password on a new install.

```bash
bash ./install.sh --yes          # never prompt
bash ./install.sh --no-start     # write the configuration only
bash ./install.sh --regenerate   # write omnibus_config.rb again from .env
bash ./install.sh --update-env   # rebuild .env from .env.sample, keep your values
```

Values are read from the environment or `.env` first, so an unattended install is:

```bash
INSTALL_TYPE=gitlab GITLAB_DOMAIN=gitlab.example.com REGISTRY_DOMAIN=registry.example.com \
  GITLAB_ROOT_PASSWORD='change-me-please' bash ./install.sh --yes
```

| Variable | Description |
| -------- | ----------- |
| `INSTALL_TYPE` | `gitlab` or `runner` |
| `GITLAB_DOMAIN` | Domain of GitLab |
| `GITLAB_VERSION` | Tag with edition suffix, for example `19.4.1-ee.0` |
| `GITLAB_IMAGE` | Image repository, for a mirror (default `gitlab/gitlab-ee`) |
| `REGISTRY_DOMAIN` | Registry domain. Empty = no registry |
| `GITLAB_TIMEZONE` | Time zone (default `UTC`) |
| `GITLAB_SSH_PORT` | Host port for git over SSH (default `22`) |
| `GITLAB_LOW_MEMORY` | `true` = settings for small servers (default) |
| `GITLAB_ROOT_PASSWORD` | `root` password, read on a new install only, then cleared |
| `SMTP_HOST` `SMTP_PORT` `SMTP_USER` `SMTP_PASSWORD` `SMTP_DOMAIN` `SMTP_FROM` `SMTP_VERIFY_NONE` | Email. Empty `SMTP_HOST` = off |
| `RUNNER_VERSION` `RUNNER_IMAGE` | Runner image tag and repository |
| `RUNNER_URL` | GitLab URL for the runner (default `http://gitlab:80`) |
| `RUNNER_TOKEN` | Runner token, read by `--runner` only, then cleared |
| `RUNNER_NAME` `RUNNER_CONCURRENT` `RUNNER_LIMIT` `RUNNER_DOCKER_IMAGE` `RUNNER_PRIVILEGED` `RUNNER_MEMORY` | Runner settings |
| `COMPOSE_FILE` `COMPOSE_PROFILES` | Set by the installer |

To upgrade GitLab, change `GITLAB_VERSION` and run `docker compose up -d`. Follow
GitLab's [upgrade path](https://docs.gitlab.com/update/upgrade_paths/) and back up first.

## GitLab settings

`omnibus_config.rb` holds GitLab's settings. The installer writes it once from
`.env` and keeps it, so you can edit it. `--regenerate` writes it again. Apply
changes with `docker compose up -d --force-recreate gitlab`. The file is
git-ignored because it can hold the SMTP password.

## Runner

1. In GitLab: Admin > CI/CD > Runners > New instance runner. Copy the token (`glrt-...`).
2. Run:

```bash
bash ./install.sh --runner
```

It registers a Docker runner (privileged, image `RUNNER_DOCKER_IMAGE`, memory
limit `RUNNER_MEMORY`) and stores the token in `gitlab-runner/config.toml`
(git-ignored), not in `.env`. For a runner-only server use `INSTALL_TYPE=runner`
and set `RUNNER_URL` to the public GitLab URL. See `examples/runner-config.toml`.

## Registry

With `REGISTRY_DOMAIN` set, the installer adds `docker-compose.registry.yaml` (the
Traefik route) and the registry settings. Images go to
`REGISTRY_DOMAIN/<group>/<project>`. CI jobs use `$CI_REGISTRY`,
`$CI_REGISTRY_IMAGE`, `$CI_REGISTRY_USER` and `$CI_REGISTRY_PASSWORD`.

## Per-server changes

Copy `examples/docker-compose.override.yaml` to `docker-compose.override.yaml`
(git-ignored), uncomment what you need and run `bash ./install.sh --update-env`.
Use it for extra networks, mounts, `extra_hosts` or Traefik labels.

## Backups

The GitLab backup does not include `gitlab.rb` and `gitlab-secrets.json`. Back up both:

```bash
docker exec gitlab gitlab-backup create SKIP=registry
docker cp gitlab:/etc/gitlab /path/to/backup/etc-gitlab
```

## Files

```
install.sh                     installer
docker-compose.yaml            GitLab and runner
docker-compose.registry.yaml   registry route (overlay)
docker-compose.override.yaml   your changes (not committed)
omnibus_config.rb              GitLab settings (not committed)
.env.sample                    variables
gitlab-runner/                 runner config (not committed)
examples/                      override and runner config samples
```

## Troubleshooting

- Traefik shows 404 while GitLab starts. Wait until `docker ps` shows `(healthy)`.
- Git over SSH fails: free the host port (the host's own SSH uses 22) or set
  `GITLAB_SSH_PORT`, and open it in the firewall.
- Logs: `docker logs -f gitlab`, `docker compose logs -f gitlab-runner`.
