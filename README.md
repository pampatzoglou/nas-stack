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
  | `photoprism_oidc_client`               | photoprism (Google SSO client ID) |
  | `photoprism_oidc_secret`               | photoprism (Google SSO client secret) |
  | `photoprism_mariadb_password`          | photoprism, its mariadb |
  | `photoprism_mariadb_root_password`     | photoprism's mariadb    |
  | `prowlarr_api_key`                     | prowlarr                |
  | `duplicati_settings_encryption_key`    | duplicati               |
  | `duplicati_webservice_password`        | duplicati               |
  | `homeassistant_secrets_yaml`           | home-assistant (`secrets.yaml`) |

## Changing the stack

Commit and push to `main`; doco-cd deploys within its poll interval (3 minutes). Failures show in
`docker logs doco-cd` and raise `DocoCdDeploymentFailed` in Alertmanager.

## PhotoPrism Google sign-in

PhotoPrism signs in with Google via OpenID Connect, next to the local `admin` password login, which
stays as the fallback. Self-registration is off: a Google account can sign in only once it is
linked by its subject ID (`sub`), Google's permanent numeric account ID, not the email address.

To add a user:

1. Get their `sub`: have them open the URL below (with `<client-id>` from
   `photoprism_oidc_client`) and pick their account. PhotoPrism shows an error and bounces to its
   login page; the redirect URL with `#id_token=...` is in their browser history (search
   `id_token`). The `sub` claim is in the token's middle, base64url-encoded segment.

   ```text
   https://accounts.google.com/o/oauth2/v2/auth?client_id=<client-id>&redirect_uri=https%3A%2F%2Fphotoprism.xhaos.info%2Fapi%2Fv1%2Foidc%2Fredirect&response_type=id_token&scope=openid+email&nonce=x&prompt=select_account
   ```

   Alternatively, the [OAuth Playground](https://developers.google.com/oauthplayground) with scope
   `openid email` and a GET of `https://openidconnect.googleapis.com/v1/userinfo` returns it.
2. Create the account linked to it (`--role user` for anyone who should not administer):

   ```sh
   docker exec -it photoprism photoprism users add --name "<name>" --email <email> \
     --role admin --auth oidc --auth-id <sub> --auth-issuer https://accounts.google.com <username>
   ```

WebDAV and mobile sync clients cannot use Google sign-in; they need an app password from
Settings → Account → Apps and Devices.

## Image updates

Every image is pinned to `<tag>@<digest>`, the digest of its `linux/amd64` manifest (the NAS's
platform), so a deploy runs exactly the reviewed image. Dependabot (`.github/dependabot.yml`)
opens PRs on Monday mornings with the new tag and digest; merging one deploys it. Minor and patch
updates arrive as one grouped PR; each major update (e.g. a MariaDB data upgrade) gets its own.
