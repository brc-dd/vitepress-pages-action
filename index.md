---
outline: deep
---

# Hello VitePress

To configure your [VitePress site](https://vitepress.vuejs.org/guide/getting-started 'Getting Started') to publish with GitHub Actions:

1. On GitHub, navigate to your site's repository.

2. Under your repository name, click **Settings**.

![Repository settings button](https://docs.github.com/assets/cb-27528/images/help/repository/repo-actions-settings.png)

3. In the "Code and automation" section of the sidebar, click **Pages**.

4. Under "Build and deployment", under "Source", select **GitHub Actions**.

![Source selection](https://user-images.githubusercontent.com/14911070/178842638-51b834d3-6c54-423e-95fa-822f734fa98a.png){style=max-width:300px;margin:auto}

If that option is not showing up for you, then refer the [official deploying guide](https://vitepress.vuejs.org/guide/deploying#github-pages) instead.

5. Click on **Create your own** below the "Source".

6. Give your file some name like `pages.yml` and replace its content with this:

```yaml
# Sample workflow for building and deploying a VitePress site to GitHub Pages
#
# To get started with VitePress see: https://vitepress.vuejs.org/guide/getting-started
#
name: Deploy VitePress site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches: [main]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

env:
  # Directory where your VitePress site is
  VP_ROOT: docs

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow one concurrent deployment
concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Detect package manager
        run: echo "PM=$(jq -r '.packageManager // "npm"' package.json | cut -d '@' -f 1)" >> $GITHUB_ENV
      - name: Install pnpm
        if: env.PM == 'pnpm'
        uses: pnpm/action-setup@v2
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: lts/*
          cache: ${{ env.PM }}
      - name: Setup Pages
        uses: actions/configure-pages@v1
      - name: Install dependencies
        run: ${{ env.PM }} ${{ env.PM == 'npm' && 'ci' || 'install --frozen-lockfile' }}
      - name: Build site using VitePress
        run: ${{ env.PM == 'npm' && 'npx --no-install' || env.PM }} vitepress build ${{ env.VP_ROOT }} --base /${{ github.event.repository.name }}/
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v1
        with:
          path: ${{ env.VP_ROOT }}/.vitepress/dist

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v1
```

7. Commit it and you're ready to go!

## Advanced Configuration

- Depending upon your need, in the workflow file, you might need to change the triggering branch (currently `main`) and/or the `VP_ROOT` variable.

- If you're deploying to some custom domain or your root repository (`github.com/user-name/user-name.github.io`), then you might need to change or remove the [`base`](https://vitepress.vuejs.org/config/app-configs#base) flag in the `build` job. It currently adds your repository name as the base.

- The workflow currently guesses your package manager using the [`packageManager`](https://nodejs.org/api/packages.html#packagemanager) field. If its not present then `npm` is assumed. I have **NOT** added any logic to check what kind of lock files are present and detect your package manager based on that.

- The workflow can be highly simplified according to need:

### For NPM

Assuming your VitePress root is `docs` and you already have proper `base` set in `docs/.vitepress/config.ts`. Also, that you have a script named `docs:build` in your `package.json` that builds your site. You can customize these as per your need.

```yaml
name: build and deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: lts/*
          cache: npm
      - uses: actions/configure-pages@v1
      - run: npm ci && npm run docs:build
      - uses: actions/upload-pages-artifact@v1
        with:
          path: docs/.vitepress/dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/deploy-pages@v1
        id: deployment
```

### For Yarn

Replace `cache: npm` with `cache: yarn` in the above code, and change `npm ci && npm run docs:build` to `yarn --frozen-lockfile && yarn docs:build`. If you're using Yarn Berry, then you might wanna use [`--immutable --immutable-cache`](https://yarnpkg.com/cli/install) instead of `--frozen-lockfile`.

### For PNPM

Replace `cache: npm` with `cache: pnpm` and use [`pnpm/action-setup`](https://github.com/pnpm/action-setup) before `setup-node`. Change `npm ci && npm run docs:build` to `pnpm i --frozen-lockfile && pnpm docs:build`.
