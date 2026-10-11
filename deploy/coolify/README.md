# Pithagoras on Coolify

How this deployment differs from `docker compose up -d --build` on a plain Docker
host, and what that means for isolation between chats.

Deployed as a Coolify **Docker Compose application** (build pack `dockercompose`)
from this branch, with `docker-compose.coolify.yml` as the compose file.

| | |
| --- | --- |
| URL | <https://pithagoras.382972.xyz> |
| Coolify app | `usn7v2ezjed2ux5ywbwzhiv1` (project `ce2wkyjoovbfsmm9gktljaw9`) |
| Repo / branch | `madhatter349/pithagoras` @ `deploy/coolify` |
| Data | `/data/pithagoras/data` on the host → `/data` |
| Workspaces | `/data/pithagoras/workspaces` on the host → `/workspaces` |
| Executor | `container` — one throwaway container per task |
| Provider | `clinepass` (api.cline.bot), model shown as **DeepSeek V4.1 Flash-Wafer** |
| Local credentials | `/data/keys/pithagoras.env` (host-side, mode 600) |

## Why the compose file is not upstream's

| Change | Reason |
| --- | --- |
| Bridge networking + `expose: 4100`, not `network_mode: host` | Coolify's Traefik terminates HTTPS on a Docker network and cannot route to a host-networked container. The portal joins the proxy network instead and is reached through the domain. |
| Host bind mounts for `/data` and `/workspaces` | Both are dedicated directories on the host, so they survive a redeploy and can be moved or inspected without going through Docker. Kept apart from each other: the second is the one a task may write to. |
| A `runner` service running `sleep infinity` | `EXECUTOR=container` launches its *own* containers from `pithagoras-runner:latest`, so nothing references that image and the host's weekly `docker system prune -af` deletes it — silently breaking every task until the next deploy. The service builds the image and keeps a container referencing it. It never runs a task. |
| `deploy.resources.limits` on both services | The host runs ~68 other containers. Without limits one runaway task competes with all of them. |
| No `PORTAL_CONTAINER_NAME` | See below. |

Everything else follows upstream: the portal image is built from the unchanged
`Dockerfile`, `init: true` is kept so orphaned shells are reaped, and the Docker
socket is mounted because the container executor launches sibling containers
through it.

### `PORTAL_CONTAINER_NAME` must stay unset here

It tells the portal which container is *itself*, so the container executor can
translate its own bind mounts into host paths. Upstream's compose sets it to a
fixed name, which works because upstream keeps that name.

Coolify does not: it renames a stack's containers on every deploy
(`portal-<uuid>-<timestamp>`). A fixed name therefore points at nothing, and every
task fails on `docker inspect` before pi starts. Left unset, the portal falls back
to its own hostname, which Docker sets to the container's ID and which stays
correct across redeploys. `server/src/executors/index.ts` and `db-upgrade.ts` were
brought in line with `voice-service.ts`, which already did this.

## The executor patch

`server/src/executors/index.ts` carries one functional change, marked with a
comment.

Upstream's `ContainerExecutor` passes only `OPENROUTER_API_KEY`,
`ANTHROPIC_API_KEY`, `OPENAI_API_KEY` and `--provider` / `--model` into a task
container. pi's configuration directory is *not* mounted, so a provider set up
under **Settings → Models** — an OpenAI-compatible endpoint, which is the whole
point of that screen — does not exist inside the container, and a task has no
model to run on unless the image happens to bake one in.

The patch mounts the portal's pi configuration directory (`settings.json`,
`models.json`, `auth.json`) at `/pi-agent` and points `PI_CODING_AGENT_DIR` at it.
A task then runs on exactly the provider the portal is configured with, and
adding a provider in the UI reaches new tasks without rebuilding the image.

Two consequences worth knowing:

- A task can **read** every provider key the portal holds and could rewrite the
  file. That is not a new trust boundary — anyone with the portal password
  already gets a terminal in the portal container, which holds the Docker socket
  — but it should be understood rather than assumed.
- The mount is translated through the portal's own mounts (`hostMountPath`), as
  the workspace and session mounts already are, so named volumes and nested bind
  mounts behave the same way.

## The seeded provider

`deploy/coolify/seed/pi-agent/` is the **template** written into
`/data/pithagoras/data/home/.pi/agent/` before the first start. It defines one
provider, `clinepass`, carrying one model: `deepseek-v4.1-flash-wafer`.

The template in git holds a `$CLINEPASS_API_KEY` placeholder and no secret. The
deployed copy holds the key itself, because the executor mounts this directory
into every task and pi reads the key straight from the file:

```sh
# on the host, once, before the first start
D=/data/pithagoras/data/home/.pi/agent
mkdir -p "$D"
cp seed/pi-agent/settings.json "$D/settings.json"
cp seed/pi-agent/models.json "$D/models.json"
# then replace the placeholder with the real key, without printing it
python3 - "$D/models.json" <<'PY'
import json,sys
p=sys.argv[1]; d=json.load(open(p))
d["providers"]["clinepass"]["apiKey"]=open("/path/to/clinepass.key").read().strip()
json.dump(d,open(p,"w"),indent=2)
PY
chmod 600 "$D/models.json" "$D/settings.json"
```

**`OPENAI_API_KEY` is deliberately not set in the Coolify environment.** It is one
of the three names the executor forwards, so it is the obvious place for a key —
but pi reads that name as its built-in OpenAI provider being configured, so the
portal then offers an OpenAI entry whose models call `api.openai.com` with the
wrong key and fail with a 401 that names the wrong thing. With the directory
mounted, no environment variable is needed at all.

### Why the model id is not literally `…-wafer`

In pi 1.1.0 (what pi-web runs) `deepseek-v4.1-flash-wafer` is a local **alias**:
the entry keeps the id users pick, rewrites the request to the real upstream
`cline-pass/deepseek-v4.1-flash` through `samplingParams.model`, and pins the
backend with `compat.vercelGatewayRouting.only = ["wafer"]`.

Pithagoras pins pi **0.82.1**, which implements `vercelGatewayRouting` (verified
in `pi-ai/dist/api/openai-completions.js`: it sends
`providerOptions: { gateway: { only: ["wafer"] } }`) but **not** `samplingParams`.
So the alias id is sent upstream unchanged, that model does not exist there, and
every task dies with `404 "model not found"`.

Using the real upstream id *and* the same routing pin reproduces exactly the
backend the alias selects, and works on the pinned pi — no version bump, which
would put the runner's `pi` out of step with the portal's RPC protocol. The model
still shows in the UI as **DeepSeek V4.1 Flash-Wafer**.

## Deploying a change

```sh
git push origin deploy/coolify
curl -sS -X POST -H "Authorization: Bearer $COOLIFY_API_TOKEN" \
  "$COOLIFY_URL/api/v1/deploy?uuid=usn7v2ezjed2ux5ywbwzhiv1"
```

Coolify builds both images with `docker compose build --pull` and restarts the
stack. Taking upstream's changes:

```sh
git fetch upstream && git merge upstream/main   # on this branch
```

Environment variables live in the Coolify UI (or `PATCH .../envs/bulk`): `PORTAL_PASSWORD`,
`PORTAL_SECRET`, `PITHAGORAS_DATA_DIR`, `PITHAGORAS_WORKSPACES_DIR`, `TZ`,
`TASK_MEMORY_MB` (1536), `TASK_CPUS` (1.5), `TASK_PIDS_LIMIT` (512).

## Isolation between chats

**`EXECUTOR=container` gives one container per session, not one workspace per
session.** Two chats in the same project get two containers that both mount the
same host directory at `/workspace`: they are separate processes with separate
resource caps, but they edit the same files and can read and overwrite each
other's work.

The check that established this, with transcripts, is in
[`ISOLATION.md`](./ISOLATION.md). In short: chat B read — and then overwrote — a
file written by chat A in the same project, while chat C in a different project
could see neither and could not reach out of its own container with `..`.

To run chats genuinely in parallel, give each its own project folder, or run them
one at a time. Nothing in the portal serialises two chats writing to one folder.

## Operating

```sh
# status
ssh-coolify "docker ps --filter name=pithagoras --format '{{.Names}}\t{{.Status}}'"

# a task container, while a chat runs
ssh-coolify "docker ps --filter name=pithagoras- --format '{{.Names}}\t{{.Command}}'"

# the image EXECUTOR=container launches from
ssh-coolify "docker run --rm pithagoras-runner:latest pi --version"

# resource use
ssh-coolify "docker stats --no-stream \$(docker ps --filter name=pithagoras --format '{{.Names}}' | tr '\n' ' ')"
```

Known gap: `/data/pithagoras` is a host bind mount, so the nightly restic backup
(`/usr/local/bin/backup-coolify.sh`, which covers `/data/coolify` and
`/var/lib/docker/volumes`) does **not** include it. `portal.db` and the chats'
session files would not survive losing the host. Add it to that script's
`EXTRA_PATHS` if the portal holds work worth keeping.
