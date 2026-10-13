---
title: "GitOps for a Homelab: Automatic Docker Compose Deploys with doco-cd"
date: 2026-09-12
draft: true
description: "Putting a homelab's Docker Compose stacks under GitOps with doco-cd: pull-based deploys over polling, secrets encrypted in git with sops, gated image updates with Renovate, and the config split that decides whether adding a service needs a restart."
tags: ["gitops", "docker", "docker-compose", "self-hosted", "homelab", "sops", "renovate", "unraid"]
ShowToc: true
---

Getting your compose files into git solves the worst problem: you stop guessing which of `docker-compose.yaml.bak2` and `docker-compose.yaml.old` was the one that worked. But it leaves a smaller problem behind, and that one is sneakier.

Deploying is still manual. You SSH into the server, `git pull`, and run something like `./dc.sh jellyfin up -d`. Which means the repository is the source of truth in theory, while the server is the source of truth in practice. The two agree right up until the evening you fix something directly on the box because it's 11pm and you just want the thing working. That edit never makes it back into git. A month later the repo describes a setup that no longer exists, and you find out when you try to rebuild.

GitOps closes that loop by inverting the direction. Instead of you pushing changes to the server, an agent on the server pulls them from git. The repository stops being a record of what you meant to deploy and becomes the thing that is actually deployed. There's no longer a copy of the compose files on the server to hand-edit in the first place — the only copy is a clone the agent manages and resets on every pull, which sounds like a restriction until you realize it's the entire point.

This post walks through that setup on my homelab: [doco-cd](https://github.com/kimdre/doco-cd) watching a git repository of Docker Compose stacks, with secrets encrypted in the repo via sops and image updates gated through Renovate. It picks up where [organizing Docker Compose files for a homelab]({{< ref "organizing-docker-compose-files.md" >}}) left off, and cashes the check that post wrote when it mentioned sops as "outside the scope."

## How it works

The whole flow, end to end:

```text
git push to main
  → doco-cd polls the repo (every 60s)
  → change detected
  → reads .doco-cd.yml for what to deploy
  → decrypts sops-encrypted .env files
  → docker compose up -d
```

The part worth noticing is what *isn't* there. No CI runner holds credentials to my homelab. Nothing reaches in from the internet. The server clones a repository over an outbound SSH connection and acts on what it finds, the same way it would fetch any other git repo. The trust flows one direction, outward, which is a meaningfully different security posture from a pipeline that needs a way to reach your network.

## Prerequisites

This builds directly on the layout from the previous post, so it assumes:

- Comfort with Docker Compose, git, and the Linux terminal
- The `apps/<name>/docker-compose.yml` plus root-level `shared.env` structure described [there]({{< ref "organizing-docker-compose-files.md" >}})
- A git remote you can reach from the server over SSH

My server runs Unraid, which shows up below in paths like `/mnt/user/appdata/`. Nothing here is Unraid-specific beyond those paths. Swap them for wherever your host keeps persistent container data.

## Why polling instead of webhooks

doco-cd supports both. A webhook gives you near-instant deploys: GitHub sends a request the moment you push, and the stack updates seconds later. Polling means doco-cd checks the repository on an interval and picks up changes on the next pass.

Both work, but I went with polling simply so I don't have to open an addtional port in my router's firewall. If I ever change networks or routers, thats one less port I have to remember to port forward.

## Running doco-cd itself

There's one stack that can't live in the GitOps repository: doco-cd. It's the thing that reads the repo, so it has to exist before the repo means anything. Its compose file sits on the server outside of version control, and it's the one piece of this setup I still manage by hand.

```yaml
services:
  doco-cd:
    image: ghcr.io/kimdre/doco-cd:latest
    container_name: doco-cd
    restart: unless-stopped
    ports:
      - "8088:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ${DOCO_SOPS_AGE_KEY_FILE}:/keys/keys.txt:ro
      - /mnt/user/appdata/doco-cd/data:/data
      - /mnt/user/appdata/doco-cd/ssh/id_ed25519:/etc/doco-cd/id_ed25519:ro
      - /mnt/user/appdata/doco-cd/poll.yaml:/poll.yaml:ro
    env_file:
      - .env
```

Each mount is doing something specific:

**`/var/run/docker.sock`** is the socket doco-cd needs to connect to docker. doco-cd drives the host's Docker daemon directly through its socket. This socket is how doco-cd is able to view, remove and create docker containers.

**`/data`** is where doco-cd clones the repositories it watches. I use a bind mount rather than a named volume so the clone is visible on the host. This is handy for looking at what doco-cd actually checked out when a deploy does something surprising, and it lands in the same backup as the rest of `/mnt/user/appdata`.

**`/etc/doco-cd/id_ed25519`** is the SSH deploy key, mounted read-only. This is why the repository URL in the poll config is the SSH form (`git@github.com:...`) rather than HTTPS.

**`/keys/keys.txt`** is the age private key that decrypts the sops-encrypted `.env` files. More on that below.

The port mapping (`8088:80`) exposes doco-cd's webhook listener. Since I'm polling, nothing sends requests to it, and it isn't forwarded through the router. It's mapped in case I change my mind later.

I start it with `docker compose up -d`, and updating it means pulling a new image and recreating it by hand. Unraid restores container run-state across reboots — anything running at shutdown comes back at boot — so in practice it survives restarts without intervention.

## Two config files, two lifecycles

doco-cd reads two different config files, and the difference between them is the single most important thing to understand about running it.

`poll.yaml` tells doco-cd *what to watch*. This only is read at startup, so changing this requires a restart of doco-cd. In this repo it's three lines:

```yaml
- url: git@github.com:samuel-santana-iii/homelab-gitops.git
  reference: main
  interval: 60s
```

`.doco-cd.yml`, at the repo root, tells doco-cd *what to deploy*. One YAML document per stack, separated by `---`:

```yaml
name: uptime-kuma
working_dir: apps/uptime-kuma
compose_files:
  - docker-compose.yml
env_files:
  - ../../shared.env
  - .env
---
name: jellyfin
working_dir: apps/jellyfin
compose_files:
  - docker-compose.yml
env_files:
  - ../../shared.env
  - .env
```

`working_dir` is relative to the repo root, and the paths in `env_files` are relative to `working_dir`. This is why `shared.env` at the root is reached as `../../shared.env`. Order matters there for the same reason it mattered when running Compose by hand: later files can override pre-existing variables, so `shared.env` goes first before the app's own `.env`.

That split isn't how I set it up originally. I originally had my deployments in my `poll.yaml`, and the reason it changed is worth explaining.

### Why the deployments moved

doco-cd lets you define deployments inline in `poll.yaml`, under a `deployments:` key, and that's where mine started:

```yaml
- url: git@github.com:samuel-santana-iii/homelab-gitops.git
  reference: main
  interval: 60s
  deployments:
    - name: uptime-kuma
      working_dir: apps/uptime-kuma
      compose_files:
        - docker-compose.yml
      env_files:
        - ../../shared.env
        - .env
    - name: jellyfin
      working_dir: apps/jellyfin
      # ...and so on for every stack
```

It worked, but every time I added a stack I had to restart the doco-cd container before it would deploy. The deployment was automated. *Registering* a deployment wasn't. For a setup whose entire premise is "push to git and watch the changes apply," needing to SSH in and restart a container to add a service defeats the point.

The cause is that `poll.yaml` is startup configuration. doco-cd reads it once when the process boots and never looks at it again. There's no file watcher on it and no reload signal. Anything defined in `poll.yaml` is frozen until the container restarts.

`.doco-cd.yml` is different. It lives in the repository, so doco-cd re-reads it out of a fresh clone on every poll. A new modification in that file is picked up on the next poll cycle like any other change.

So the fix was to move every deployment out of `poll.yaml` and into `.doco-cd.yml`, leaving `poll.yaml` with only the three lines above. Adding a stack is now a commit and a push, live within 60 seconds, with nothing to restart.

> **A warning if you go looking for more.** Ensure you point `POLL_CONFIG_FILE` to a file on on host machine instead of the git repo. That way you can easily change the github url if you ever change the repository url. I keep a copy in my repository, but that for source control so I know the file format.

## Secrets: encrypted in git with sops

The previous post gitignored every `.env` file. This is done for security purposes, but if we ever need to restore from source control, all the environment variables will need to be manually set. 

[sops](https://github.com/getsops/sops) allows us to check in our secrets by encrypting the real values and committing those. The file in git holds actual configuration, just unreadable without the key:

```env
TIMEZONE=ENC[AES256_GCM,data:n8Kq2Rm7Xp4vLzT0aWcBdQfEgg==,iv:h3Nb9PwXqVzR...]
APPDATA=ENC[AES256_GCM,data:R7mX2pQ9vKzT4aWcBdEyNg==,iv:p4Lw8RtYmVzQ...]
```

Encryption rules live in `.sops.yaml` at the repo root:

```yaml
creation_rules:
  - path_regex: \.env$
    age: "age1exampleexamplexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  - path_regex: \.enc\.yaml$
    age: "age1exampleexamplexxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

Any file matching those patterns gets encrypted to that age recipient. Editing is a single command that decrypts to a temp file, opens your editor, and re-encrypts on save:

```bash
sops apps/jellyfin/.env      # edit in place
sops -d apps/jellyfin/.env   # decrypt to stdout
```

Note that sops encrypts *values*, not keys. Variable names stay readable in the committed file, so `git diff` still shows you which variable changed even when it can't show you the new value. That's a useful feature. You get a meaningful history without leaking anything.

### Getting the key onto the server

I didn't cover creating a age key for sops, but it will need to be copied to your server. I copied they key with `scp`.

The key is what decrypts the repository, so it can't live *in* the repository. Once copied doco-cd needs to know where to find it. One environment variable point at that key:

```env
DOCO_SOPS_AGE_KEY_FILE=/mnt/user/appdata/doco-cd/keys/keys.txt
```

## Removing a stack

Adding a stack is a commit. Removing one is not as straightforward.

Deleting a stack's entry from `.doco-cd.yml` doesn't tear anything down. doco-cd stops managing the stack, but the containers keep running. That's intentional. This ensures a accidental remove does not delete data or stop your container.

Teardown has to be requested explicitly, then cleaned up, in two commits. First, mark the stack for destruction in `.doco-cd.yml`:

```yaml
name: tailscale
working_dir: apps/tailscale
destroy:
  enabled: true
  remove_dir: false
compose_files:
  - docker-compose.yml
```

On the next poll, doco-cd stops and removes that stack's containers, volumes, and images. Once that's confirmed, a second commit deletes the document and the `apps/tailscale/` directory.

## Keeping images current with Renovate

Automatic updates for a container can eventually take down your service. An unpinned `:latest` tag means a new image release can deploy on your server at any hour. Its a convenient feautre until a version with breaking changes deploys. 

Every image version here is pinned, and Renovate applies updates as pull requests. Version updates become something you manually approve. You can see if a major version is pending.

`renovate.json` is scoped to Docker Compose files and runs weekly:

```json
{
  "enabledManagers": ["docker-compose"],
  "timezone": "America/Los_Angeles",
  "schedule": ["before 4am on Monday"]
}
```

Related images are grouped so one PR covers a whole stack rather than five arriving separately:

```json
{
  "matchPackageNames": [
    "ghcr.io/linuxserver/prowlarr",
    "ghcr.io/linuxserver/sonarr",
    "ghcr.io/linuxserver/radarr",
    "ghcr.io/linuxserver/sabnzbd",
    "fallenbagel/jellyseerr"
  ],
  "groupName": "arr stack",
  "automerge": false
}
```

### Gating updates that aren't just image swaps

Most updates are a tag change and a recreate. Postgres major versions aren't — the data directory format changes between majors, so the container will refuse to start against a data directory written by an older version. It needs a dump and restore, which is a maintenance window, not a merge.

Renovate can express that distinction:

```json
{
  "matchDatasources": ["docker"],
  "matchPackageNames": ["postgres"],
  "matchUpdateTypes": ["major"],
  "dependencyDashboardApproval": true,
  "addLabels": ["needs-manual-db-migration"]
}
```

`dependencyDashboardApproval` means Renovate won't even open the PR until I tick a box on the dependency dashboard. Postgres majors stop showing up as something I might merge half-awake on a Monday morning, and become something I go and ask for when I have time to do the migration properly.

Immich needed extra work here. Its Postgres image uses a compound tag that isn't semver:

```text
ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
```

Renovate can't tell which part of that is the major version, so it needs a regex telling it how to read the tag:

```json
{
  "matchDatasources": ["docker"],
  "matchPackageNames": ["ghcr.io/immich-app/postgres"],
  "versioning": "regex:^(?<major>\\d+)-vectorchord0\\.(?<minor>\\d+)\\.(?<patch>\\d+)-pgvectors0\\.(?<build>\\d+)\\.(?<revision>\\d+)$"
}
```

The part that cost me time: this has to be its own rule. My first attempt put the `versioning` regex and the `matchUpdateTypes` gate in a single `packageRule`, which silently didn't work — Renovate needs the custom versioning applied before it can classify an update as major or minor, and asking it to do both in one rule is circular. Splitting them into two rules, one defining how to read the version and one gating on the result, fixed it.

## Takeaway

Adding a service to my homelab is now:

1. Write `apps/<name>/docker-compose.yml`
2. Encrypt its `.env` with sops
3. Add a document to `.doco-cd.yml`
4. Commit and push

It's running within 60 seconds. Nothing to SSH into, nothing to restart, no wrapper script to remember the flags for.

The bigger change is that most of the drift is gone. The repository is what put the containers there, `git log` is the deployment history, and `git revert` is the rollback.

It's worth being precise about how much drift that actually covers, because "GitOps means the server can't disagree with git" is an overstatement. On each poll doco-cd compares what's running against what the compose files say should be running, and redeploys when something is missing. Stop or delete a container by hand and it comes back within the minute. But that check is about whether services are *running*, not whether a running container's configuration still matches the file. Nothing continuously diffs a live container against the compose file it came from, so a hand-applied change to an already-running stack can sit there until the next commit touches it. Configuration drift is discouraged rather than impossible, and the honest version of the benefit is that the path of least resistance now runs through git.

It isn't free, and the tradeoffs are worth naming. doco-cd manages every stack except itself, so there's one container I still update by hand. Polling means deploys take up to a minute. Mounting the Docker socket gives doco-cd root-equivalent access to the host, which is inherent to what it does but shouldn't be waved away. And the age key is a single secret whose loss means re-encrypting every `.env` in the repo, so it belongs in your backups before you need it.

For a homelab, that's a good trade. The failure mode of the old setup was silent drift I'd discover during a rebuild, months later, with no idea what changed. The failure mode of this one is a container that won't start and a `git log` explaining exactly why.
