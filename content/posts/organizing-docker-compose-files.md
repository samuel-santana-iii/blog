---
title: "Organizing Docker Compose Files for a Homelab"
date: 2026-09-01
draft: false
description: "A directory structure for Docker Compose that separates app data from configuration, shares environment variables across services, tracks the whole setup in git, and handles version updates without breaking things."
tags: ["docker", "docker-compose", "self-hosted", "homelab", "environment-variables", "git"]
ShowToc: true
---

If you're running more than a couple of containers without version control, you've probably lived some version of this: you tweak a `docker-compose.yaml`, it breaks something, so you copy the old one to `docker-compose.yaml.bak` before trying again. A few edits later you've got `docker-compose.yaml.bak2` and `docker-compose.yaml.old`, no idea which one was actually working, and no record of what changed between them.

Putting your compose files in git fixes that. Every edit becomes a commit with a message, `git diff` shows you exactly what changed, and rolling back a bad config is a simple `git checkout` command. Pair that with a repository that's kept separate from your container data, shared environment variables to cut down on duplication, and a small script to run everything, and your containers become a lot easier to manage.

Here's the setup: a directory layout that keeps app data out of git, a shared env file that removes duplicated settings, a small wrapper for running Compose commands, and a version-pinning approach that makes updates as reviewable as everything else.

The examples below use two apps: Jellyfin, a self-hosted media server, and Uptime Kuma, a self-hosted uptime monitor. Here's the layout I use:

```text
appdata/              # Configs, databases, and logs; sibling to homelab/, not tracked in git
homelab/
├── apps/             # Compose and .env files
│   ├── uptime-kuma/
│   │   ├── .env
│   │   ├── .env.sample
│   │   └── docker-compose.yaml
│   └── jellyfin/
│       ├── .env
│       ├── .env.sample
│       └── docker-compose.yaml
├── .gitignore
├── shared.env        # Variables common to every app
└── shared.env.sample
```

## Prerequisites

This structure assumes some background. To follow along, you should be comfortable with:

- git (version control)
  - I will not go over any git commands or initializing a git repository
- basic bash scripting (just enough to write and execute a simple script)
- the Linux terminal (this setup is optimized for the terminal)

## Separating appdata from apps

Splitting `appdata/` and `homelab/` allows you to separate your container configuration from its actual state.

`homelab/` is the defined configuration: compose files, `.env` files, anything that describes how a container should run. It's small, it's text, and it belongs in git.

`appdata/` is the state: SQLite databases, media library metadata, uploaded files, logs. It's often large, it changes constantly, and none of it belongs in a git history.

Mixing the two means your repo either balloons with binary data you don't want tracked, or you end up manually picking through a folder to figure out what's safe to `git add`. Keeping them separate means `homelab/` stays a clean, versioned record of your setup, while `appdata/` is just a directory you back up, snapshot, or `rsync` to a new host independently of the code that defines your containers.

## Sharing variables with shared.env

`shared.env` holds the variables that every app needs but that have nothing to do with any single app's logic:

```env
TIMEZONE=America/New_York
APPDATA=/opt/appdata
MEDIA_FOLDER=/mnt/media
HOST_IP=192.168.1.2
PUID=1000
PGID=1000
```

- **APPDATA** points at the `appdata/` folder from above. Every container's config path is built from this one variable, so moving the whole setup to a new disk, or a new OS entirely, is a one-line change instead of an edit-every-compose-file exercise.
- **MEDIA_FOLDER** does the same job for anything that touches your media library, keeping the path in one place instead of copy-pasted into every media app's compose file.
- **PUID/PGID** set the user and group IDs a container runs as. Keeping these consistent across containers means every app writes files with the same ownership. This prevents permission mismatches between, say, Jellyfin and your download client. This matters a lot on Unraid, where containers default to a generic `nobody` account. That account often doesn't match the UID of the user you actually use to browse or edit those same files over a network share, which leads to permission-denied errors the moment something else touches a file the container created.
- **TIMEZONE** holds the time zone value most containers ask for.
- **HOST_IP** covers containers that need to know the host's LAN address such as reverse proxies.

Each compose file decides what to call the variable once it hands it to the container, which is why the Jellyfin example below maps it with `TZ=${TIMEZONE}`: that's the name the Jellyfin image expects.

The idea is to change it once in `shared.env`, and every app that depends on it picks up the new value.

## Using environment variables in Compose

Docker Compose deals with environment variables in two different places, and it's worth keeping them straight:

1. **Variable substitution in the compose file itself.** Anywhere you write `${PUID}` or `${APPDATA}` in `docker-compose.yaml`, Compose replaces it before the file is parsed. This is how `${APPDATA}/jellyfin` becomes an actual path.
2. **Variables passed into the container.** The `environment:` and `env_file:` keys control what a process running *inside* the container sees. These are separate from substitution, though they're often fed by the same values.

Here's `apps/jellyfin/docker-compose.yaml` using both:

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TIMEZONE}
    volumes:
      - ${APPDATA}/jellyfin:/config
      - ${MEDIA_FOLDER}:/media
    ports:
      - "8096:8096"
    restart: unless-stopped
```

The `${...}` references here need to resolve to something at the time you run `docker compose`, which means Compose needs to see both `shared.env` and the app's own `.env`. By default, Compose only auto-loads a file named `.env` sitting next to the compose file, so `shared.env`, a couple of directories up, won't be picked up on its own. Point Compose at both files explicitly with `--env-file`, which recent versions of Docker Compose (v2.24+) let you pass more than once:

```bash
cd apps/jellyfin
docker compose --env-file ../../shared.env --env-file .env up -d
```

Order matters in that command. Compose applies `--env-file` values in the order they're given and lets later files win, so `shared.env` has to come first: that way the app's own `.env` can override a shared value when it needs to (a different `PUID`, a media path override for just this app) without touching the shared file. List them the other way around and `shared.env` would clobber anything app-specific, which defeats the point of having a per-app `.env` at all.

That's also what `apps/uptime-kuma/.env` and `apps/jellyfin/.env` are for: app-specific settings such as an API key or a password. Nothing that's already in `shared.env` needs to be repeated there.

### Failing fast on missing variables

If a variable in `shared.env` gets renamed, mistyped, deleted, or just never set on a new host, plain `${VAR}` substitution doesn't complain. It silently resolves to an empty string, and you end up with something like `/media` mounted as a bind mount for `:/media`, which fails in a much more confusing way than a missing variable would.

`${VAR:?error_msg}` fixes that. It tells Compose to stop and print an error if the variable is unset or empty, instead of substituting nothing. Applied to the Jellyfin service above, only the `environment:` and `volumes:` lines change:

```diff
     environment:
-      - PUID=${PUID:?err}
-      - PGID=${PGID:?err}
-      - TZ=${TIMEZONE:?err}
     volumes:
-      - ${APPDATA:?err}/jellyfin:/config
-      - ${MEDIA_FOLDER:?MEDIA_FOLDER is not set}:/media
```

Forget to pass `--env-file ../../shared.env`, or mistype a variable name in `shared.env` itself, and `docker compose up` refuses to start:

```text
error while interpolating services.jellyfin.environment.[1]: required variable TIMEZONE is missing a value: err
error while interpolating services.jellyfin.volumes.[1]: required variable MEDIA_FOLDER is missing a value: MEDIA_FOLDER is not set
```

That's a lot easier to debug than a container that starts fine but silently mounts the wrong path. I use `:?err` on anything from `shared.env` where a missing value would cause a container to run against the wrong directory rather than just fail outright, especially `APPDATA` and `MEDIA_FOLDER`. It's less necessary for something like `TZ`, where an empty value just means the container falls back to UTC instead of doing something actively wrong.

To check what a compose file actually resolves to without starting anything, run the same command with `config` instead of `up`:

```bash
docker compose --env-file ../../shared.env --env-file .env config
```

It prints the fully substituted YAML, which is the fastest way to confirm a variable is landing where you expect before something fails at container-start time.

## A wrapper for common commands

Typing both `--env-file` flags every time gets old fast, so I keep a small wrapper instead of repeating it:

```bash
# apps/dc.sh
#!/usr/bin/env bash
cd "$(dirname "$0")/$1" || exit 1
shift
docker compose --env-file ../../shared.env --env-file .env "$@"
```

```bash
./dc.sh jellyfin up -d
./dc.sh jellyfin down
```

`down` still has to parse `docker-compose.yaml` to know what to tear down, so it needs the same `--env-file` values as `up` does. If `shared.env` is missing or a `${VAR:?err}` variable isn't set, `down` fails with the same interpolation error `up` would; it doesn't get a pass just because it's removing things rather than creating them.

## Tracking changes with git

`homelab/` is exactly what you want under version control: small Docker Compose and `.env.sample` files that describe your entire setup, with a full history of every change. Before the first commit, add a `.gitignore` at the root of `homelab/`:

```text
# homelab/.gitignore
.env
shared.env
```

A pattern with no leading slash matches at any depth, so `.env` here covers every app's `apps/*/.env`, and `shared.env` covers the one at the root. `.env.sample` and `shared.env.sample` don't match either pattern, so both stay tracked.

Run `git init` inside `homelab/`, then commit `.gitignore`, `apps/`, and `shared.env.sample`.

`appdata/` sits outside `homelab/` entirely, which is what [keeps it out of git in the first place](#separating-appdata-from-apps): no ignore rule to write, nothing for a stray `git add` to sweep up.

The reason to gitignore `shared.env` too, not just the per-app files, is that what counts as sensitive there isn't fixed. It might only hold paths and IDs today, but it's just as capable of picking up a domain name or an API token for a reverse proxy or DNS provider as any per-app `.env` is. Rather than deciding case by case, everything gets the same treatment: `.env` and `shared.env` stay out of git, and `.env.sample` and `shared.env.sample` document their variable names instead, with placeholder or blank values, so the shape of what each file needs is still visible in the repo, just not the actual value.

There's one exception to not checking in `.env` files. A file can be checked into git with real values in it if it's encrypted first, with a tool like [sops](https://github.com/getsops/sops). That's outside the scope of this post, but worth knowing about if you'd rather keep secrets versioned alongside everything else than manage them out of band.

`dc.sh` is just as much a definition of how the stack runs as the compose files are, so it's committed alongside them in `apps/`. Make sure it's executable before the first commit (`chmod +x apps/dc.sh`), since git tracks that bit and a clone without it will fail with `Permission denied` on `./dc.sh`.

## Updating an app

Pin a major version instead of `latest`:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
```

A floating major tag like `2` protects against an unannounced jump to a breaking `3.x` release (I'm looking at you, Immich.) Most software maintainers only update what `2` points to within that major version's minor and patch releases, never across a major version boundary.

Because the tag itself doesn't change within a major version, `up -d` alone won't pick up anything new behind it. Compose only pulls automatically when there's no local image for that tag yet. Getting the latest build within `2` means pulling explicitly, then recreating:

```bash
./dc.sh uptime-kuma pull
./dc.sh uptime-kuma up -d
```

Moving to a new major version is the one update that should still go through git, since that's the point where behavior is actually likely to change:

```bash
git diff apps/uptime-kuma/docker-compose.yaml
```

```diff
-    image: louislam/uptime-kuma:2
+    image: louislam/uptime-kuma:3
```

```bash
git commit -am "Move uptime-kuma to major version 3"
./dc.sh uptime-kuma up -d
```

That commit is a deliberate, reviewable decision to accept a possible breaking change, while day-to-day updates inside a major version stay a `pull` and `up -d` away without touching the compose file at all. If a major-version jump goes badly, `git revert` puts the tag back and `up -d` recreates the container on the old major version again.

## Takeaway

Getting the whole stack running on a new machine comes down to a few steps:

1. Clone `homelab/`.
2. Restore `appdata/` from backup into its sibling directory.
3. Copy `shared.env.sample` to `shared.env` and each app's `.env.sample` to `.env`, and fill in the new host's values.
4. Bring the containers up.

No hunting through folders, no guessing what a compose file expects to find where, and no wondering which version of an app is actually running, since that's sitting right there in `git log`.
