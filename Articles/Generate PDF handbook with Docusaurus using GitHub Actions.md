---
canonicalUrl: https://dev.to/opensauced/generate-pdf-handbook-with-docusaurus-using-github-actions-lfb
tags:
- actionshackathon21
- GitHub
- docker
- node
---

# # Generate PDF handbook with Docusaurus using GitHub Actions

![[protsessory-oboi-3840x1440_92.jpg]]

## Motivation

Having [@open-sauced docs](https://docs.opensauced.pizza) built using [Docusaurus](https://docusaurus.io), we started exploring the plugin ecosystem and identified various improvements we could apply.

One of the community plugins we found during that process was [signcl/docusaurus-prince-pdf](https://github.com/signcl/docusaurus-prince-pdf), an npm package leveraging [sindresorhus/got](https://github.com/sindresorhus/got) to crawl all the documentation and generate a PDF version.

Having a portable document generated as a [downloadable release asset](https://docs.opensauced.pizza/open-sauced-docs.pdf), we could more easily share the entirety of our docs and have that document available for offline use.

We faced the challenge of having to access the upcoming version during the deployment workflow, installing additional binaries, and deploying the generated asset as part of the release.

## My Workflow

[Note]: # (Please make sure the project links to the appropriate GitHub Actions repository, and includes [an open source license](https://choosealicense.com/) and README.)

The full workflow is available here: [.github/workflows/release.yml](https://github.com/open-sauced/docs.opensauced.pizza/blob/main/.github/workflows/release.yml)

In order to generate a handbook out of a [Docusaurus](https://docusaurus.io) instance we need to build the application in a container before uploading it as a build artifact for later workflow steps.

This is all taken care of in the new `docker` job that was created for the hackaton:

```yaml
jobs:
  docker:
    name: Build container
    runs-on: ubuntu-latest
    steps:
      - name: "☁️ checkout repository"
        uses: actions/checkout@v2

      - name: "🔧 setup buildx"
        uses: docker/setup-buildx-action@v1

      - name: "🔧 cache docker layers"
        uses: actions/cache@v2
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      - name: "🔧 docker meta"
        id: meta
        uses: docker/metadata-action@v3
        with:
          images: ${{ github.repository }}
          tags: latest

      - name: "📦 docker build"
        uses: docker/build-push-action@v2
        with:
          context: .
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          outputs: type=docker,dest=/tmp/docker.tar
          push: false
          cache-from: type=gha, scope=${{ github.workflow }}
          cache-to: type=gha, scope=${{ github.workflow }}

      - name: "📂 docker artifacts"
        uses: actions/upload-artifact@v2
        with:
          name: docker
          path: /tmp/docker.tar
```

Concurrently, the `build` job prepares static `npm` assets and uploads them as an artifact for a subsequent job:

```yaml
jobs:
  build:
    name: Build application
    runs-on: ubuntu-latest
    steps:
      - name: "☁️ checkout repository"
        uses: actions/checkout@v2

      - name: "🔧 setup node"
        uses: actions/setup-node@v2.1.5
        with:
          node-version: 16

      - name: "🔧 install npm@latest"
        run: npm i -g npm@latest

      - name: "📦 install dependencies"
        uses: bahmutov/npm-install@v1

      - name: "🚀 static app"
        run: npm run build

      - name: "📂 production artifacts"
        uses: actions/upload-artifact@v2
        with:
          name: build
          path: build
```

We then move to the `release` job, downloading all the artifacts and running our custom configuration [@open-sauced/semantic-release-conventional-config](https://github.com/marketplace/actions/open-sauced-semantic-release-conventional-config) from a [docker socket](https://docs.github.com/en/actions/creating-actions/dockerfile-support-for-github-actions):

```yaml
jobs:
  release:
    environment:
      name: production
      url: https://github.com/${{ github.repository }}/releases/tag/${{ env.RELEASE_TAG }}
    name: Semantic release
    needs:
      - docker
      - build
    runs-on: ubuntu-latest
    steps:
      - name: "☁️ checkout repository"
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: "📂 download docker artifacts"
        uses: actions/download-artifact@v2
        with:
          name: docker
          path: /tmp

      - name: "📦 load tag"
        run: |
          docker load --input /tmp/docker.tar
          docker image ls -a

      - name: "📂 download build artifacts"
        uses: actions/download-artifact@v2
        with:
          name: build
          path: /tmp/build

      - name: "🚀 release"
        id: semantic-release
        uses: docker://ghcr.io/open-sauced/semantic-release-conventional-config:3.0.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

The next step is a little verbose, in order to crawl our [docs website](https://docs.opensauced.pizza) we need to:
- mount the previously built container on port `8080`
- install [Prince 14](https://www.princexml.com)
- run [docusaurus-prince-pdf](https://github.com/signcl/docusaurus-prince-pdf)
- deploy to static [gh-pages](https://pages.github.com)

The `deploy` job is doing all the heavy lifting:

```yaml
jobs:
  deploy:
    name: Deploy to static
    needs:
      - build
      - release
    runs-on: ubuntu-latest
    services:
      docs:
        image: ghcr.io/${{ github.repository }}:latest
        ports:
          - 8080:80
    steps:
      - name: "☁️ checkout repository"
        uses: actions/checkout@v2

      - name: "📂 download artifacts"
        uses: actions/download-artifact@v2
        with:
          name: build
          path: /home/runner/build

      - name: Install Prince
        run: |
          curl https://www.princexml.com/download/prince-14.2-linux-generic-x86_64.tar.gz -O
          tar zxf prince-14.2-linux-generic-x86_64.tar.gz
          cd prince-14.2-linux-generic-x86_64
          yes "" | sudo ./install.sh

      - name: "🔧 setup node"
        uses: actions/setup-node@v2.1.5
        with:
          node-version: 16

      - name: "🔧 install npm@latest"
        run: npm i -g npm@latest

      - name: "📦 install dependencies"
        uses: bahmutov/npm-install@v1

      - name: "📂 copy artifacts"
        run: cp -R /home/runner/build .

      - name: "🚀 generate pdf"
        run: npm run pdf

      - name: "🚀 deploy static"
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build
          commit_message: ${{ github.event.head_commit.message }}
          enable_jekyll: false
          cname: docs.opensauced.pizza
```

As a final step, we need to clean up our `build` and `docker`  artifacts, a simple but necessary step:

```yaml
jobs:
  cleanup:
    name: Cleanup actions
    needs:
      - deploy
    runs-on: ubuntu-latest
    steps:
      - name: "♻️ remove build artifacts"
        uses: geekyeggo/delete-artifact@v1
        with:
          name: |
            build
            docker
```

## Submission Category: DIY Deployments

## Yaml File or Link to Code

Live repository using this workflow:

{% github open-sauced/docs.opensauced.pizza %}

Yaml file link: 
[@open-sauced/docs.opensauced.pizza/main/.github/workflows/release.yml](https://github.com/open-sauced/docs.opensauced.pizza/blob/main/.github/workflows/release.yml)

## Additional Resources / Info

Here are all the open source actions we are using to power this release workflow:

- [actions/checkout@v2](https://github.com/actions/checkout) - most performant git checkout
- [actions/setup-node@v2.1.5](https://github.com/actions/setup-node) - we use it to set the `node` version to 16
- [actions/upload-artifact@v2](https://github.com/actions/upload-artifact) - we use it to transport our artifacts in between jobs
- [actions/download-artifact@v2](https://github.com/actions/download-artifact) - we use it to download our artifacts in between jobs
- [docker/setup-buildx-action@v1](https://github.com/docker/setup-buildx-action) - we use it to set up the docker builder
- [actions/cache@v2](https://github.com/actions/cache) - we use it to cache docker layers
- [docker/metadata-action@v3](https://github.com/docker/metadata-action) - we use it to normalize most of our docker container values
- [docker/build-push-action@v2](https://github.com/docker/build-push-action) - we use it to build the container
- [bahmutov/npm-install@v1](https://github.com/bahmutov/npm-install) - lightning-fast `npm ci` with built-in cache
- [open-sauced/semantic-release-conventional-config@v3](https://github.com/open-sauced/semantic-release-conventional-config) - semantic-release configuration, docker container and GitHub action
- [peaceiris/actions-gh-pages@v3](https://github.com/peaceiris/actions-gh-pages) - deploys any folder(s) to `gh-pages`, can use it for multiple static endpoints
- [geekyeggo/delete-artifact@v1](https://github.com/GeekyEggo/delete-artifact) - deletes produced artifacts

Be sure to include the DEV usernames of your collaborators:
{% user mtfoley %}
