# Chat isolation: what `EXECUTOR=container` does and does not guarantee

Tested on the live deployment (`https://pithagoras.382972.xyz`, Coolify app
`usn7v2ezjed2ux5ywbwzhiv1`) on 2026-10-11, pi 0.82.1, Docker 29.1.3.

The question: does one container per task mean one *workspace* per chat? It does
not. A task container is bounded by the **project** it was started in, not by the
chat. Chat-to-chat separation inside a project does not exist.

## What a task container is given

`server/src/executors/index.ts` launches, per session:

```text
docker run --rm -i --init \
  --user 0:0 --cap-drop ALL --security-opt no-new-privileges \
  --memory 1536m --memory-swap 1536m --cpus 1.5 --pids-limit 512 \
  -v <project dir>:/workspace \
  -v <session dir>:/sessions \
  -v <pi agent dir>:/pi-agent \
  pithagoras-runner:latest pi --mode rpc --session-dir /sessions
```

Two chats in one project, observed together while both ran a task:

```text
$ docker ps --filter name=pithagoras-
pithagoras-sAcXFnWckbwl | Up 33 seconds
pithagoras-o77KIN6xL-48 | Up 35 seconds

  == pithagoras-sAcXFnWckbwl
     /data/pithagoras/workspaces/demo-project -> /workspace      <-- same host dir
     /data/pithagoras/data/sessions/sAcXFnWckbwl -> /sessions    <-- own
     /data/pithagoras/data/home/.pi/agent -> /pi-agent
  == pithagoras-o77KIN6xL-48
     /data/pithagoras/workspaces/demo-project -> /workspace      <-- same host dir
     /data/pithagoras/data/sessions/o77KIN6xL-48 -> /sessions    <-- own
     /data/pithagoras/data/home/.pi/agent -> /pi-agent
```

Separate containers, separate session directories, **the same project bind
mount**. Both containers write to the same host directory, so they race exactly as
one working tree would.

## The test

Three chats: **A** and **B** in `demo-project`, **C** in `other-project`. A writes
a file, B reads it, C tries to.

**A** — `printf 'MARKER-FROM-CHAT-A-7f3a\n' > marker-from-A.txt`

**B** — `cat marker-from-A.txt`, in the same project:

```text
MARKER-FROM-CHAT-A-7f3a
```

Read across chats: **yes**. B also overwrote it —
`printf 'MODIFIED-BY-CHAT-B\n' > marker-from-A.txt` — and the host file changed:

```text
before: MARKER-FROM-CHAT-A-7f3a
after:  MODIFIED-BY-CHAT-B
```

**C** — in a different project, probing for an escape:

```text
1. ls -a .
   .  ..                       <- its own project only, empty
2. cat marker-from-A.txt
   cat: marker-from-A.txt: No such file or directory
3. cat /workspace/../demo-project/marker-from-A.txt
   cat: /workspace/../demo-project/...: No such file or directory
```

So the boundary the container does draw holds: a task cannot see another
project, and `..` out of `/workspace` lands on the task container's own
filesystem, not the host's workspace root. It also cannot reach the Docker
socket, which is mounted into the portal only — verified on a live task
container (0 occurrences of `docker.sock` in its mounts).

## Summary

| Question | Answer |
| --- | --- |
| Do two chats in one project get separate containers? | Yes — one per session, named `pithagoras-<sessionId>`. |
| Do they get separate workspaces? | **No.** Same project directory, bound read-write into both. |
| Can they read each other's files? | **Yes** — demonstrated. |
| Can they modify or delete each other's files? | **Yes** — demonstrated. |
| Are pi's own conversation/session files shared? | No. `/sessions` is per session. |
| Can a task in one project read another project? | No. Only its own project is mounted. |
| Can a task escape to the host or the Docker socket? | No. `..` stays inside the container; the socket is not mounted into runners. |

`--cap-drop ALL`, `no-new-privileges`, 1536 MB (swap capped to the same, so it
cannot swap instead), 1.5 CPUs and 512 PIDs are all in force on a live task
container, and it is **not** privileged. Note it runs as `0:0`: the runner takes
the portal's uid, and the portal runs as root, so a task is root *inside* a
container that has no capabilities and nothing mounted but its project.

## Practical consequence

Running two chats on one project in parallel means two processes editing one
working tree — the same conflict as two people in one checkout, with no merge
step and no warning. To run chats genuinely in parallel:

- give each chat its **own project folder** (the isolation above is real and
  sufficient), or
- let the tasks target different subdirectories of one project and accept that
  nothing enforces it, or
- run them one at a time.

Container mode is still worth having over `host` mode: it keeps a task out of the
portal process, out of the Docker socket, out of every other project, and inside
a memory and CPU ceiling. It is not, on its own, a workspace-per-chat guarantee,
and nothing in the portal serialises two chats writing to one folder.
