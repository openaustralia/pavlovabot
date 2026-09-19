# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, GitHub Copilot,
and others) when working with code in this repository. `CLAUDE.md` and
`.github/copilot-instructions.md` point here so the guidance lives in one
place.

## Org-wide standards

OAF keeps its shared standards in the `openaustralia/.github` repository. They
are referenced here rather than restated, because copies drift. Read them before
you start. Where this file disagrees with them, they win and this file is what
needs fixing.

- [`AGENTS.md`](https://github.com/openaustralia/.github/blob/main/AGENTS.md).
  Its "Working as an agent in any OAF repository" section covers how OAF
  writes, staging commits instead of making them, opening pull requests as
  drafts, leaving issue creation to a human unless asked, and handling secrets.
- [`.github/CONTRIBUTING.md`](https://github.com/openaustralia/.github/blob/main/.github/CONTRIBUTING.md).
  The authority on branch naming, pull requests, the DCO sign-off and AI
  disclosure. This repository has no local `CONTRIBUTING.md`, so that one
  applies in full.
- The issue and pull request templates there are inherited by this repository,
  so they are not found locally.

## What this repository is

PavlovaBot is a GitHub App plus a reusable workflow,
`.github/workflows/dependabot-tapioca.yml`. On a Dependabot bundler pull
request, it regenerates Sorbet's Tapioca RBI files and pushes them back to the
PR only once `bin/srb` passes. `README.md` explains the design and the one-off
App setup.

There is no build, lint or test step. The repository is one workflow, one App
manifest and documentation.

## Things to know before editing

- **Callers pin `@main`.** Other repositories call the workflow as
  `openaustralia/pavlovabot/.github/workflows/dependabot-tapioca.yml@main`, so
  a merge here changes their CI immediately. Check the caller in
  `openaustralia/planningalerts` before changing the workflow's `secrets` or
  inputs.
- **Dependabot secrets, not Actions secrets.** `PAVLOVABOT_APP_ID` and
  `PAVLOVABOT_PRIVATE_KEY` must live in the Dependabot secrets store. GitHub
  withholds Actions secrets from any run whose actor is `dependabot[bot]`. Do
  not "fix" this by switching the callers to `pull_request_target`, which
  exposes secrets to a job that runs `bundle install`.
- **Never push with `GITHUB_TOKEN`.** A commit pushed with it does not
  re-trigger other `pull_request` workflows, which is why the first attempt at
  this was reverted in
  [planningalerts#2048](https://github.com/openaustralia/planningalerts/pull/2048).
- **On a real Sorbet error the job must push nothing.** A human needs to look
  at a genuine type conflict.
- **Never commit the App's private key** (the `.pem` file) or read it into an
  AI conversation.
- The workflow's Postgres image and action versions mirror
  `openaustralia/planningalerts`'s
  `.github/workflows/rubyonrails.yml`. Nothing checks this, so keep them
  consistent by hand.
