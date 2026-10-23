# LocalGitTea

Local Gitea stack running in Docker Swarm mode: Traefik (HTTPS reverse proxy), Gitea, PostgreSQL, a Docker registry with UI, and a Gitea Actions runner.

| Service     | URL                              |
|-------------|----------------------------------|
| Gitea       | https://gitea.localhost          |
| Registry    | https://registry.localhost       |
| Registry UI | https://registry-ui.localhost    |
| Git over SSH| `localhost:2222`                 |

Traefik uses a self-signed certificate, so the browser will ask you to accept it once.

## Before the first start

### 1. Change the database password (required)

The compose file ships with a default password, `gitea_secure_password`. Change it in **both** places in `docker-compose.yml` (they must match):

- `gitea` service: `GITEA__database__PASSWD`
- `db` service: `POSTGRES_PASSWORD`

> PostgreSQL only applies `POSTGRES_PASSWORD` when the database volume is created for the first time. If you change it later, either change the password inside Postgres as well or remove the volume (`docker volume rm gitea-stack_gitea_db`, **this deletes all data**).

### 2. Set the runner registration token (`GITEA_RUNNER_REGISTRATION_TOKEN`)

The `gitea-runner` service contains the placeholder `DEIN_GEHEIMER_TOKEN_AUS_GITEA`. Replace it with a real token:

1. Start the stack (see below) and finish the Gitea setup / create an admin user.
2. In Gitea go to **Site Administration → Actions → Runners → Create new runner** (or the org/repo/user settings for a scoped runner) and copy the registration token.
3. Put it into `GITEA_RUNNER_REGISTRATION_TOKEN` in `docker-compose.yml`.
4. Re-deploy the stack (same `docker stack deploy` command as below).

Until then the runner container will fail and restart; this does not affect the other services.

## Start

```bash
# once, if Swarm mode is not active yet
docker swarm init

# deploy (or update) the stack
docker stack deploy -c docker-compose.yml gitea-stack
```

The stack name must be `gitea-stack`: the Traefik service names and the network label in the compose file depend on it.

Check status:

```bash
docker stack services gitea-stack
docker service logs gitea-stack_traefik
```

Allow a minute for all services to start. If you get a 404 or 504 right after deploying, wait and retry.

## Stop

```bash
# remove all services (data volumes are kept)
docker stack rm gitea-stack
```

Optional:

```bash
# delete all data (repos, database, registry images, runner state)
docker volume rm gitea-stack_gitea_data gitea-stack_gitea_db gitea-stack_registry_data gitea-stack_gitea_runner_data

# leave Swarm mode completely
docker swarm leave --force
```

Volumes can only be removed after the stack's containers have fully stopped; wait a few seconds after `docker stack rm`.
