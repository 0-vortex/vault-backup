---
canonicalUrl: https://dev.to/opensauced/semantic-release-to-npm-andor-ghcr-without-any-tooling-5730
tags:
- actionshackathon21
- GitHub
- docker
- node
---

# Semantic release to npm and/or ghcr without any tooling

![[fractal_185-wallpaper-1920x1080.jpg]]

## Motivation

Having our semantic release process available as a scoped package was a useful practice but it became obvious that having it installed in our development dependencies across multiple repositories would pose a challenge for other maintainers while increasing our build times.

The only thing that could simplify this process was to have all the release dependencies offloaded to a [Docker container action](https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action) we could tag major versions and reduce maintenance cost by deploying release configuration improvements without touching workflows.

## My Workflow
[Note]: # (Please make sure the project links to the appropriate GitHub Actions repository, and includes [an open source license](https://choosealicense.com/) and README.)

This action is a thoroughly engineered semantic-release shareable configuration, meant to simplify configuration and workflow environment variables to just `GITHUB_TOKEN` and, if you are deploying to npmjs, `NPM_TOKEN`. 

There are 2 ways of using this action in a workflow:

1. From a docker container without an updated marketplace tag, we use this technique to test if the action is fully working for GitHub marketplace users, before deploying and updating our major action tag. Running it this way the preferred outputs are stored to environment variables.

2. From the GitHub marketplace, ensuring stability and having workflow step outputs that can be cross-referenced.

There are multiple use cases for this action/workflow, we will go through them all in the next sections.

### Any kind of npm package

The simplest use case for a typical NPM package, almost zero setup time on GitHub actions and no additional installed tools. Assuming there are no build steps, setting up node/npm is not required.

Release to npm from [ghcr container](https://github.com/open-sauced/semantic-release-conventional-config/pkgs/container/semantic-release-conventional-config):

```yaml
name: "Release"

on:
  push:
    branches:
      - main
      - alpha
      - beta
      - next
      - next-major

jobs:
  release:
    environment:
      name: production
      url: https://github.com/${{ github.repository }}/releases/tag/${{ env.RELEASE_TAG }}
    runs-on: ubuntu-latest
    steps:
      - name: "☁️ checkout repository"
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: "🚀 release"
        id: semantic-release
        uses: docker://ghcr.io/open-sauced/semantic-release-conventional-config:3.0.0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

      - name: '♻️ cleanup'
        run: |
          echo ${{ env.RELEASE_TAG }}
          echo ${{ env.RELEASE_VERSION }}
```

Release to npm from [marketplace action](https://github.com/marketplace/actions/open-sauced-semantic-release-conventional-config):

```yaml
name: "Release"

on:
  push:
    branches:
      - main
      - alpha
      - beta
      - next
      - next-major

jobs:
  release:
    environment:
      name: production
      url: https://github.com/${{ github.repository }}/releases/tag/${{ steps.semantic-release.outputs.release-tag }}
    name: Semantic release
    runs-on: ubuntu-latest
    steps:
      - name: "☁️ checkout repository"
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: "🚀 release"
        id: semantic-release
        uses: open-sauced/semantic-release-conventional-config@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}

      - name: '♻️ cleanup'
        run: |
          echo ${{ steps.semantic-release.outputs.release-tag }}
          echo ${{ steps.semantic-release.outputs.release-version }}
```

### Containerised nodejs application

This is a typical example for NodeJS backend applications or React frontends. Assuming there are no build steps or the package is set as private, setting up node/npm is not required and the docker build job will take care of all the limitations.

```yaml
name: "Release"

on:
  push:
    branches:
      - main
      - alpha
      - beta
      - next
      - next-major

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

  release:
    environment:
      name: production
      url: https://github.com/${{ github.repository }}/releases/tag/${{ steps.semantic-release.outputs.release-tag }}
    name: Semantic release
    needs:
      - docker
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

      - name: "🚀 release"
        id: semantic-release
        uses: open-sauced/semantic-release-conventional-config@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: "♻️ cleanup"
        run: |
          echo ${{ steps.semantic-release.outputs.release-tag }}
          echo ${{ steps.semantic-release.outputs.release-version }}

```

### Containerised nodejs GitHub action

This is the most niche usage, it requires building and storing the build artifact, releasing the docker container, and then updating the `action.yml` as part of the release process. This requires manually editing the release to push to the marketplace and updating the major action tag.

```yaml
name: "Release"

on:
  push:
    branches:
      - main
      - alpha
      - beta
      - next
      - next-major

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

  release:
    environment:
      name: production
      url: https://github.com/${{ github.repository }}/releases/tag/${{ steps.semantic-release.outputs.release-tag }}
    name: Semantic release
    needs:
      - docker
    runs-on: ubuntu-latest
    steps:
      - name: "☁️ checkout repository"
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: "🔧 setup node"
        uses: actions/setup-node@v2.1.5
        with:
          node-version: 16

      - name: "🔧 install npm@latest"
        run: npm i -g npm@latest

      - name: "📦 install dependencies"
        uses: bahmutov/npm-install@v1

      - name: "📂 download docker artifacts"
        uses: actions/download-artifact@v2
        with:
          name: docker
          path: /tmp

      - name: "📦 load tag"
        run: |
          docker load --input /tmp/docker.tar
          docker image ls -a

      - name: "🚀 release"
        id: semantic-release
        uses: open-sauced/semantic-release-conventional-config@v3.0.1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  cleanup:
    name: Cleanup actions
    needs:
      - release
    runs-on: ubuntu-latest
    steps:
      - name: "♻️ remove build artifacts"
        uses: geekyeggo/delete-artifact@v1
        with:
          name: |
            docker
```

We thought about enabling some flexibility for users wanting minimal visual changes without them having to fork the repository and release another semantic configuration. 

It's possible to release the container to another private GitHub repository or the docker registry by manipulating these variables:
- `DOCKER_USERNAME=$GITHUB_REPOSITORY_OWNER`
- `DOCKER_PASSWORD=$GITHUB_TOKEN`

It's possible to change the release commit name and author by manipulating these variables:
- `GIT_COMMITTER_NAME="open-sauced[bot]"`
- `GIT_COMMITTER_EMAIL="63161813+open-sauced[bot]@users.noreply.github.com"`
- `GIT_AUTHOR_NAME=$GITHUB_SHA.authorName`
- `GIT_AUTHOR_EMAIL=$GITHUB_SHA.authorEmail`

Here are all the [semantic-release](https://github.com/semantic-release/semantic-release) plugins and steps explained:
- [`@semantic-release/commit-analyzer`](https://github.com/semantic-release/commit-analyzer) - analyzes conventional commits deciding if they are bumping a patch, minor or major release tag
- [`@semantic-release/release-notes-generator`](https://github.com/semantic-release/release-notes-generator) - generates release notes based on the analysed commits 
- [`@semantic-release/changelog`](https://github.com/semantic-release/changelog) - generates a fancy changelog using the repository name the workflow was run for as a title and cool emojis [example]
- [`conventional-changelog-conventionalcommits`](https://github.com/conventional-changelog/conventional-changelog) - the conventional commit specification configuration preset
- [`@semantic-release/npm`](https://github.com/semantic-release/npm)
- [`@google/semantic-release-replace-plugin`](https://github.com/google/semantic-release-replace-plugin) - if the repository is deploying a containerised action, this updates `action.yml` with the recently released version tag
- [`semantic-release-license`](https://github.com/cbhq/semantic-release-license) - if the repository has a `LICENSE*` file, this updates the year
- [`@semantic-release/git`](https://github.com/semantic-release/git) - this creates the GitHub release commit [example]
- [`@semantic-release/github`](https://github.com/semantic-release/github) - generates GitHub release notes with added release channel links at the bottom [example]
- [`@eclass/semantic-release-docker`](https://github.com/eclass/semantic-release-docker) - if the repository has a `Dockerfile`, this takes care of releasing the container to ghcr.io [example]
- [`@semantic-release/exec`](https://github.com/semantic-release/exec) - used to set GitHub action environment variables when run as from docker container and GitHub action outputs when run as a marketplace action
- [`execa`](https://github.com/sindresorhus/execa) - used to check the commit author and check for various settings in the repository using this action
- [`npmlog`](https://github.com/npm/npmlog) - used to log the setup process

## Submission Category: DIY Deployments

[Note]: # (Maintainer Must-Haves, DIY Deployments, Interesting IoT, Phone Friendly, or Wacky Wildcards)


## Yaml File or Link to Code

[Note]: # (Our markdown editor supports pretty embeds. Try this syntax: `{% github link_to_your_repo %}` to share a GitHub repository)

Live repository using this action in a workflow:

{% github 0-vortex/semantic-release-docker-test %}

GitHub action: 
[@open-sauced/semantic-release-conventional-config/action.yml](https://github.com/open-sauced/semantic-release-conventional-config/blob/main/action.yml)

GitHub container registry Dockerfile:
[@open-sauced/semantic-release-conventional-config/Dockerfile](https://github.com/open-sauced/semantic-release-conventional-config/blob/main/Dockerfile)

Full semantic release configuration:
[@open-sauced/semantic-release-conventional-config/release.config.js](https://github.com/open-sauced/semantic-release-conventional-config/blob/main/release.config.js)

## Additional Resources / Info

Here are all the open source actions we are using to power this release workflow in our repositories and examples:

- [actions/checkout@v2](https://github.com/actions/checkout) - most performant git checkout
- [actions/setup-node@v2.1.5](https://github.com/actions/setup-node) - we use it to set the `node` version to 16
- [actions/upload-artifact@v2](https://github.com/actions/upload-artifact) - we use it to transport our artifacts in between jobs
- [actions/download-artifact@v2](https://github.com/actions/download-artifact) - we use it to download our artifacts in between jobs
- [docker/setup-buildx-action@v1](https://github.com/docker/setup-buildx-action) - we use it to setup the docker builder
- [actions/cache@v2](https://github.com/actions/cache) - we use it to cache docker layers
- [docker/metadata-action@v3](https://github.com/docker/metadata-action) - we use it to normalise most of our docker container values
- [docker/build-push-action@v2](https://github.com/docker/build-push-action) - we use this to build the container
- [bahmutov/npm-install@v1](https://github.com/bahmutov/npm-install) - lightning fast `npm ci` with built-in cache
- [open-sauced/semantic-release-conventional-config@v3](https://github.com/open-sauced/semantic-release-conventional-config) - semantic-release configuration, docker container and GitHub action
- [geekyeggo/delete-artifact@v1](https://github.com/GeekyEggo/delete-artifact) - deletes produced artifacts

Be sure to include the DEV usernames of your collaborators:
{% user mtfoley %}

