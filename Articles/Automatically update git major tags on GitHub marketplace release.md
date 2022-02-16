---
tags:
- actionshackathon21
- GitHub
- actions
- javascript
---

# Automatically update git major tags on GitHub marketplace release

![[620d438993be1.jpg]]

## Motivation

Having previously dockerized our semantic release configuration in [@open-sauced/semantic-release-conventional-config](https://github.com/open-sauced/semantic-release-conventional-config), a manual step was needed to publish the action in the GitHub marketplace.

This meant that forcing a major tag update with [@semantic-release/exec](https://github.com/semantic-release/exec) as part of the release process was possible, but would result in the major tag (for example v3) linking to a valid repository commit SHA that is not actually released in the marketplace.

Sensibly solving these issues required:
- keeping the semantic release configuration intact
- testing the action container in another repository
- having a maintainer manually edit and save the release 
- letting the `marketplace.yml` workflow update major tags in the curated release context

It may sound tedious but in practice, all a maintainer has to do to update the marketplace tags is edit a published release and press the save button. That's exactly 2 clicks from the repository front page and the simplest way to have [acceptance testing](https://en.wikipedia.org/wiki/Acceptance_testing).

## My Workflow

```yaml
name: "Action major version tag"

on:
  release:
    types:
      - published
      - edited

jobs:
  latest:
    name: Update action tags
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          ref: ${{ github.ref }}

      - name: "🚀 publish action"
        uses: Actions-R-Us/actions-tagger@v2
        with:
          publish_latest_tag: false
          prefer_branch_releases: false
```

The full workflow is available here: [.github/workflows/marketplace.yml](https://github.com/open-sauced/semantic-release-conventional-config/blob/main/.github/workflows/marketplace.yml)

## Submission Category: Maintainer Must-Haves

## Yaml File or Link to Code

Live repository using this workflow:
{% github open-sauced/semantic-release-conventional-config %}

## Additional Resources / Info

Here are all the open source actions we are using to power this release workflow:
- [Actions-R-Us/actions-tagger@v2](https://github.com/Actions-R-Us/actions-tagger) - updates major tag on release

Be sure to include the DEV usernames of your collaborators:
{% user mtfoley %}
