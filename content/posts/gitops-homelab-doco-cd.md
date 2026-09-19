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

GitOps closes that loop by inverting the direction. Instead of you pushing changes to the server, an agent on the server pulls them from git. The repository stops being a record of what you meant to deploy and becomes the thing that is actually deployed. Editing a compose file on the server no longer works, because the next reconcile overwrites it — which sounds like a restriction until you realize it's the entire point.

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

My server runs Unraid, which shows up below in paths like `/mnt/user/appdata/`. Nothing here is Unraid-specific beyond those paths — swap them for wherever your host keeps persistent container data.

## Why polling instead of webhooks

doco-cd supports both. A webhook gives you near-instant deploys: GitHub sends a request the moment you push, and the stack updates seconds later. Polling means doco-cd checks the repository on an interval and picks up changes on the next pass.

Webhooks need doco-cd to be reachable from the internet. That means an inbound port forwarded through the router to a service that accepts requests from outside my network. For a homelab, that's a new piece of exposed attack surface to keep patched and think about, in exchange for saving under a minute.

Polling only makes outbound connections. Nothing needs to be reachable, no port gets forwarded, and the router configuration doesn't change at all. The cost is latency: with a 60-second interval, a push takes up to a minute to land.

For a homelab that's an easy trade. I'm not shipping to production on a deadline; I'm adding a media server on a Saturday. Sixty seconds of waiting costs nothing, and not exposing an endpoint to the internet is worth real money in attention I don't have to spend. If you're running this somewhere the latency matters, the calculus changes — but be honest about whether it actually matters before you open the port.

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

**`/var/run/docker.sock`** is the mechanism. doco-cd doesn't shell out to a `docker` binary or talk to a remote API — it drives the host's Docker daemon directly through its socket. That's also the security caveat, and it's worth stating plainly: a container with the Docker socket mounted can start any container it likes with any privileges it likes, which is effectively root on the host. I accept that here because doco-cd's whole job is managing containers on this machine and there's no way for it to do that job with less access. It's a deliberate trade, not an oversight, and it's the reason this container should be something you actually trust.

**`/data`** is where doco-cd clones the repositories it watches. It's a bind mount rather than a named volume so the clone survives container recreation, which matters more than it looks like it does.

**`/etc/doco-cd/id_ed25519`** is the SSH deploy key, mounted read-only. This is why the repository URL in the poll config is the SSH form (`git@github.com:...`) rather than HTTPS — there's no token involved, just a key the server holds.

**`/keys/keys.txt`** is the age private key that decrypts the sops-encrypted `.env` files. More on that below.

The port mapping (`8088:80`) exposes doco-cd's webhook listener. Since I'm polling, nothing sends requests to it, and it isn't forwarded through the router. It's mapped in case I change my mind later.

I start it with `docker compose up -d`, and updating it means pulling a new image and recreating it by hand. Unraid restores container run-state across reboots — anything running at shutdown comes back at boot — so in practice it survives restarts without intervention.

One honest inconsistency: doco-cd is pinned to `:latest` while every stack it manages is pinned to a specific version and gated behind Renovate. That's backwards, given it's the most privileged container on the box. It's on my list.

## Two config files, two lifecycles

doco-cd reads two different config files, and the difference between them is the single most important thing to understand about running it.

`poll.yaml` tells doco-cd *what to watch*. In this repo it's three lines:

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
  - .env
  - ../../shared.env
---
name: jellyfin
working_dir: apps/jellyfin
compose_files:
  - docker-compose.yml
env_files:
  - .env
  - ../../shared.env
```

`working_dir` is relative to the repo root, and the paths in `env_files` are relative to `working_dir` — which is why `shared.env` at the root is reached as `../../shared.env`. Order matters there for the same reason it mattered when running Compose by hand: later files win, so `shared.env` goes first and the app's own `.env` can override a shared value when it needs to.

That split isn't how I set it up originally, and the reason it changed is worth explaining.

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
        - .env
        - ../../shared.env
    - name: jellyfin
      working_dir: apps/jellyfin
      # ...and so on for every stack
```

It worked, but every time I added a stack I had to restart the doco-cd container before it would deploy. The deployment was automated; *registering* a deployment wasn't. For a setup whose entire premise is "push to git and walk away," needing to SSH in and restart a container to add a service defeats the point.

The cause is that `poll.yaml` is startup configuration. doco-cd reads it once when the process boots and never looks at it again. There's no file watcher on it and no reload signal. Anything defined in `poll.yaml` is frozen until the container restarts.

`.doco-cd.yml` is different. It lives in the repository, so doco-cd re-reads it out of a fresh clone on every poll. A new document in that file is picked up on the next cycle like any other change.

So the fix was to move every deployment out of `poll.yaml` and into `.doco-cd.yml`, leaving `poll.yaml` with only the three lines above. Adding a stack is now a commit and a push, live within 60 seconds, with nothing to restart.

> **A warning if you go looking for more.** It's tempting to point `POLL_CONFIG_FILE` at a copy of `poll.yaml` inside the repository, on the theory that doco-cd will then pick up poll config changes from git too. It won't. The read-once behavior is a property of when the file is loaded at startup, not of where the file lives, so relocating it changes nothing — and pointing it inside doco-cd's own clone means a wiped data directory leaves it unable to start at all. Keep `poll.yaml` mounted in from outside the repo, and keep everything that should change without a restart in `.doco-cd.yml`.

## Secrets: encrypted in git with sops

The previous post gitignored every `.env` file and committed `.env.sample` files in their place. That works, but it has a failure mode: the samples drift. Someone adds a variable to the real `.env`, forgets the sample, and six months later a rebuild fails on a variable nobody remembers.

[sops](https://github.com/getsops/sops) removes the need for samples entirely by encrypting the real values and committing those. The file in git holds actual configuration, just unreadable without the key:

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

Note that sops encrypts *values*, not keys. Variable names stay readable in the committed file, so `git diff` still shows you which variable changed even when it can't show you the new value. That's a useful property — you get a meaningful history without leaking anything.

### Getting the key onto the server

There's exactly one step in this entire setup that can't be automated, and it's this one: the age private key has to reach the server out of band. I copied it with `scp`.

That's not a gap in the tooling, it's unavoidable. The key is what decrypts the repository, so it obviously can't live *in* the repository. Every secrets-management scheme bottoms out in one credential that has to be delivered by some other means — you can push that credential into a cloud KMS or a vault, but then something has to authenticate to *that*, and you're holding a different key instead. At some point a human copies a secret to a machine. This setup has exactly one of those, which is about as good as it gets.

Two environment variables point at that key, and the similar names make them easy to confuse:

```env
DOCO_SOPS_AGE_KEY_FILE=/mnt/user/appdata/doco-cd/keys/keys.txt
SOPS_AGE_KEY_FILE=/keys/keys.txt
```

`DOCO_SOPS_AGE_KEY_FILE` is a **host** path. It's consumed by the compose file as the source of the bind mount (`${DOCO_SOPS_AGE_KEY_FILE}:/keys/keys.txt:ro`). `SOPS_AGE_KEY_FILE` is the **container** path — it's what sops itself reads once it's running inside doco-cd. They describe the two ends of the same mount, and getting them backwards produces a decryption failure that doesn't obviously point at either one.

### The file that stays in plaintext

One `.env` in the repo is deliberately unencrypted, with a comment saying so:

```env
# Plaintext on purpose: Renovate needs to read this value directly to
# open/gate version-bump PRs. Do not run `sops -e` on this file.
IMMICH_VERSION=v3
```

Renovate parses files in the repository to find version strings it can update. It has no access to the age key and no way to decrypt anything, so a version pinned inside an encrypted file is invisible to it. Splitting the version out into its own plaintext `version.env` keeps it visible to Renovate while the actual secrets for that stack stay encrypted next to it. The comment exists because this looks exactly like a mistake, and future me needed to know it wasn't.

## Removing a stack

Adding a stack is a commit. Removing one is not symmetrical, and this surprised me enough to be worth flagging.

Deleting a stack's document from `.doco-cd.yml` doesn't tear anything down. doco-cd stops managing the stack, but the containers keep running. That's intentional: a reconciler that destroys containers and volumes whenever an entry disappears from a config file is one bad merge away from deleting data you wanted.

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

`remove_dir: false` is the part to get right. The default is `true`, which deletes the deployment's directory after teardown — but every deployment in this repo shares one checkout under `/data`. "The deployment's directory" is the whole clone, not just that app's folder, so the default would take every other stack's configuration with it. If you're running one deployment per repository, the default is fine. If you're running a monorepo of stacks like this one, it isn't.

I haven't had to run this yet — the one stack I've removed predates the current layout, and I deleted it the blunt way. But it's the kind of thing worth understanding before you need it rather than during.

## Keeping images current with Renovate

Automatic deploys make stale images worse, not better. If pushing to git deploys instantly, an unpinned `:latest` tag means a maintainer's release can land on your server at any hour with no involvement from you. Everything here is pinned, and updates arrive as pull requests.

`renovate.json` is scoped to Docker Compose files and runs weekly:

```json
{
  "enabledManagers": ["docker-compose"],
  "timezone": "America/Los_Angeles",
  "schedule": ["before 4am on Monday"]
}
```

Nothing automerges. That's the same reasoning as the major-version bump in the previous post: a version change is a decision to accept a possible breaking change, and a pull request is where that decision gets made and recorded. The difference is that Renovate now finds the updates instead of me noticing them eventually.

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

The bigger change is that the drift is gone. The server can't disagree with the repository anymore, because the repository is what put it there. `git log` is the deployment history. `git revert` is the rollback. A "quick fix" applied directly on the box gets reverted within a minute, which is annoying exactly once and then teaches you to make the change in git instead.

It isn't free, and the tradeoffs are worth naming. doco-cd manages every stack except itself, so there's one container I still update by hand. Polling means deploys take up to a minute. Mounting the Docker socket gives doco-cd root-equivalent access to the host, which is inherent to what it does but shouldn't be waved away. And the age key is a single secret whose loss means re-encrypting every `.env` in the repo, so it belongs in your backups before you need it.

For a homelab, that's a good trade. The failure mode of the old setup was silent drift I'd discover during a rebuild, months later, with no idea what changed. The failure mode of this one is a container that won't start and a `git log` explaining exactly why.
