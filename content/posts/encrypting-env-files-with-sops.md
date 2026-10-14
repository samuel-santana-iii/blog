---
title: "Encrypting .env Files for Version Control with sops and age"
date: 2026-10-13
draft: true
description: "Committing real secret values to git with sops and age: what each tool does, generating and backing up a key, encryption rules, a file naming scheme that fails safe, and using encrypted .env files with Docker Compose."
tags: ["sops", "age", "secrets", "docker-compose", "homelab", "git", "environment-variables"]
ShowToc: true
---

Your homelab server's disk dies. You clone your repo onto the replacement, and everything that took weeks to get right comes back in seconds: every compose file, every bind mount, every pinned image tag. Then you bring up Immich:

```text
error while interpolating services.database.environment.POSTGRES_PASSWORD: required variable DB_PASSWORD is missing a value: err
```

The repo has everything except the one file you deliberately kept out of it.

If you keep Docker Compose stacks in git, you've probably gitignored your `.env` files, because they hold passwords and API tokens. You might have committed a `.env.sample` with blank values so you at least know which variables exist. That keeps secrets off GitHub, but it also leaves them out of the one place everything else is backed up. The sample can tell you that `DB_PASSWORD` exists. It can't tell you what it was. So a rebuild turns into resetting database passwords and regenerating API keys, one service at a time.

The values were only excluded because they were plaintext. Encrypt them, and they can go in git like everything else. This post sets that up with sops and age. By the end, your real secrets are committed right next to the compose files that use them, unreadable to anyone without a single key that you keep in a password manager.

## The problem with `.env.sample`

The `.env.sample` approach works by splitting each secrets file in two. `.env` holds the real values and never touches git. `.env.sample` holds the same variable names with blank values and is committed, so the *shape* of the config is versioned:

```env
# apps/immich/.env.sample
DB_USERNAME=
DB_PASSWORD=
DB_DATABASE_NAME=
```

That protects against the obvious failure of pushing a password to GitHub. But it leaves the values with no backup, no history, and no record of when they changed. A config change that touches both `docker-compose.yml` and `.env` only half-lands in git. And every new variable has to be added in two places, by hand, and kept in sync.

You know that secrets didn't belong in git, but everything else about the config did. Encryption keeps them as secrets in git. The only reason the values were excluded before was that they were in plaintext.

## What sops is, and what age is

[sops](https://github.com/getsops/sops) ("Secrets OPerationS") encrypts the *values* in structured files (dotenv, YAML, JSON, INI) and leaves the structure readable. A `.env` file goes in like this:

```env
DB_PASSWORD=hunter2
TZ=America/Los_Angeles
```

and comes out like this:

```env
DB_PASSWORD=ENC[AES256_GCM,data:x7Rk2mQp9w==,iv:Lq3v...,tag:9fKd...,type:str]
TZ=ENC[AES256_GCM,data:pV8nW2kRtLm4QxZ0cBaYdEhFg==,iv:Hn7c...,tag:2mWq...,type:str]
sops_age__list_0__map_enc=-----BEGIN AGE ENCRYPTED FILE-----\n...
sops_age__list_0__map_recipient=age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
sops_lastmodified=2026-10-13T20:27:13Z
sops_mac=ENC[AES256_GCM,data:...]
sops_unencrypted_suffix=_unencrypted
sops_version=3.13.3
```

`hunter2` is the *plaintext*, the readable original. Everything inside `ENC[...]` is the *ciphertext*, the encrypted form that's unreadable without the key. The rest of this post uses both terms.

[age](https://github.com/FiloSottile/age) is the other half. It's a small encryption tool built around a single keypair, with none of GPG's keyrings or trust model. The two tools split the work in a pattern called *envelope encryption*. sops generates a random data key for each file and encrypts every value with it. age then encrypts only that data key to your public key, and the result is the `sops_age__list_0__map_enc` block at the bottom of the file. To read the file, sops uses your age private key to unlock the data key, then uses the data key to decrypt the values. That's two layers of encryption, but each value is only encrypted once. The second layer protects the key, not the values.

Encrypting values instead of the whole file has some useful consequences:

- **Diffs stay meaningful.** Variable names are plaintext, so `git diff` shows that `DB_PASSWORD` changed, even though it can't show what it changed to. You get a readable history without leaking anything.
- **Tampering is detected, not just hidden.** `sops_mac` is a checksum over every value, and each value is also individually authenticated. Change one character of ciphertext and decryption refuses outright:

  ```text
  Error decrypting tree: Error walking tree: Could not decrypt value: Could not decrypt with AES_GCM: cipher: message authentication failed
  ```

- **Without the key, you get nothing.** Not a partial file, and not a garbled one:

  ```text
  Recovery failed because no master key was able to decrypt the file. In
  order for SOPS to recover the file, at least one key has to be successful,
  but none were.
  ```

### Why sops and not something else

The other common options each miss something for this use case:

- **git-crypt** encrypts whole files transparently. Diffs of an encrypted file are binary noise, so you lose the which-variable-changed history.
- **Vault, Infisical, or another secrets server** means running a service that your other services depend on to start. In a homelab, the secrets server is usually one of the containers it's supposed to be unlocking.
- **Docker Compose `secrets:`** solves getting a secret *into* a container as a file. It doesn't solve where that file comes from or how it's backed up.

sops is a single binary, and it works offline with nothing to host. The encrypted files live in the same repo and follow the same review and history as everything else.

## Prerequisites

You need two tools: `sops` and `age` (which includes `age-keygen`).

On Arch, both are in the official repos:

```bash
sudo pacman -S sops age
```

On Fedora, `age` is packaged, but sops isn't. The sops project publishes an `.rpm` with each [release](https://github.com/getsops/sops/releases), and `dnf` can install it directly from the URL:

```bash
sudo dnf install age
sudo dnf install https://github.com/getsops/sops/releases/download/v3.13.3/sops-3.13.3-1.x86_64.rpm
```

On anything else, sops also ships as a static binary (`sops-v3.13.3.linux.amd64`) that you can drop into `~/.local/bin`.

Check that both are on your `PATH`:

```bash
sops --version
age --version
```

## Generating a key

sops looks for age keys in `~/.config/sops/age/keys.txt` by default on Linux, so generate the key there:

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

```text
Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

`age-keygen` creates the file with `600` permissions already, so only your user can read it. Inside, it looks like this:

```text
# created: 2026-10-13T13:56:50-07:00
# public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
AGE-SECRET-KEY-1...
```

The difference between those two lines is the most important thing in this post:

- **`age1...` is the public key**, also called the *recipient*. It can only encrypt. It's safe to commit, post, or paste anywhere. It goes in `.sops.yaml`.
- **`AGE-SECRET-KEY-1...` is the private key.** It decrypts every file ever encrypted to that recipient. It never goes in git.

If you'd rather keep the key somewhere else, point sops at it with `SOPS_AGE_KEY_FILE=/path/to/keys.txt`. Otherwise, the default location means sops finds it without any configuration.

## Backing up the key

Before encrypting anything, decide on where the key lives. If you lose it the enccrypted data is unrecoverable. sops has no reset and no recovery question. If no copy of the private key exists, every value encrypted to it is gone, in every repo, and you're back to resetting each password through each service's own recovery flow. That's the exact situation we want to avoid.

My key lives in three places:

1. **My development machine**, at `~/.config/sops/age/keys.txt`, for creating and editing secrets.
2. **The server**, which needs it to decrypt `.env` files at deploy time (more on that [below](#using-encrypted-files-with-docker-compose)).
3. **My password manager.** Many password managers let you add a custom field with a label and mark it hidden. I store the `AGE-SECRET-KEY-1...` line in a hidden field and the `age1...` public key in a normal one for reference.

The password manager is the copy that matters for recovery. If the server and the laptop both die, the key comes back from there, the repo comes back from GitHub, and every secret is readable again.

## Telling sops what to encrypt: `.sops.yaml`

Rather than passing the recipient on every command, put a `.sops.yaml` in the root of the repo:

```yaml
creation_rules:
  - path_regex: \.enc\.env$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

When sops encrypts a file, it looks for `.sops.yaml` starting in your *current directory* and walking up, not starting from the file's location, so run sops from inside the repo. It then uses the first rule whose `path_regex` matches the filename. Commit this file. It only contains the public key, and anyone cloning the repo needs it to encrypt new files to the right recipient.

The rule matches on the *filename*. It isn't just a convenience, but a way to prevent encrypting the wrong files. If no rule matches, sops refuses to encrypt:

```text
error loading config: no matching creation rules found
```

That detail shapes how you should name things.

## Name encrypted files so mistakes fail safe

My homelab repository names its encrypted files plain `.env`, with a `path_regex` of `\.env$`. It works, and doco-cd reads them without issue. But there's a gap I only noticed later.

Because the encrypted files are called `.env`, `.env` has to be committed. So `.gitignore` can't list `.env`, and git has no way to tell an encrypted `.env` from a plaintext one. If you ever `cp .env.sample .env` and fill it in, planning to encrypt it in a minute, a `git add .` in the meantime commits real passwords without any warning. Nothing went wrong for me, but the only thing preventing it is remembering.

A later project fixed this by giving encrypted files their own suffix and ignoring everything else:

```gitignore
# .gitignore
.env
```

```yaml
# .sops.yaml
creation_rules:
  - path_regex: \.enc\.env$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

Now the two kinds of file can't be confused. Plaintext is always `.env`, which git ignores. Ciphertext is always `.enc.env`, which git tracks. Forgetting to encrypt means the file never leaves your machine. The mistake does nothing instead of leaking. If I were starting the homelab repo today, this is the naming I'd use.

## Creating and editing encrypted files

Keep a template with blank values next to each encrypted file. It's the `.env.sample` from before, and it still has a job: documenting what each variable is and where its value comes from.

```env
# .env.sample
DB_USERNAME=
# Letters and numbers only. Generate with: openssl rand -hex 32
DB_PASSWORD=
DB_DATABASE_NAME=
```

To create the encrypted file, copy the template, fill it in, and encrypt it:

```bash
cp .env.sample .env               # gitignored
$EDITOR .env                      # fill in real values
sops encrypt --filename-override app.enc.env .env > app.enc.env
rm .env
```

`--filename-override` is what makes this work with the naming scheme. The input file is called `.env`, which doesn't match `\.enc\.env$`, so without the flag you'd get the `no matching creation rules found` error. The flag tells sops to choose a rule (and the dotenv format) as if the file were named `app.enc.env`.

After that, you never handle plaintext on disk again. Use these commands:

```bash
sops edit app.enc.env      # decrypt to a temp file, open $EDITOR, re-encrypt on save
sops decrypt app.enc.env   # print plaintext to stdout
```

`sops edit` only re-encrypts the values you changed, so the commit diff is limited to those lines plus the updated `sops_mac`.

The `sops_*` lines at the bottom of the file are sops' own metadata:

- **`sops_age__list_0__map_recipient`** records which public key the file was encrypted to.
- **`sops_age__list_0__map_enc`** is the age-encrypted data key.
- **`sops_mac`** is the tamper check.
- **`sops_version`** and **`sops_lastmodified`** are bookkeeping.

Commit them with the file. Without them, sops can't decrypt it.

If you're coming from the `.env` + `.env.sample` layout in the [previous post]({{< ref "organizing-docker-compose-files.md" >}}), encrypt each existing `.env` to `.enc.env` with the command above, and keep the `.env.sample` files as your templates. Then clone the repo somewhere fresh and run `sops decrypt` on a few files to make sure everything you need actually made it into git.

## Using encrypted files with Docker Compose

Docker Compose can't read sops files. Point `--env-file` at an `.enc.env` and you get every variable set to a string of ciphertext. Something has to decrypt first.

The manual way is `sops exec-env`, which decrypts a file into a child process's environment and runs a command. Plaintext never touches disk:

```bash
sops exec-env app.enc.env 'docker compose up -d'
```

Compose uses shell environment variables for `${...}` interpolation, so the decrypted values fill in the compose file just as an `.env` file would.

It handles one file at a time, though. The layout from the previous post combines a `shared.env` with each app's own `.env`, and nesting `exec-env` calls for every deploy gets tedious fast. I never actually ran my stacks this way. I went straight to having a deployment tool decrypt on every deploy, and that's the subject of the next post in this series.

Whatever does the decrypting, the server needs the private key. Copy it over and lock it down:

```bash
ssh server 'mkdir -p ~/.config/sops/age'
scp ~/.config/sops/age/keys.txt server:~/.config/sops/age/keys.txt
ssh server 'chmod 600 ~/.config/sops/age/keys.txt'
```

Keep it outside any git repository on the server. The key is what decrypts the repo, so it can't live *in* the repo.

## Takeaway

Secrets can live in git, as long as their values are only ever committed as ciphertext. sops encrypts the values and leaves the variable names readable, so you keep a meaningful history, a real backup, and a single commit for a change that spans a compose file and its secrets.

The whole arrangement comes down to one key. Back it up somewhere that survives every machine you own, never commit the private half, and name your files so that forgetting to encrypt fails safe instead of leaking.

<!-- TODO: link to the doco-cd post here once it's merged to main -->
