---
canonicalUrl: https://dev.to/opensauced/how-to-lint-prs-and-welcome-contributors-using-github-actions-4elo
tags:
- actionshackathon21
- GitHub
- opensource
---

# How to lint PRs and welcome contributors using GitHub Actions

![[hexagons-wallpaper-2560x1600.jpg]]

## Motivation

Expanding the [@open-sauced ecosystem](https://github.com/open-sauced), it became tedious to apply all the necessary tooling available in [open-sauced/open-sauced](https://github.com/open-sauced/open-sauced) to newly created repositories and synchronising existing ones with minor updates.

Having [github actions reusable workflows](https://github.blog/2021-11-29-github-actions-reusable-workflows-is-generally-available/), we figured it would make a lot of sense to centralise our workflows and referencing them in other repositories.

The process was very simple and we want to showcase how to apply a similar process for your own organisation.

## My Workflow
[Note]: # (Please make sure the project links to the appropriate GitHub Actions repository, and includes [an open source license](https://choosealicense.com/) and README.)

```yaml
name: "Compliance"

on:
  pull_request_target:
    types:
      - opened
      - edited
      - synchronize

permissions:
  pull-requests: write

jobs:
  compliance:
    uses: open-sauced/open-sauced/.github/workflows/compliance.yml@main
```

The full workflow is available here: [.github/workflows/compliance.yml](https://github.com/open-sauced/docs.opensauced.pizza/blob/main/.github/workflows/compliance.yml)

## Submission Category: Maintainer Must-Haves

[Note]: # (Maintainer Must-Haves, DIY Deployments, Interesting IoT, Phone Friendly, or Wacky Wildcards)


## Yaml File or Link to Code

[Note]: # (Our markdown editor supports pretty embeds. Try this syntax: `{% github link_to_your_repo %}` to share a GitHub repository)

Live repository using this workflow:

{% github open-sauced/conventional-commit %}

Yaml file link: 
[@open-sauced/open-sauced/.github/workflows/compliance.yml](https://github.com/open-sauced/open-sauced/blob/main/.github/workflows/compliance.yml)

## Additional Resources / Info

Here are all the open source actions we are using to power this compliance workflow:

- [actions/first-interaction@v1](https://github.com/actions/first-interaction) - welcomes first time contributors and invites them to [@open-sauced discord](https://discord.gg/U2peSNf23P)
- [amannn/action-semantic-pull-request@v3.4.0](https://github.com/amannn/action-semantic-pull-request) - ensures pull request title matches [conventional commits specification](https://www.conventionalcommits.org/en/v1.0.0/)
- [mtfoley/pr-compliance-action@v0.2.1](https://github.com/mtfoley/pr-compliance-action) - check PR for compliance on title, linked issues, and files changed 

Be sure to include the DEV usernames of your collaborators:
{% user mtfoley %}

[Reminder]: # (Submissions are due on December 8th, 2021 (11:59 PM PT or 2 AM ET/6 AM UTC on December 9th.)