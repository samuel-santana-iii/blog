+++
date = '2026-07-10T06:32:04-07:00'
draft = false
title = 'GitOps in Practice: Automating My Blog Deployment Pipeline'
+++


I wanted a place to document my projects, things like Kubernetes clusters, Linux setups, and homelab experiments. Somewhere I could reference my own work and maybe help someone else along the way. I'd built personal sites with WordPress before and knew going in that it would mean ongoing maintenance: plugin updates, core updates, and the occasional breakage that had nothing to do with anything I wrote. For a simple technical blog, that overhead wasn't worth it.

That's where I came across Hugo. With Hugo I built a pipeline where I can write, commit, and push. Building, publishing, and deploying happens on its own. There's no server to maintain, no database to back up, and no platform eating into the time I'd rather spend on the work itself.

This post walks through exactly how that works: Hugo on GitHub Pages, wired up with GitHub Actions, with a custom domain. The whole thing is GitOps: the repository is the source of truth, and a push to `main` is the deployment trigger.

## Deployment Architecture

The flow is simple and fully automated once set up:

```
Write post
  → git push to main
  → GitHub Action triggers
  → Hugo builds
  → Artifact uploaded
  → GitHub Pages deploys
  → Live site
```

There are no manual steps after the push. The GitHub Actions workflow handles the build-and-deploy in two jobs: `build` (which compiles the site and uploads the artifact) and `deploy` (which pushes it to GitHub Pages). The `deploy` job is gated on `build` completing successfully, so a broken build never goes live.

## Stack and Why

**Hugo** is a static site generator written in Go. Builds are fast and the output is plain HTML, CSS, and JS with no runtime dependencies. Place these files on a web server, and the website is live.

**GitHub Pages** was the obvious hosting choice given the code is already in GitHub. The Pages integration is first-class: GitHub provides a dedicated `github-pages` environment, an OIDC-based deploy token (no stored secrets needed), and the `actions/deploy-pages` action handles the upload. It also supports custom domains with free HTTPS via Let's Encrypt.

**PaperMod** is a clean, fast Hugo theme with good defaults. It supports share buttons, post navigation links, and a simple home info section. This is everything I want in a minimal blog without having to build it myself. It ships as a git submodule, which keeps the theme versioned and the main repo clean.

## Setting Up a New Hugo Project

If you're starting from scratch, install Hugo and scaffold the site:

```bash
hugo new site blog
cd blog
git init
```

Then add PaperMod as a submodule. Using a submodule (rather than copying the theme files in) means the exact commit you're on is tracked in git, and you control if and when you take updates:

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Set the theme in `hugo.toml`:

```toml
baseURL = 'https://yourdomain.com/'
locale = 'en-us'
title = "Your Blog Title"
theme = "PaperMod"

[params]
  env = 'production'
  ShowReadingTime = 'true'
  ShowShareButtons = 'true'
  ShowPostNavLinks = 'true'

  [params.homeInfoParams]
    Title = "Your Name"
    Content = "What you write about"
```

Create your first post:

```bash
hugo new content posts/my-first-post.md
```

Hugo scaffolds the frontmatter. Set `draft = false` when you're ready to publish.

### Pinning the PaperMod Version

When you run `git submodule add`, git records the exact commit of PaperMod at that moment in your parent repo. That pointer is your pin. Anyone who clones your repo and runs `git submodule update --init` gets that exact commit, not whatever is latest on PaperMod's master branch. This is intentional. PaperMod doesn't publish versioned releases, so master can introduce breaking changes at any time. Locking to a known-good commit keeps your build stable.

The distinction to know:

```bash
git submodule update          # checks out the pinned commit — what you almost always want
git submodule update --remote # pulls the latest from upstream master — use deliberately
```

If you do want to take a PaperMod update, run `git submodule update --remote`, verify the build still works locally with `hugo server`, then commit the updated submodule pointer:

```bash
git add themes/PaperMod
git commit -m "Update PaperMod to latest"
```

That commit becomes your new pin. One thing to be aware of: a newer version of PaperMod may require a newer version of Hugo. If your build fails after updating the submodule, check the PaperMod changelog and bump `HUGO_VERSION` in `.github/workflows/hugo.yml` accordingly.

## GitHub Actions Workflow

GitHub provides a starter workflow for Hugo that covers the basics. To use it, go to your repo → **Actions** → **New workflow** → search for "Hugo" → select **Deploy Hugo site to GitHub Pages**. This generates `.github/workflows/hugo.yml` pre-configured for Pages deployment. This workflow runs every time a new commit is added to the main branch.

The workflow runs two sequential jobs, `build` then `deploy`, so a failed build never touches the live site. Here's what each step does and why it matters:

1. **Install Hugo (pinned version)** — downloads a specific Hugo release rather than whatever happens to be latest. Pinning prevents a surprise breaking change in Hugo from taking down your pipeline.
2. **Install Dart Sass** — compiles SCSS to CSS. PaperMod uses SCSS for its styles; without this step the build errors out before producing any output.
3. **Checkout with `submodules: recursive`** — clones your repo and pulls in PaperMod from its own upstream repository. Without this flag, the `themes/PaperMod` directory lands empty and Hugo can't find the theme.
4. **Configure Pages** — sets up the GitHub Pages environment and outputs the correct `base_url` for your site, which Hugo needs to generate valid absolute URLs.
5. **Build with `--minify`** — compiles all markdown posts through PaperMod's templates into plain HTML, CSS, and JS in `./public`. The `--minify` flag strips whitespace to reduce file sizes.
6. **Upload artifact** — packages `./public` and hands it off to GitHub's artifact storage so the `deploy` job can retrieve it.
7. **Deploy** — pushes the artifact to the GitHub Pages environment using an OIDC token scoped to this deployment, so no credentials need to be stored as secrets.

The starter workflow works out of the box with two changes. The default `HUGO_VERSION` it ships with was not recent enough to support PaperMod, causing cryptic template errors during the build. PaperMod requires Hugo Extended for SCSS processing, and newer theme features depend on a current release. The fix was to bump the version and ensure the `hugo_extended` variant is downloaded:

```yaml
env:
  HUGO_VERSION: 0.146.0  # bumped to meet PaperMod's requirements
```

The second change was adding `submodules: recursive` to the checkout step, since the starter workflow doesn't include it by default:

```yaml
- name: Checkout
  uses: actions/checkout@v4
  with:
    submodules: recursive  # required to pull in PaperMod
```

If you see errors like `template: ... function "xxxx" not defined` during the build step, a Hugo version bump is the first thing to check.

## Domain and DNS Wiring

GitHub Pages supports custom domains with free HTTPS, which means you're not locked into a `yourusername.github.io` URL and you don't need to manage a certificate yourself.

The setup is a two-step handshake between GitHub and your DNS provider.

**1. Add the domain in GitHub**

In your repo: Settings → Pages → Custom domain. Enter your domain (e.g., `blog.yourdomain.com`) and save. GitHub will attempt to verify DNS and provision a TLS certificate automatically once the DNS record is in place.

**2. Create a CNAME record at your DNS provider**

| Type  | Name | Value                      |
|-------|------|----------------------------|
| CNAME | blog | yourusername.github.io     |

A CNAME works for subdomains like `blog.yourdomain.com`. For a root domain (`yourdomain.com` with no subdomain), DNS spec doesn't allow CNAMEs at the root, so GitHub provides four A records to use instead. In my case, the root domain was already in use for my portfolio site, so a subdomain was the natural choice.

DNS propagation typically takes a few minutes to a few hours. Once it resolves, GitHub automatically provisions the Let's Encrypt certificate. Make sure "Enforce HTTPS" is checked in the Pages settings once the certificate is active.

Also update `baseURL` in `hugo.toml` to your custom domain so Hugo generates correct absolute URLs in the sitemap and canonical tags.

## Takeaway

The total ongoing cost is zero and the deployment overhead is zero. Writing a post is now: open editor, write markdown, push. The pipeline handles the rest.

The GitOps part is what makes this worth it beyond just convenience. The git history records every post, every edit, and every draft that got published. Rolling back a bad deploy is a `git revert` away. And because there's no server, there's also no server to patch, no uptime to monitor, and nothing to break at 2am.

For a personal blog focused on technical content, this stack is close to ideal.
