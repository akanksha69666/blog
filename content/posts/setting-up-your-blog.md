---
title: "Setting Up This Blog: Hugo, GitHub Actions, and Cloudflare Pages"
date: 2026-10-25T16:53:52+05:30
draft: false
tableOfContents: true
description: "How this site is put together: Hugo for the build, Cloudflare Pages for hosting, and GitHub Actions to glue the two together."
categories:
  - Tools
tags:
  - Hugo
  - Cloudflare
  - GitHub Actions
  - Static Sites
---

## Preface

I've been meaning to write things down for a while now. Not the polished, "here's a tutorial" kind of writing, more like notes from whatever I'm currently breaking my head against. The problem was always the same: setting up a blog felt like a side project in itself.

So this first post is about that side project. How this site is built, how a `git push` turns into a live page, and the bits that tripped me up along the way. If you're planning to do the same, you can probably skip a few of my detours.

The stack is deliberately boring:

- **Hugo** builds the site. It's fast, it's a single binary, and content is just Markdown.
- **[Lowkey](https://github.com/nixentric/Lowkey-Hugo-Theme)** is the theme. Clean, minimal, gets out of the way.
- **Cloudflare Pages** hosts it. Free, fast, and no server to babysit.
- **GitHub Actions** does the building and deploying, so I never run a deploy command by hand.

That's it. No database, no CMS, no Docker. The whole pipeline is essentially `hugo --minify` followed by an upload.

## What you'll need

Nothing exotic here:

- Hugo **extended** (v0.168.0 at the time of writing)
- Node and npm — the theme pipes its CSS through PostCSS and Tailwind, so the build needs them
- Git, and a GitHub account
- A Cloudflare account (the free plan is more than enough)
- The `gh` CLI is optional, but it makes the secrets part less clicky

On macOS, Hugo comes from Homebrew:

```bash
brew install hugo
hugo version
```

Make sure the version string says `extended`. The non-extended build will work for a lot of themes, but you don't want to find out halfway through a deploy that it doesn't.

## Scaffolding the site

Creating a Hugo site is a one-liner:

```bash
hugo new site blog
cd blog
git init
```

Hugo generates the usual directories — `content/`, `layouts/`, `static/`, `themes/` — and a config file. You'll notice there's no content yet. That's next.

## Adding the theme

The theme lives in its own repository, and I wanted to be able to pull updates without vendoring a copy into my repo, so it goes in as a git submodule:

```bash
git submodule add https://github.com/nixentric/Lowkey-Hugo-Theme.git themes/lowkey
```

> **Gotcha:** a submodule is a pointer, not the files. If someone clones the repo without `--recurse-submodules`, `themes/lowkey` will be empty and the build will fail with a confusing "template not found" error. Clone with `git clone --recurse-submodules`, or run `git submodule update --init` afterwards.

Then point Hugo at it in `config/_default/hugo.toml`:

```toml
theme = 'lowkey'
title = 'Akanksha'
baseURL = 'https://akankshablog.pages.dev/'
```

The theme ships with an `ExampleSite` folder containing a working config. I copied the structure (`hugo.toml`, `menus.toml`, `params.toml`) and trimmed it down instead of starting from scratch. Menus, social links, avatar, description — it's all in there, and it's easier to delete what you don't need than to guess what the theme expects.

## Writing the first post

Hugo's archetypes handle the front matter:

```bash
hugo new posts/setting-up-your-blog.md
```

Which gives you something like this:

```yaml
---
title: "Setting Up Your Blog"
date: 2026-10-25T16:53:52+05:30
draft: true
description: ""
categories:
  - Uncategories
tags:
  -
---
```

Drafts are excluded from the build by default, which is exactly what you want while writing. Preview with the draft flag:

```bash
hugo server -D
```

The `-D` matters. Without it you'll stare at an empty post list and wonder what broke. The dev server also hot-reloads, so you can keep the browser open while you edit.

For the assets to build locally, install the theme's frontend dependencies once:

```bash
npm install
```

> If `npm install` complains about peer dependencies, check the Node version. I'm on Node 22 and the toolchain is happy there.

## Pushing to GitHub

Nothing special here, just the usual:

```bash
git add .
git commit -m "chore: initial hugo setup"
gh repo create blog --private --source=. --push
```

Or create the repo on github.com and add the remote manually. Either works. The important part is that `main` exists and has your code, because that's what the deploy workflow will watch.

## Creating the Cloudflare account and project

Sign up at [dash.cloudflare.com](https://dash.cloudflare.com). You don't need a domain, and you don't need to add a site to Cloudflare — Pages works fine on its own `*.pages.dev` subdomain.

Now, there are two ways to put a Hugo site on Pages. One is Cloudflare's Git integration, where you connect the repo in the dashboard and Cloudflare runs the build for you. The other is building in GitHub Actions and uploading the output with Wrangler. I went with the second one, mostly because I wanted the build to be reproducible and visible in the same place as everything else.

Since we're uploading directly, the Pages project has to exist first. Authenticate Wrangler and create it:

```bash
npx wrangler login
npx wrangler pages project create akankshablog --production-branch main
```

`akankshablog` is the project name. It becomes part of your URL, so pick something you're okay with seeing in a browser bar: `https://akankshablog.pages.dev`.

> Wrangler can create a project on the fly during an interactive deploy, but in CI nobody's there to answer the prompt. Create it once locally and save yourself the failed run.

## Giving GitHub permission to deploy

The workflow needs two things from Cloudflare: an API token and the account ID.

For the token, go to **My Profile → API Tokens → Create Token → Create Custom Token**, and give it:

- **Permissions:** Account → Cloudflare Pages → Edit
- **Account Resources:** include your account

Copy the token — Cloudflare only shows it once. The account ID is on the **Workers & Pages** overview page, or in the dashboard URL after `/accounts/`.

Then store both as GitHub Actions secrets:

```bash
gh secret set CLOUDFLARE_API_TOKEN
gh secret set CLOUDFLARE_ACCOUNT_ID
```

Or, if you prefer the UI: **Repo → Settings → Secrets and variables → Actions → New repository secret**.

> Don't put the token in the workflow file, and don't commit a `.env`. Wrangler reads `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` straight from the environment, which is exactly what Actions gives it.

## The GitHub Actions workflow

Here's the whole thing, at `.github/workflows/deploy.yml`:

```yaml
name: Build and deploy

on:
  push:
    branches: [main]
    paths:
      - "config/**"
      - "content/**"
      - ".github/workflows/deploy.yml"

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
      CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
      CLOUDFLARE_PROJECT_NAME: akankshablog

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Install Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.168.0"
          extended: true

      - name: Install Node
        uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build site
        run: hugo --minify

      - name: Install Wrangler
        run: npm install --global wrangler

      - name: Deploy to Cloudflare Pages
        run: wrangler pages deploy public --project-name "$CLOUDFLARE_PROJECT_NAME" --branch main
```

A few details worth calling out:

- `submodules: recursive` is not optional. Without it, the checkout has no theme.
- `npm ci` instead of `npm install` — it installs exactly what's in `package-lock.json`, so CI and my laptop agree.
- `hugo --minify` writes the finished site to `public/`, which is what gets uploaded.
- `--branch main` tells Pages this is a production deploy, not a preview.

The `paths` filter means the workflow only runs when config, content, or the workflow itself changes. It keeps the Actions tab quiet, but there's a catch: changes to `themes/**` won't trigger it. If you bump the theme submodule, either add that path or push an empty commit. I learned that the uninteresting way.

## The first deploy, and the `baseURL` trap

Push, open the Actions tab, and watch it go. If everything's wired up, the last step prints a URL, and that's your site.

If the page loads but looks like unstyled HTML, check `baseURL`. Hugo uses it to generate absolute asset paths, and if it's still pointing at `http://example.org/`, every CSS and JS request 404s. Annoyingly, `hugo server` overrides `baseURL` locally, so the site looks perfect on your machine and broken in production. Set it to the real Pages URL *before* the first deploy:

```toml
baseURL = 'https://akankshablog.pages.dev/'
```

That was my first "fix" commit. Hopefully you get to skip it.

## Where this leaves us

From here on, the loop is: write Markdown, `git push`, wait a minute, refresh. No build step to remember, no FTP, no server to patch. The whole thing costs nothing and will keep working even if I ignore it for six months.

There are things I'd still like to add — a custom domain, maybe comments, an RSS reader setup that isn't embarrassing — but none of that is blocking the actual writing, which was the point.

If you're on the fence about starting a blog: the setup is an afternoon, and most of that is picking a theme. The hard part is the second post. 🙂
