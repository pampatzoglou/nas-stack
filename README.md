# nas-stack

The NAS's Docker Compose stack, deployed by [doco-cd](https://github.com/kimdre/doco-cd): it polls
this repository and runs `docker compose up` for `compose.yaml` (project `nas`) on every new
commit. doco-cd itself, monitoring (Prometheus, Loki, Grafana, Alertmanager, Alloy) and Docker are
provisioned by Ansible in
[provision-server](https://github.com/pampatzoglou/provision-server) (`doco_cd` role).

| Path                                    | What                                                    |
|-----------------------------------------|---------------------------------------------------------|
| `compose.yaml`                          | the stack                                               |
| `.doco-cd.yml`                          | doco-cd's deployment config                             |
| `config/homeassistant/configuration.yaml` | Home Assistant's config, bind-mounted read from the repo |
| `config/mosquitto/mosquitto.conf`       | Mosquitto's config, bind-mounted read-only              |

## Host prerequisites

- **Data**: named volumes bind to host directories under `/media/` (`/media/movies`,
  `/media/volumes/<service>`, ...). They must exist before a deploy, or `docker compose up` fails.
  Nothing persistent goes in the repository: doco-cd deploys each revision from its own copy.
- **Secrets**: files under `/opt/secrets/nas-stack/`, root-owned, `0600`. Not in git (the
  repository is public); to move to doco-cd's Bitwarden integration later. Compose mounts them
  with the host file's owner and mode (a secret's `uid`/`gid`/`mode` only apply in Swarm), so
  `cloudflare_token` must be owned by `1000`, the user traefik runs as.

  | File                                   | Used by                 |
  |----------------------------------------|-------------------------|
  | `cloudflare_token`                     | traefik (DNS challenge) |
  | `photoprism_admin_password`            | photoprism              |
  | `photoprism_mariadb_password`          | photoprism, its mariadb |
  | `photoprism_mariadb_root_password`     | photoprism's mariadb    |
  | `prowlarr_api_key`                     | prowlarr                |
  | `duplicati_settings_encryption_key`    | duplicati               |
  | `duplicati_webservice_password`        | duplicati               |
  | `homeassistant_secrets_yaml`           | home-assistant (`secrets.yaml`) |

## Changing the stack

Commit and push to `main`; doco-cd deploys within its poll interval (3 minutes). Failures show in
`docker logs doco-cd` and raise `DocoCdDeploymentFailed` in Alertmanager.

## Image updates

Every image is pinned to `<tag>@<digest>`, the digest of its `linux/amd64` manifest (the NAS's
platform), so a deploy runs exactly the reviewed image. [Renovate](https://github.com/apps/renovate)
(`.github/renovate.json5`) opens PRs on Monday mornings with the new tag and the new amd64
digest; merging one deploys it. linuxserver.io's `<version>-ls<build>` tags have their own
versioning rules there, and MariaDB major updates get a `needs-review` label (data upgrade).
