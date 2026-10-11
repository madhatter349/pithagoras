# Pithagoras on Coolify

How this deployment differs from `docker compose up -d --build` on a plain Docker
host, and what that means for isolation between chats.

Deployed as a Coolify **Docker Compose application** (build pack `dockercompose`)
from this branch, with `docker-compose.coolify.yml` as the compose file.

## Why the compose file is not upstream's

| Change | Reason |
| --- | --- |
| Bridge networking + `expose: 4100`, not `network_mode: host` | Coolify's Traefik terminates HTTPS on a Docker network and cannot route to a host-networked container. The portal is joined to the proxy network instead, and reached through the domain. |
| Host bind mounts for `/data` and `/workspaces` | Both are dedicated directories on the host, so they survive a redeploy and can be moved or inspected without going through Docker. Kept apart from each other: the second is the one a task may write to. |
| A `runner` service running `sleep infinity` | `EXECUTOR=container` launches its *own* containers from `pithagoras-runner:latest`, so nothing references that image and the host's weekly `docker system prune -af` deletes it — silently breaking every task until the next deploy. The service builds the image and keeps a container referencing it. It never runs a task. |
| `deploy.resources.limits` on both services | The host runs ~68 other containers. Without limits one runaway task would compete with all of them. |

Everything else follows upstream: the portal image is built from the unchanged
`Dockerfile`, `init: true` is kept so orphaned shells are reaped, and the Docker
socket is mounted because the container executor launches sibling containers
through it.

## The executor patch

`server/src/executors/index.ts` carries one change, marked with a comment.

Upstream's `ContainerExecutor` passes only `OPENROUTER_API_KEY`,
`ANTHROPIC_API_KEY`, `OPENAI_API_KEY` and `--provider` / `--model` into a task
container. pi's configuration directory is *not* mounted, so a provider set up
under **Settings → Models** — an OpenAI-compatible endpoint, which is the whole
point of that screen — does not exist inside the container, and a task has no
model to run on unless the image happens to bake one in.

The patch mounts the portal's pi configuration directory (`settings.json`,
`models.json`, `auth.json`) read-write at `/pi-agent` and points
`PI_CODING_AGENT_DIR` at it. A task then runs on exactly the provider the portal
is configured with, and adding a provider in the UI reaches new tasks without
rebuilding the image.

Consequences worth knowing:

- The task can **read** every provider key the portal holds, and could rewrite
  the file. It runs with the portal's own uid inside a container that can reach
  nothing else but the Docker socket is not mounted into it. This is not a new
  trust boundary — anyone with the portal password already gets a terminal in
  the portal container, which has the socket — but it is worth stating.
- The mount is translated through the portal's own mounts (`hostMountPath`), as
  the workspace and session mounts already are, so named volumes and nested bind
  mounts work the same way.

## The seeded provider

`deploy/coolify/seed/pi-agent/` holds the `models.json` and `settings.json` placed
in `${PITHAGORAS_DATA_DIR}/home/.pi/agent/` before the first start. They define one
OpenAI-compatible provider, `gateway`, pointing at the self-hosted gateway on this
VPS with `"apiKey": "$OPENAI_API_KEY"` — so the key lives in the compose
environment and never in a file. Any other OpenAI-compatible endpoint can be added
under **Settings → Models**, and because of the patch above it reaches tasks too.

## Isolation between chats — what is actually guaranteed

`EXECUTOR=container` gives **one container per session**, not one workspace per
session. Each task container gets exactly two mounts:

- `<one project directory>:/workspace` — read-write
- `<that session's directory>:/sessions` — read-write

So:

- Two chats in **different** projects cannot see each other's files: neither
  mount overlaps.
- Two chats in the **same** project get two containers that both mount the same
  host directory at `/workspace`. They are separate processes with separate
  resource caps, but they edit **the same files** and can read and overwrite each
  other's work. The container boundary limits what a task can reach *outside* its
  project; it does not partition a project between chats.
- The session directory is per-session, so pi's own conversation files are not
  shared.

Upstream's README states the first half of this ("Use `container` mode or
separate workspaces to run those in parallel") but the wording invites the
assumption that container mode alone settles it. It does not: parallel chats in
one project still need separate workspaces, or the tasks must be serialised.

See `ISOLATION.md` for the transcript of the check that established this on the
live deployment.

## Operating

```bash
# status
ssh-coolify "docker ps --filter name=pithagoras --format '{{.Names}}\t{{.Status}}'"
ssh-coolify "docker inspect -f '{{.State.Health.Status}}' pithagoras"

# what a task container looks like while a chat runs
ssh-coolify "docker ps --filter name=pithagoras- --format '{{.Names}}\t{{.Command}}'"

# data
ls /data/pithagoras/data          # portal.db, sessions/, home/, agents/
ls /data/pithagoras/workspaces    # what tasks may write

# the runner image that EXECUTOR=container launches
docker run --rm pithagoras-runner:latest pi --version
```

Redeploy after an upstream change:

```bash
git fetch upstream && git merge upstream/main   # on this branch
git push origin deploy/coolify                  # Coolify redeploys from the branch
```
